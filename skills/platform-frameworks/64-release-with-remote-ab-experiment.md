---
title: "Releasing an App Version Together with a Remote A/B Experiment"
description: "Release an approved build through the App Store Connect API and start a remotely configured experiment (Statsig, Firebase, LaunchDarkly…) in the right order: verify the SDK key is compiled into the archive, match parameter values to the code, start the experiment before release because assignments are sticky, set guardrails, and keep later releases from contaminating the measurement."
tags: [experimentation, ab-testing, feature-flags, release, app-store-connect, phased-release, statsig, sentry]
related_skills: [20-fastlane-app-store-connect-publishing, 32-sentry-telemetrydeck-integration, 65-sentry-crash-analysis-for-releases-and-experiments, 63-app-store-connect-subscriptions-via-api]
category: platform-frameworks
---

# Releasing an App Version Together with a Remote A/B Experiment

## Context

A common shape for risky changes is to ship the new code path in the binary, default OFF, behind a
remotely assigned experiment, and turn it on for a small slice after release. The experiment only means
something if four things hold at the moment users open the new version:

1. The binary actually contains the SDK key. Otherwise every user silently gets the default: control.
2. The remote parameter values match what the code parses.
3. The experiment is live **before** anyone launches the new build, because most SDKs cache or stick the
   first value a device sees.
4. Nothing else in the release, or later releases, changes the metric being measured.

Each of these fails silently: the dashboard fills with "control" and nobody notices for a week.

## Pattern

### 1. Confirm the key is in the archive that was uploaded

Keys usually come from a generated secrets file that isn't in git. So "the code reads the key" proves
nothing about the build. Check the archive, **without printing the key**:

```bash
ARCHIVE=~/Library/Developer/Xcode/Archives/<date>/<App> <time>.xcarchive
/usr/libexec/PlistBuddy -c "Print :ApplicationProperties:CFBundleShortVersionString" \
                        -c "Print :ApplicationProperties:CFBundleVersion" "$ARCHIVE/Info.plist"
APP=$(ls -d "$ARCHIVE"/Products/Applications/*.app)
KEY=$(grep -E '^EXPERIMENT_CLIENT_KEY=' .env | cut -d= -f2-)
if strings -a "$APP/$(basename "$APP" .app)" | grep -qF "$KEY"; then
  echo "binary contains the client key"
else
  echo "KEY NOT FOUND: experiment will run all users on control"
fi
```

Match the archive's version and build number to the build that's actually pending release in App Store
Connect, and read that state through the API (next step).

### 2. Read the release state before acting

```ruby
app = Spaceship::ConnectAPI::App.find(ENV.fetch("BUNDLE_ID"))
app.get_app_store_versions(filter: { platform: "IOS" },
                           includes: "build,appStoreVersionPhasedRelease", limit: 3).each do |v|
  puts [v.version_string, v.app_store_state, v.release_type,
        "build=#{v.build&.version}", "phased=#{v.app_store_version_phased_release&.phased_release_state}"].join(" | ")
end
# e.g.  X.Y.Z | PENDING_DEVELOPER_RELEASE | MANUAL | build=N | phased=INACTIVE
```

`PENDING_DEVELOPER_RELEASE` means it's approved and waiting for you. `phased=INACTIVE` means a phased release
is configured and starts on release.

### 3. Check the remote config against the code

Read the experiment through the platform's console API (read-only) and compare the parameter values
with the code's parser:

```bash
curl -s -H "STATSIG-API-KEY: $CONSOLE_KEY" -H "STATSIG-API-VERSION: 20240601" \
  https://statsigapi.net/console/v1/experiments/<experiment_id> \
| python3 -c 'import json,sys; e=json.load(sys.stdin)["data"]
print(e["status"], e["allocation"], e["idType"])
for g in e["groups"]: print(g["name"], g["size"], g["parameterValues"])'
```

```bash
git grep -n -E '"control-value"|"treatment-value"' <release-ref> -- App/   # raw values the code accepts
```

