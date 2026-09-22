---
title: "Design-System Compliance Gate"
description: "Keep every new SwiftUI screen on the design system and the HIG: a zero-cost lint that checks only added lines, a written rules file agents read before coding, a reviewer checklist for what lint can't see, text-style typography tokens so Dynamic Type works everywhere, and WCAG contrast tests for token pairs."
tags: [design-system, lint, dynamic-type, accessibility, hig, contrast, swiftui, agents]
related_skills: [04-design-system-as-core-module, 21-accessibility-voiceover-dynamic-type-patterns, 61-multi-agent-feature-delivery-single-build-lane, 17-snapshot-testing-with-swift-snapshot-testing]
category: ui
---

# Design-System Compliance Gate

## Context

A design system only helps if new code uses it. Under deadline, and especially with AI agents writing UI,
screens drift: `.padding(12)`, `.foregroundStyle(.white)`, `.font(.system(size: 56))`, a bare
`.onTapGesture`, three competing primary buttons. Each is small; together they make an app look
unpolished and fail accessibility review.

A review at the end catches drift too late and costs tokens. This skill puts the rules **before** the code
and adds two cheap gates after it:

| Layer | Catches | Cost |
|---|---|---|
| Rules file in the writer's prompt | Most drift, before it's written | A few k tokens per writer |
| `design_lint.sh` on added lines | Hard-coded values (mechanical) | Zero tokens, no build |
| Design reviewer (read-only) | Hierarchy, rhythm, states, contrast, Dynamic Type layout | One short review per feature |
| Contrast and snapshot tests | Token pairs below WCAG; visual regressions | Part of the normal test run |

In one release, **every** writer's code passed the lint, yet design reviewers still raised 94 findings
across 8 features (hierarchy, missing states, contrast, targets), and 85 were fixed at merge. Lint alone
isn't enough, and neither is review alone.

## Pattern

### 1. Fix typography at the root: tokens must be text styles

A common and silent design-system bug:

```swift
// ❌ Fixed point sizes never scale with Dynamic Type, in every screen that uses the token.
static let body = FontStyle(font: .system(size: 17, weight: .regular))
```

Apple's text styles have the same point sizes at the default content size (Large):

| Text style | Default pt | Text style | Default pt |
|---|---|---|---|
| `.largeTitle` | 34 | `.callout` | 16 |
| `.title` | 28 | `.subheadline` | 15 |
| `.title2` | 22 | `.footnote` | 13 |
| `.title3` | 20 | `.caption` | 12 |
| `.body` / `.headline` | 17 | `.caption2` | 11 |

So when your token sizes match this ladder, the migration **looks pixel-identical at default size** and
makes every `.typography(...)` call scale:

```swift
// ✅ Same look at default size; scales with the user's setting.
static let body          = FontStyle(font: .system(.body, design: .default, weight: .regular))
static let serifTitle1   = FontStyle(font: .system(.title, design: .serif, weight: .bold))
static let bodyEmphasized = FontStyle(font: .system(.body, weight: .semibold))   // headline weight
```

For sizes off the ladder (hero numerals, glyphs), use `@ScaledMetric(relativeTo:)`. Prove the migration
with the existing default-size snapshot tests: they must pass **unchanged**. Accessibility-size
snapshots will legitimately change; get a human to approve new references, never re-record automatically.

### 2. Write the rules down, in the terms of *your* tokens

A `docs/design-rules.md` that every UI-writing agent reads first. Keep it short and concrete:

```markdown
## Never hard-code a design value
| Instead of | Use |
| Color(red:…), .white, .black, .gray | AppColors.* |
| .font(.system(size:)), .font(.title) | .typography(AppTypography.*) |
| .padding(12), spacing: 10 | AppSpacing.* (xs 4, sm 8, md 16, lg 24, xl 32) |
| cornerRadius: 10 | AppRadius.* |
| custom shadows / animations | AppShadow.*, AppAnimation.* (+ reduce-motion variant) |
| home-made buttons/cards/chips | AppButton, AppCard, AppChip, AppEmptyState, AppSkeleton |
Escape hatch: `// design-lint: allow <reason>` on the same line.

