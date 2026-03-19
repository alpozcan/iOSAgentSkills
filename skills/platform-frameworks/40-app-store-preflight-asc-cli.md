---
title: "App Store Preflight with asc CLI"
description: "Pre-submission validation pipeline using the asc CLI (brew install asc) for App Store Connect automation and app-store-preflight-skills for rejection-pattern scanning — metadata pull, rule-based compliance checks, and autofix before review submission."
tags: [asc, app-store-connect, preflight, compliance, metadata, review-guidelines]
related_skills: [20-fastlane-app-store-connect-publishing, 33-app-store-optimization-aso-strategy, 22-github-actions-ci-cd-for-ios, 18-makefile-for-ios-project-workflows]
category: platform-frameworks
---

# App Store Preflight with asc CLI

## Context

You're ready to submit to the App Store but rejections cost days — sometimes weeks. Apple's review guidelines span hundreds of rules across metadata, privacy, subscriptions, entitlements, and design. Manual compliance checks are tedious and error-prone. The `asc` CLI (`brew install asc`) is a single Go binary that wraps every App Store Connect API endpoint (1,200+) with no Ruby or Bundler dependency. Combined with `app-store-preflight-skills` (rule-based rejection-pattern scanning), you get a fully automated preflight pipeline that catches common rejection reasons *before* you hit Submit.

Use this alongside [[20-fastlane-app-store-connect-publishing]] for the publishing pipeline and [[33-app-store-optimization-aso-strategy]] for metadata optimization.

## Pattern

### Installation

```bash
# Install asc CLI
brew install asc

# Verify
asc --version

# Install preflight skills for AI agent integration
npx skills add truongduy2611/app-store-preflight-skills
```

### Authentication

`asc` uses App Store Connect API keys (the same `.p8` key used by fastlane):

```bash
# One-time setup — stores credentials in keychain
asc auth login \
  --name "MyApp" \
  --key-id "XXXXXXXXXX" \
  --issuer-id "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx" \
  --private-key ~/.appstoreconnect/AuthKey_XXXXXXXXXX.p8
```

Keep the `.p8` key out of version control. In CI, pass the key via environment variable or secret mount — never commit it.

### Metadata Pull

Pull your live App Store metadata locally so the preflight scanner can analyze it:

```bash
# Pull all localized metadata for an app
asc metadata pull --app "YOUR_APP_ID"

# List current localizations
asc localizations list --app "YOUR_APP_ID"

# Check current app status
asc status --app "YOUR_APP_ID" --output table
```

This syncs your App Store Connect metadata (name, subtitle, keywords, description, promotional text, screenshots) to a local directory — the same data that [[33-app-store-optimization-aso-strategy]] optimizes.

### Preflight Scan Workflow

The `app-store-preflight-skills` project defines a five-step validation pipeline:

1. **App Type Identification** — loads the appropriate checklist (subscription app, kids app, health/fitness, games, AI app, crypto/finance, VPN, etc.)
2. **Metadata Extraction** — uses `asc metadata pull` to fetch live App Store metadata
3. **Rule-based Scanning** — evaluates your project against organized rejection rules
4. **Issue Reporting** — flags violations with severity levels and file references
5. **Autofix & Validation** — applies corrections and re-runs affected checks

### Rule Categories

The preflight scanner checks five areas that account for the majority of App Store rejections:

#### Metadata Compliance

```
rules/metadata/
├── competitor-brand-mentions.md   # Guideline 2.3.7 — no competitor names
├── apple-trademark-usage.md       # Guideline 5.2.1 — no Apple trademarks
├── inaccurate-descriptions.md     # Guideline 2.3.1 — description matches functionality
└── screenshot-accuracy.md         # Guideline 2.3.4 — screenshots show real UI
```

Common metadata rejections:
- Competitor brand names in keywords or description
- Using "iPhone", "iPad", or Apple trademarks in app name/subtitle
- Description promising features that don't exist in the build
- Screenshots showing UI that differs from the actual app