The code should treat a missing experiment, an SDK error or a malformed value as **control**. The
allocation should be small at first (for example 1–2% total), with 50/50 groups.

### 4. Start the experiment first, then release

```bash
curl -s -X PUT -H "STATSIG-API-KEY: $CONSOLE_KEY" -H "STATSIG-API-VERSION: 20240601" \
  https://statsigapi.net/console/v1/experiments/<experiment_id>/start
# then re-read: status must be "active"
```

Why first: SDKs commonly persist the first assignment a device sees (for example "keep device value" and
sticky bucketing, or a cached config used at next launch). A user who opens the new version in the minutes
between release and experiment start can be pinned to the default for the whole experiment.

Then release the approved version:

```ruby
v = app.get_app_store_versions(filter: { platform: "IOS", versionString: ENV.fetch("VERSION") },
                               includes: "build").first
abort "unexpected: #{v.app_store_state} build #{v.build&.version}" unless
  v.app_store_state == "PENDING_DEVELOPER_RELEASE" && v.build&.version == ENV.fetch("BUILD")
v.create_app_store_version_release_request
# re-read → READY_FOR_SALE, phased release ACTIVE day 1
```

The guard against an unexpected state or build number is what stops a script from releasing the wrong build.

Phased release only throttles **automatic updates**. Anyone who updates manually gets the build on day 1,
so exposure starts immediately.

### 5. Guardrails you can actually read

- Tag crash reports with the arm **before** the code path first runs (for example in the crash SDK's
  initial scope). Crashes that kill the process before a final event is sent can then still be split by
  arm.
- Log exposure at the moment the variant is actually used, not at app launch.
- Put a primary metric and one or two guardrails in the experiment, and write down stop conditions in
  advance: for example, a crash-rate increase in treatment, a lower completion rate, or visual bug
  reports.
- Pulse and results endpoints often need **group IDs**, not names, and compute daily: expect "no data"
  on day 1.

### 6. Do the power arithmetic before promising a verdict

With few users, a small allocation can't detect a change in a rare event:

```
users over the window × allocation per arm × baseline event rate = expected events per arm
```

Hypothetical example: 1,000 active users over the window, 1% per arm (10 users each) and a 0.3% crashed-user
baseline give an expected 0.03 crashes per arm. That is **zero** in practice, whatever the treatment does. Decide on the high-frequency measurements every exposed user produces (memory,
latency, completion), and use crash data only as a stop guardrail. Ramp the allocation (1% → 10% → 50%)
only when the guardrails are clean.

### 7. Protect the measurement in later releases

While the experiment runs:

- **Freeze the code under test.** Mark the files it depends on as off-limits for other work.
- **Don't ship anything that moves the metric in both arms.** For example, a second heavy view of the
  same kind skews a memory metric regardless of arm. Hold such features until the experiment ends.
- Record every release date in the experiment timeline.
- Agree on an **end date**, and let dependent work wait for it.

## Edge Cases

- **The key is missing from the uploaded build.** Don't start the experiment. It would measure nothing.
  Ship a build with the key first.
- **The experiment platform's console key only has read scopes.** Starting needs a write-capable key.
  Confirm before the release window.
- **A hotfix during the experiment.** It resets nothing if assignments are keyed by a stable ID. Note
  the date anyway.

## Why This Matters

The release and the experiment start are one operation with an order. Done in the wrong order, or with
a build that lacks the key, the experiment runs for weeks and reports a confident "no difference" that's
really "nobody was in treatment".

## Anti-Patterns

- ❌ Trusting that "the code reads the key" means the uploaded binary has it.
- ❌ Releasing, then starting the experiment "in a minute".
- ❌ Crash tags added after the risky code has already run.
- ❌ Promising a crash-rate verdict from a 1% arm of a small app.
- ❌ Shipping other features that change the measured metric while the test runs.
- ✅ Verify the archive → read the state → match the parameters → start → release → guardrails → ramp.