## Hierarchy (one of each level a screen uses, in this order)
Screen title → section header → card title → body → supporting → metadata.
One primary action per screen; an upsell is never louder than the screen's own content.

## Rhythm
Side margins 16 · between sections 24 · within a group 8 · label↔value 4 · bottom clearance for floating bars.

## HIG
All text scales · rows stack at accessibility sizes (ViewThatFits / isAccessibilitySize) · 44×44 targets ·
tappables are Buttons · labelled symbols, hidden decorative images · state never by colour alone ·
contrast ≥ 4.5:1 body, ≥ 3:1 large text/icons · leading/trailing, never left/right (RTL) · Reduce Motion.

## States every screen designs
Loading (skeleton shaped like content) · empty · error with retry · locked (preview + upsell; no auto-paywall).
```

Name the surfaces if your app has more than one (for example dark app chrome vs. light detail sheets).
Say which colour tokens belong to each, and which colour **roles** are for status only.

### 3. Lint only the lines you added

Linting the whole codebase on day one blocks all work on legacy debt. Lint the **diff**:

```bash
#!/bin/bash
# scripts/design_lint.sh [base-ref]  — flags hard-coded design values in ADDED Swift lines.
set -uo pipefail
BASE="${1:-main}"
cd "$(git rev-parse --show-toplevel)" || exit 2
# Paths allowed to use raw values: the design system itself, tests, and any frozen areas.
EXCLUDE="${DESIGN_LINT_EXCLUDE:-^(App/DesignSystem/|AppTests/|AppUITests/)}"

RULES=(  # 'regex|message'  (message must not contain |)
  'Color\((red|hue|white):|Color\(#|UIColor\(red:|Color\.(white|black|gray|red|blue|green)([^a-zA-Z]|$)|\.(foregroundColor|foregroundStyle|background|tint|fill)\(\.(white|black|gray)\)|hard-coded colour: use colour tokens'
  '\.font\(\.(system|largeTitle|title|title2|title3|headline|body|callout|subheadline|footnote|caption|caption2)([^a-zA-Z0-9]|$)|UIFont\.|Font\.custom\(|hard-coded font: use typography tokens'
  '\.padding\((\.[a-zA-Z]+, *)?[0-9]+(\.[0-9]+)?\)|hard-coded padding: use spacing tokens'
  '(spacing|Spacing): *[1-9][0-9]*(\.[0-9]+)?([^a-zA-Z]|$)|hard-coded stack spacing: use spacing tokens'
  '(cornerRadius|\.cornerRadius\(|RoundedRectangle\(cornerRadius:) *:? *[0-9]+|hard-coded corner radius: use radius tokens'
  '\.shadow\(color:|\.shadow\(radius:|custom shadow: use shadow tokens'
  '\.animation\(\.(easeIn|easeOut|easeInOut|linear|spring)|withAnimation\(\.(easeIn|easeOut|easeInOut|linear|spring)|raw animation: use animation tokens'
  '\.lineLimit\(1\)|lineLimit(1) on content: badges only, with minimumScaleFactor'
  '\.onTapGesture *\{|onTapGesture: use Button for accessibility'
  'alignment: *\.(left|right)([^a-zA-Z]|$)|\.multilineTextAlignment\(\.(left|right)\)|left/right: use leading/trailing (RTL)'
)

