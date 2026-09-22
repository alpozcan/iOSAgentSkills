---
title: "Multi-Agent Feature Delivery with a Single Build Lane"
description: "Ship several iOS features at once with parallel AI agents without freezing the developer's Mac or burning tokens: writers never build, one low-priority serial build lane compiles and merges, a script-generated context pack replaces per-agent exploration, and cross-feature audits run before release."
tags: [agents, orchestration, xcodebuild, worktrees, tokens, release, workflow]
related_skills: [62-design-system-compliance-gate, 05-swift-testing-and-tdd-patterns, 17-snapshot-testing-with-swift-snapshot-testing, 18-makefile-for-ios-project-workflows, 01-tuist-modular-architecture]
category: workflow-and-planning
---

# Multi-Agent Feature Delivery with a Single Build Lane

## Context

Parallel coding agents write Swift quickly. But Xcode builds are the bottleneck, and the naive setup of
"each agent in its own worktree builds and tests its own feature" fails in three ways:

1. **The machine locks up.** A single `xcodebuild` of a mid-sized SwiftUI app with a few SPM dependencies
   fans out into dozens of compiler processes. In one measured run, **one** build drove a 12-core laptop to
   a load average of 38. Ten agents means ten cold builds, plus ten simulator clones if they run tests.
2. **Every worktree builds cold.** A fresh worktree has no DerivedData, so every agent recompiles every
   dependency. Incremental builds of a single checkout are far cheaper than N cold builds.
3. **Tokens go to exploration and retries, not code.** Each agent re-reads the same core files and a large
   shared spec, then pastes long build logs back into its context on every try.

This skill splits **writing** from **building**, and adds the cross-feature checks that per-feature review
misses.

## Pattern

### The shape

```
            ┌─ writer A ─┐ ┌─ design review A ─┐
 context    ├─ writer B ─┤ ├─ design review B ─┤        (parallel, NO builds)
 pack  ───▶ ├─ writer C ─┤ ├─ design review C ─┤
            └─ ...      ─┘ └─ ...             ─┘
                  │ each commits on feat/x-<id> in its own worktree
                  ▼
   build lane (ONE agent at a time, serial, low priority)
   step 0: make base compile → merge A → fix → build → A's tests → merge B → ...
                  ▼
   translation batch ─▶ full suite once ─┬─ cross-feature audits (read-only)
                                         └─ fix pass + Release build + checklist
```

The build lane starts step 0 while writers are still writing, and merges each package as soon as it is
ready, in dependency order. There is no barrier between writing and building.

### Rule 1: writers never build

Writer prompt constraints:

- No `xcodebuild`, `tuist`/`xcodegen`, `swift build`, simulators or test runners.
- Read the real declarations you depend on; open a file rather than guess a signature.
- Write focused tests and **report their suite names**, so the lane runs only those.
- Report the signatures you were unsure of, so the lane checks those first.

Writers produce compile-ready code most of the time. The lane fixes the rest in batches. In one run of
8 feature packages, the lane needed **1–3 builds per package (18 builds in total)** for everything,
including a Release build.

### Rule 2: one build lane, low priority, incremental

```bash
# Half the cores, no indexing, no parallel test clones.
nice -n 15 xcodebuild build \
  -workspace App.xcworkspace -scheme App \
  -destination "platform=iOS Simulator,id=$SIM_UDID" \
  -configuration Debug -jobs 6 \
  COMPILER_INDEX_STORE_ENABLE=NO CODE_SIGNING_ALLOWED=NO -quiet 2>&1 \
  | grep -E "error:|BUILD" | sort -u | head -40

nice -n 15 xcodebuild test ... -parallel-testing-enabled NO \
  -only-testing:AppTests/NewFeatureSuite -only-testing:AppTests/OtherSuite 2>&1 \
  | grep -E "error:|✘|failed|passed|TEST (SUCCEEDED|FAILED)" | tail -30
```

- Pin the simulator **by UDID**, never by `name=` (ambiguous names silently pick the wrong runtime).
- `-parallel-testing-enabled NO` stops Xcode cloning several simulators for one test run.
- **Filter output before it reaches the model.** Errors only, deduplicated, capped.
- Read every error from one build and batch the fixes before rebuilding. Budget about 3 builds per step.
  After about 4, switch the feature off behind its flag and move on, rather than looping.
- Run only the merged feature's suites per step, and the **full suite once** at the end.
- Keep a shared `builder-notes.md` that each lane step appends to (recurring errors, API corrections,
  conflict recipes). Lane steps can then be short-lived agents without relearning the same lessons.

If the machine is still sluggish, stop the run. Anything uncommitted stays in the working tree, and a
workflow engine with resume support replays completed steps from cache.

### Rule 3: a context pack instead of exploration

Before launching agents, generate one small file **with a script, not an agent**:

```bash
{
  echo "## Source tree"; git ls-files App | grep '\.swift$' | sed 's#^App/##' | sort
  echo "## Bundled data"; for f in $(git ls-files 'App/Resources/*.json'); do
    printf -- "- %s: " "$f"; python3 -c "import json;d=json.load(open('$f'));d=d if isinstance(d,list) else [d];print(sorted(d[0])[:12], len(d))"
  done
  for f in App/Core/ServiceProvider.swift App/Services/Protocols/*.swift App/Models/*.swift; do
    echo "## $f"; grep -nE '^\s*(public |final |@MainActor )*(protocol|struct|class|enum|actor|func|init|case|static let|var|let) ' "$f" \
      | grep -v private | cut -c1-150 | head -60
  done
  echo "## Design system"; grep -hnE '^(struct|enum|extension) |static let|init\(' App/DesignSystem/**/*.swift | head -90
  echo "## Test style"; sed -n 1,30p AppTests/SomeRepresentativeTests.swift
} > context.md     # ~30 KB ≈ 8k tokens
```