#### Subscription & IAP Compliance

```
rules/subscriptions/
├── missing-terms-privacy.md       # Links to Terms & Privacy Policy required
├── misleading-pricing.md          # Clear pricing display before purchase
├── restore-purchases.md           # Restore button must be accessible
└── trial-disclosure.md            # Trial terms clearly stated
```

#### Privacy

```
rules/privacy/
├── unnecessary-data-collection.md # Only request data the app needs
├── missing-privacy-manifest.md    # PrivacyInfo.xcprivacy required
├── tracking-transparency.md       # ATT prompt before tracking
└── data-deletion.md               # Account deletion if account creation exists
```

#### Design Guidelines

```
rules/design/
├── sign-in-with-apple.md          # Required if any third-party login exists
├── minimum-functionality.md       # App must do more than wrap a website
└── permissions-purpose-strings.md # All Info.plist usage strings must be accurate
```

#### Entitlements

```
rules/entitlements/
├── unused-capabilities.md         # Don't declare capabilities you don't use
├── background-modes.md            # Each background mode must be justified
└── associated-domains.md          # Domains must resolve and match
```

### Running the Preflight Check

Integrate into your Makefile ([[18-makefile-for-ios-project-workflows]]):

```makefile
# Preflight checks before App Store submission
preflight:
	asc metadata pull --app "$(APP_ID)"
	@echo "--- Metadata pulled. Running preflight scan ---"
	@# The AI agent runs the preflight skill against the pulled metadata
	@# and project source to flag rejection patterns

preflight-status:
	asc status --app "$(APP_ID)" --output table

preflight-builds:
	asc builds list --app "$(APP_ID)" --limit 5 --output table
```

### App-Type Checklists

The preflight scanner loads specialized checklists based on your app type. Ten checklists cover the most common app categories:

| Checklist | Key Checks |
|-----------|------------|
| Universal | Metadata accuracy, privacy manifest, minimum functionality |
| Subscription | Pricing display, restore purchases, trial disclosure, Terms/Privacy links |
| Social / UGC | Content moderation, reporting mechanism, blocking, CSAM scanning |
| Kids (Made for Kids) | No ads without parental gate, no external links, COPPA compliance |
| Health / Fitness | Disclaimer for medical advice, HealthKit data accuracy, data sensitivity |
| Games | Loot box odds disclosure, age rating accuracy, Game Center integration |
| macOS | Sandbox compliance, Hardened Runtime, notarization |
| AI Apps | Content generation disclaimers, safety filters, human review option |
| Crypto / Finance | Regulatory disclaimers, no simulated trading without disclosure |
| VPN | NEVPNManager usage justification, no data harvesting |

### asc CLI Quick Reference

Beyond preflight, `asc` replaces many fastlane commands with a single binary:

```bash
# Builds
asc builds list --app "APP_ID" --limit 10 --output table
asc builds upload --app "APP_ID" --ipa "path/to/App.ipa"

# TestFlight
asc testflight groups list --app "APP_ID"
asc crashes --app "APP_ID" --sort -createdDate --limit 10

# Submission
asc submit create --app "APP_ID" --version "1.2.3" --build "BUILD_ID" --confirm

# One-command publish (upload + submit)
asc publish appstore --app "APP_ID" --ipa "app.ipa" --version "1.2.3" --submit --confirm

# Code signing
asc certificates list
asc profiles list --output table

# Screenshots
asc screenshots list --app "APP_ID"

# Xcode Cloud
asc xcode-cloud run --app "APP_ID" --workflow "CI" --branch "main" --wait
```

### CI Integration

Add preflight as a gate before App Store submission in [[22-github-actions-ci-cd-for-ios]]:

```yaml
# .github/workflows/preflight.yml
name: App Store Preflight
on:
  workflow_dispatch:
  push:
    tags: ['v*']

jobs:
  preflight:
    runs-on: macos-15
    steps:
      - uses: actions/checkout@v4

      - name: Install asc
        run: brew install asc

      - name: Authenticate
        run: |
          asc auth login \
            --name "CI" \
            --key-id "${{ secrets.ASC_KEY_ID }}" \
            --issuer-id "${{ secrets.ASC_ISSUER_ID }}" \
            --private-key <(echo "${{ secrets.ASC_PRIVATE_KEY }}")

      - name: Pull metadata
        run: asc metadata pull --app "${{ secrets.APP_ID }}"

      - name: Check app status
        run: asc status --app "${{ secrets.APP_ID }}" --output table

      - name: List recent builds
        run: asc builds list --app "${{ secrets.APP_ID }}" --limit 5 --output table

      - name: Check for crashes
        run: asc crashes --app "${{ secrets.APP_ID }}" --sort -createdDate --limit 10
```

## Edge Cases

- **Multiple apps:** `asc auth login` supports named profiles. Use `--name` to switch between apps: `asc auth login --name "AppA" ...` and `asc --profile AppA status`.
- **Expired API key:** App Store Connect API keys expire after 20 minutes per session. `asc` handles token refresh automatically, but if you see 401 errors in CI, verify the `.p8` key hasn't been revoked in App Store Connect.
- **Kids category:** Apps in the "Made for Kids" category face the strictest rules. The preflight scanner flags any external links, ad SDKs without parental gates, or data collection that violates COPPA.
- **Privacy manifest requirement:** Starting 2024, Apple requires `PrivacyInfo.xcprivacy` for apps using required reason APIs. The preflight scanner checks your binary for these API calls and verifies the manifest exists and covers them.
- **macOS notarization:** If your app ships on macOS, `asc` can also manage the notarization workflow. This is separate from iOS submission but uses the same API key.
- **JSON output for scripting:** Append `--output json` to any `asc` command for machine-readable output. Pipe into `jq` for custom CI checks.

## Why This Matters

App Store rejections are the most expensive kind of bug — they block your entire release, often for days. The median rejection-to-resolution cycle is 3-5 business days, and each resubmission restarts the review queue. A preflight pipeline that catches metadata violations, missing privacy manifests, subscription compliance gaps, and entitlement misconfigurations *before* submission eliminates the most common rejection categories. The `asc` CLI makes this automatable in CI with zero Ruby dependencies and full API coverage, while the preflight skills provide the rule library that maps Apple's guidelines to concrete, scannable checks.

## Anti-Patterns

- **Skipping preflight on "minor" updates** — even a metadata-only update can be rejected for a stale screenshot or changed privacy practice. Run preflight on every submission.
- **Submitting without pulling fresh metadata** — your local metadata may be stale. Always `asc metadata pull` before scanning to check what's actually live.
- **Ignoring low-severity warnings** — preflight warnings often become rejections in edge cases (specific reviewer, specific region). Fix all warnings, not just errors.
- **Hardcoding the app ID** — use environment variables or Makefile variables so the same preflight pipeline works across multiple apps and CI environments.
- **Using asc for everything, fastlane for nothing** — `asc` excels at API calls and metadata. Fastlane still has stronger screenshot generation (`snapshot` + `frameit`) and match for team code signing. Use both where each is strongest.
- **Running preflight only in CI** — run it locally first. Catching issues before push saves a CI round-trip.

## Related Skills

- [[20-fastlane-app-store-connect-publishing]] — the full publishing pipeline; `asc` complements fastlane, not replaces it
- [[33-app-store-optimization-aso-strategy]] — metadata optimization that preflight validates
- [[22-github-actions-ci-cd-for-ios]] — CI/CD pipeline where preflight runs as a gate
- [[18-makefile-for-ios-project-workflows]] — Makefile targets for local preflight runs
- [[32-sentry-telemetrydeck-integration]] — observability setup that preflight checks for privacy compliance