violations=0
while IFS= read -r file; do
  [[ "$file" =~ $EXCLUDE ]] && continue
  added=$(git diff -U0 "$BASE"...HEAD -- "$file" | awk '
    /^@@/ { split($3, a, ","); ln = substr(a[1], 2) + 0; next }
    /^\+\+\+/ { next }
    /^\+/ { print ln ": " substr($0, 2); ln++ }')
  [ -z "$added" ] && continue
  while IFS= read -r line; do
    [[ "$line" == *"design-lint: allow"* ]] && continue
    code="${line#*: }"
    [[ "$code" =~ ^[[:space:]]*// || "$code" =~ \#Preview|PreviewProvider ]] && continue
    for rule in "${RULES[@]}"; do
      if [[ "$code" =~ ${rule%|*} ]]; then
        echo "$file:${line%%:*}: ${rule##*|}"; violations=$((violations + 1))
      fi
    done
  done <<< "$added"
done < <(git diff --name-only --diff-filter=AM "$BASE"...HEAD -- '*.swift')

[ "$violations" -gt 0 ] && { echo "design_lint: $violations violation(s)"; exit 1; }
echo "design_lint: clean"
```

Notes:
- macOS bash regex (ERE) has **no `\b`**. Use `([^a-zA-Z]|$)` instead, or the rule silently never matches.
- Writers run it before committing (`design_lint.sh <their-base-branch>`). The merge step runs it against
  the pre-merge commit, as a hard gate.
- Before release, run it against the last shipped tag. Separately, list whole-file raw values in
  pre-existing screens as **design debt** (in one app: 331 raw values in 57 legacy files). Don't block the
  release on them; schedule them.

### 4. The reviewer checklist (what lint can't see)

A read-only reviewer (a design-system agent or a human) answers these per screen, citing file:line and
the exact token fix:

1. What is the **one primary action**? Is anything louder than it?
2. Is the hierarchy in order, with the right token per level?
3. Is the rhythm right (margins, section gaps, in-group gaps)? Is any card padded twice?
4. Does it stay on one surface's colours?
5. **Contrast**, computed from the token RGB values rather than guessed: which pairs are below 4.5:1?
6. At accessibility text sizes, what breaks: truncation, overlap, horizontal rows that should stack?
7. Are touch targets 44 pt? Are tappables Buttons? Is the VoiceOver grouping and order sensible?
8. Are the loading, empty, error and locked states all designed?

### 5. Contrast tests for token pairs

```swift
import Testing
import SwiftUI

@Suite struct DesignContrastTests {
    static func luminance(_ c: Color) -> Double {
        let ui = UIColor(c); var r: CGFloat = 0, g: CGFloat = 0, b: CGFloat = 0, a: CGFloat = 0
        ui.getRed(&r, green: &g, blue: &b, alpha: &a)
        func lin(_ v: CGFloat) -> Double { let v = Double(v); return v <= 0.03928 ? v / 12.92 : pow((v + 0.055) / 1.055, 2.4) }
        return 0.2126 * lin(r) + 0.7152 * lin(g) + 0.0722 * lin(b)
    }
    static func ratio(_ a: Color, _ b: Color) -> Double {
        let (l1, l2) = (luminance(a), luminance(b)); return (max(l1, l2) + 0.05) / (min(l1, l2) + 0.05)
    }

    @Test(arguments: [
        (AppColors.textPrimary, AppColors.background),
        (AppColors.textSecondary, AppColors.background),
        (AppColors.sheetTextPrimary, AppColors.sheetBackground),
    ])
    func bodyTextMeetsAA(pair: (Color, Color)) {
        #expect(Self.ratio(pair.0, pair.1) >= 4.5)
    }
}
```

Pairs the app already uses below the threshold should be recorded as **known issues**
(`withKnownIssue`), so they're visible without blocking. Brand accent colours used as *text* on light
backgrounds are the usual offenders; keep those colours for fills and icons.

## Edge Cases

- **Glyph sizes next to text.** Use `@ScaledMetric(relativeTo: .body)` so the icon grows with its label.
  Tab bars can cap at Large and offer the Large Content Viewer beyond that.
- **Share cards and widgets render at a fixed canvas.** Tokens still apply; Dynamic Type doesn't.
- **Legacy screens touched by a new feature.** Only the added lines are linted, so the author fixes what
  they touch and nothing more. That keeps diffs reviewable.

## Why This Matters

Accessibility and polish are also App Review and ratings issues. Fixing typography at the token level
turns hundreds of call sites Dynamic-Type-aware in one commit with no visual change at default size.
Linting the diff costs nothing and stops regressions at the source.

## Anti-Patterns

- ❌ Design tokens defined with `.system(size:)`.
- ❌ Linting the whole repo on day one (legacy noise blocks everything), or never linting at all.
- ❌ Re-recording snapshots to make a design-system change pass.
- ❌ Reviewing UI only at the end of a release.
- ✅ Rules in the prompt, lint on the diff, a reviewer for judgement, and tests for contrast and regressions.