Also **split big specs into one file per feature**, keeping only what an implementer needs (the reviewed
spec plus its correction notes, without the reviewers' evidence logs). In one run a 486 KB shared spec
became about 40 KB per feature, so every writer saved roughly 100k tokens of input.

### Rule 4: shared contracts, stated up front

Packages written in parallel must meet at named seams. Put exact signatures in every writer prompt and
say which package owns each:

```swift
// [OWNER: NOTIFY] — others code against this, never re-declare it
protocol NotificationScheduling: Sendable {
    func schedule(_ items: [LocalNotification], owner: NotificationOwner) async
    func removeAll(owner: NotificationOwner) async
}
// [OWNER: FLAGS]
enum AppFeature: String, CaseIterable, Sendable { case featureA, featureB }
enum FeatureGate { static func isEnabled(_ f: AppFeature) -> Bool }
```

Rules for shared files (project manifest, DI registry, root view, analytics, `Localizable.strings`): keep
edits **additive** (append, never reorder or reformat), and prefer per-feature extension files
(`Registry+FeatureA.swift`, `Analytics+FeatureA.swift`, `FeatureA.strings`). The lane then resolves
conflicts mechanically:

| Conflict | Recipe |
|---|---|
| `project.pbxproj`, workspace | Take either side, regenerate with Tuist or XcodeGen, commit the result. |
| Strings tables, DI/analytics call sites | Union. |
| Project manifest (plist keys, schemes) | Union of keys. Watch the commas. |
| A hunk that splits a closure | Check brace balance after resolving (a common silent break). |

### Rule 5: build only what ships switched on

Features that ship behind a flag set to OFF still cost writing, review, merge and fix tokens, with zero
launch value. Defer them to the next release. Cut **scope** before optimizing orchestration: in one
planning pass this removed about 40% of the work.

### Rule 6: batch mechanical work on a cheaper model, but size it

Translation and data generation suit a cheaper model. But **give it a bounded, enumerated job**. An
open-ended "translate everything missing" found about 21,000 strings (hundreds of keys × 40+ locales),
judged it too big, and translated almost nothing. Instead:

- A script lists the missing (table, locale) pairs and verifies completeness afterwards.
- Fan out **one agent per table or group of locales**, each with an explicit key list.
- Check placeholder parity (`%1$@`, `%d`, plural categories) with the script, not by eye.
- Translate legal and subscription text first, and flag it for human review.

### Rule 7: per-package review misses cross-feature problems, so audit the integrated branch

Reviews scoped to one package's diff approved code that was wrong **in combination**. On the integrated
branch, two read-only audits then found 5 blockers and about 20 major issues, for example:

- A new content filter was honoured by its own screens but bypassed by three **other** new features
  (a daily puzzle, a quiz result card, a widget).
- A notification campaign was scheduled for everyone under provisional authorization, while its UI
  promised opt-in.
- The paywall "stays free" line contradicted a gate another package had added.
- Privacy copy, the privacy manifest and the live privacy policy disagreed with each other.

So after integration, always run at least:

1. **Cross-cutting behaviour audit**: every new rule (filters, gates, consent) checked against every
   surface, not just the feature that introduced it.
2. **App Review and privacy audit**: paywall disclosures, every premium gate listed as a perk, manifest vs.
   code vs. store copy vs. live policy, permission timing, version strings.
3. **Full test suite once**, reporting failures without auto-fixing snapshots.

Then run one fix pass with the same build discipline. Owner-only items (snapshot re-recording, publishing
a policy page, App Store Connect settings) go to a checklist, never done silently.

## Edge Cases

- **Interrupted writers.** Commit what they left, labelled "(unverified)", and make the lane's step 0
  compile it. That's cheaper than rewriting.
- **Long-lived feature branch behind main.** Merge main into a *new* branch so the old one stays as a
  fallback. Regenerated secrets or config files may need their generator re-run with the right env
  (for example `SRCROOT` unset outside Xcode).
- **Snapshot tests after a design-system change.** Never re-record to get green. Save candidate images
  and ask a human to sign off.
- **Diagnostics noise.** An editor shows "No such module" errors for files in unbuilt worktrees. Ignore
  them; the lane is the source of truth.

## Why This Matters

Measured across one release (8 packages):

| | Naive parallel plan | Single build lane |
|---|---|---|
| Agents | about 60 | 30 |
| Builds | about 40 cold, parallel | 18 incremental, serial, low priority |
| Full-suite runs | about 20 | 2 |
| Machine | unusable | usable throughout |

## Anti-Patterns

- ❌ Letting every agent run `xcodebuild test` in its own worktree.
- ❌ Reviewers that rebuild to "verify". Review the diff and use the lane's build output as evidence.
- ❌ One huge shared spec file read by every agent.
- ❌ Building features that ship switched off.
- ❌ "Translate everything missing" as a single open-ended task.
- ❌ Treating per-package approval as release readiness.
- ✅ Writers write; one lane builds; audits look across features; humans sign off on snapshots and
  anything published.
