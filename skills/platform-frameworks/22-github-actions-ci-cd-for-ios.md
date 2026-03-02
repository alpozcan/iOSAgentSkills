---
title: "GitHub Actions CI/CD for iOS"
description: "Complete CI/CD pipeline for Tuist-based iOS apps using GitHub Actions: PR testing, snapshot validation across locale/device matrix, archive builds with fastlane match code signing, App Store Connect submission, Tuist and SPM caching, and macOS runner optimization."
---

# GitHub Actions CI/CD for iOS

## Context

A Tuist-based modular iOS app needs automated CI/CD: test on every PR, validate snapshots, build archives, and submit to App Store Connect. GitHub Actions is the CI platform, with [[18-makefile-for-ios-project-workflows|Makefile targets]] as the interface between CI and the build system. This keeps CI workflow files thin — they call `make test`, `make snapshots`, `make archive`, and `make submit` rather than embedding raw `xcodebuild` invocations.

Three workflows cover the full lifecycle: `test.yml` validates every pull request, `build.yml` archives on every push to `main`, and `release.yml` submits to App Store Connect on version tags. All workflows share caching for Tuist, SPM, and Ruby (Bundler) dependencies.

## Pattern

### Workflow Structure

The three workflows map to three stages of the development lifecycle:

```
PR opened/updated  →  test.yml    →  unit tests + snapshot validation
Push to main       →  build.yml   →  archive + upload artifact
Tag v*             →  release.yml →  archive + submit to App Store Connect
```

### Workflow 1: test.yml — PR Validation

Runs on every pull request. Validates that unit tests pass and snapshot tests produce no diffs.

```yaml
# .github/workflows/test.yml
name: Test

on:
  pull_request:
    branches: [main]

concurrency:
  group: test-${{ github.head_ref }}
  cancel-in-progress: true

jobs:
  lint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: SwiftLint
        uses: norio-nomura/action-swiftlint@3.2.1
        with:
          args: --strict

  unit-tests:
    runs-on: macos-15
    needs: lint
    timeout-minutes: 30
    steps:
      - uses: actions/checkout@v4

      - name: Select Xcode
        run: sudo xcode-select -s /Applications/Xcode_16.2.app/Contents/Developer

      - name: Cache Tuist
        uses: actions/cache@v4
        with:
          path: ~/.tuist
          key: tuist-${{ runner.os }}-xcode16.2-${{ hashFiles('Project.swift', 'Tuist/**') }}
          restore-keys: |
            tuist-${{ runner.os }}-xcode16.2-

      - name: Cache SPM packages
        uses: actions/cache@v4
        with:
          path: ~/Library/Developer/Xcode/DerivedData/**/SourcePackages
          key: spm-${{ runner.os }}-${{ hashFiles('Tuist/Package.resolved') }}
          restore-keys: |
            spm-${{ runner.os }}-

      - name: Install Tuist
        run: |
          curl -Ls https://install.tuist.io | bash
          tuist version

      - name: Setup project
        run: make setup

      - name: Run unit tests
        run: make test

  snapshots:
    runs-on: macos-15
    needs: lint
    timeout-minutes: 45
    strategy:
      fail-fast: false
      matrix:
        locale: [en, ar, de, ja, tr]
        device: ["iPhone SE", "iPhone 16 Pro", "iPhone 16 Pro Max"]
    steps:
      - uses: actions/checkout@v4

      - name: Select Xcode
        run: sudo xcode-select -s /Applications/Xcode_16.2.app/Contents/Developer

      - name: Cache Tuist
        uses: actions/cache@v4
        with:
          path: ~/.tuist
          key: tuist-${{ runner.os }}-xcode16.2-${{ hashFiles('Project.swift', 'Tuist/**') }}
          restore-keys: |
            tuist-${{ runner.os }}-xcode16.2-

      - name: Cache SPM packages
        uses: actions/cache@v4
        with:
          path: ~/Library/Developer/Xcode/DerivedData/**/SourcePackages
          key: spm-${{ runner.os }}-${{ hashFiles('Tuist/Package.resolved') }}
          restore-keys: |
            spm-${{ runner.os }}-

      - name: Install Tuist
        run: |
          curl -Ls https://install.tuist.io | bash
          tuist version

      - name: Setup project
        run: make setup

      - name: Verify simulator availability
        run: |
          xcrun simctl list devices available | grep -q "${{ matrix.device }}" || \
            xcrun simctl create "CI-${{ matrix.device }}" \
              "com.apple.CoreSimulator.SimDeviceType.$(echo '${{ matrix.device }}' | tr ' ' '-')"

      - name: Run snapshot tests
        run: make snapshots LOCALE=${{ matrix.locale }} DEVICE="${{ matrix.device }}"

      - name: Upload snapshot diffs on failure
        if: failure()
        uses: actions/upload-artifact@v4
        with:
          name: snapshot-diffs-${{ matrix.locale }}-${{ matrix.device }}
          path: |
            **/Snapshots/__Failures__/
          retention-days: 7
```

### Workflow 2: build.yml — Archive on Main

Runs on every push to `main`. Archives the app with proper code signing via fastlane match and uploads the `.ipa` as a build artifact.

```yaml
# .github/workflows/build.yml
name: Build

on:
  push:
    branches: [main]

concurrency:
  group: build-main
  cancel-in-progress: true

jobs:
  archive:
    runs-on: macos-15
    timeout-minutes: 45
    steps:
      - uses: actions/checkout@v4

      - name: Select Xcode
        run: sudo xcode-select -s /Applications/Xcode_16.2.app/Contents/Developer

      - name: Cache Tuist
        uses: actions/cache@v4
        with:
          path: ~/.tuist
          key: tuist-${{ runner.os }}-xcode16.2-${{ hashFiles('Project.swift', 'Tuist/**') }}
          restore-keys: |
            tuist-${{ runner.os }}-xcode16.2-

      - name: Cache SPM packages
        uses: actions/cache@v4
        with:
          path: ~/Library/Developer/Xcode/DerivedData/**/SourcePackages
          key: spm-${{ runner.os }}-${{ hashFiles('Tuist/Package.resolved') }}
          restore-keys: |
            spm-${{ runner.os }}-

      - name: Cache Ruby gems
        uses: actions/cache@v4
        with:
          path: vendor/bundle
          key: gems-${{ runner.os }}-${{ hashFiles('Gemfile.lock') }}
          restore-keys: |
            gems-${{ runner.os }}-

      - name: Install Tuist
        run: |
          curl -Ls https://install.tuist.io | bash
          tuist version

      - name: Install Ruby dependencies
        run: |
          bundle config path vendor/bundle
          bundle install --jobs 4 --retry 3

      - name: Setup project
        run: make setup

      - name: Setup code signing
        env:
          MATCH_PASSWORD: ${{ secrets.MATCH_PASSWORD }}
          MATCH_GIT_URL: ${{ secrets.MATCH_GIT_URL }}
          APP_STORE_CONNECT_API_KEY_ID: ${{ secrets.APP_STORE_CONNECT_API_KEY_ID }}
          APP_STORE_CONNECT_API_KEY_ISSUER_ID: ${{ secrets.APP_STORE_CONNECT_API_KEY_ISSUER_ID }}
          APP_STORE_CONNECT_API_KEY_CONTENT: ${{ secrets.APP_STORE_CONNECT_API_KEY_CONTENT }}
        run: bundle exec fastlane match appstore --readonly

      - name: Archive
        run: make archive

      - name: Upload build artifact
        uses: actions/upload-artifact@v4
        with:
          name: MyApp-${{ github.sha }}.ipa
          path: build/*.ipa
          retention-days: 30
```

### Workflow 3: release.yml — App Store Submission

Runs when a version tag is pushed (e.g., `v1.2.0`). Builds, signs, and submits to App Store Connect via [[20-fastlane-app-store-connect-publishing|fastlane]].

```yaml
# .github/workflows/release.yml
name: Release

on:
  push:
    tags:
      - 'v*'

jobs:
  submit:
    runs-on: macos-15
    timeout-minutes: 60
    steps:
      - uses: actions/checkout@v4

      - name: Select Xcode
        run: sudo xcode-select -s /Applications/Xcode_16.2.app/Contents/Developer

      - name: Cache Tuist
        uses: actions/cache@v4
        with:
          path: ~/.tuist
          key: tuist-${{ runner.os }}-xcode16.2-${{ hashFiles('Project.swift', 'Tuist/**') }}
          restore-keys: |
            tuist-${{ runner.os }}-xcode16.2-

      - name: Cache SPM packages
        uses: actions/cache@v4
        with:
          path: ~/Library/Developer/Xcode/DerivedData/**/SourcePackages
          key: spm-${{ runner.os }}-${{ hashFiles('Tuist/Package.resolved') }}
          restore-keys: |
            spm-${{ runner.os }}-

      - name: Cache Ruby gems
        uses: actions/cache@v4
        with:
          path: vendor/bundle
          key: gems-${{ runner.os }}-${{ hashFiles('Gemfile.lock') }}
          restore-keys: |
            gems-${{ runner.os }}-

      - name: Install Tuist
        run: |
          curl -Ls https://install.tuist.io | bash
          tuist version

      - name: Install Ruby dependencies
        run: |
          bundle config path vendor/bundle
          bundle install --jobs 4 --retry 3

      - name: Setup project
        run: make setup

      - name: Setup code signing
        env:
          MATCH_PASSWORD: ${{ secrets.MATCH_PASSWORD }}
          MATCH_GIT_URL: ${{ secrets.MATCH_GIT_URL }}
          APP_STORE_CONNECT_API_KEY_ID: ${{ secrets.APP_STORE_CONNECT_API_KEY_ID }}
          APP_STORE_CONNECT_API_KEY_ISSUER_ID: ${{ secrets.APP_STORE_CONNECT_API_KEY_ISSUER_ID }}
          APP_STORE_CONNECT_API_KEY_CONTENT: ${{ secrets.APP_STORE_CONNECT_API_KEY_CONTENT }}
        run: bundle exec fastlane match appstore --readonly

      - name: Archive and submit
        env:
          APP_STORE_CONNECT_API_KEY_ID: ${{ secrets.APP_STORE_CONNECT_API_KEY_ID }}
          APP_STORE_CONNECT_API_KEY_ISSUER_ID: ${{ secrets.APP_STORE_CONNECT_API_KEY_ISSUER_ID }}
          APP_STORE_CONNECT_API_KEY_CONTENT: ${{ secrets.APP_STORE_CONNECT_API_KEY_CONTENT }}
        run: |
          make archive
          make submit

      - name: Create GitHub Release
        uses: softprops/action-gh-release@v2
        with:
          generate_release_notes: true
          files: build/*.ipa
```

### Tuist Caching

Tuist stores generated projects and cached artifacts in `~/.tuist/` and the local `Derived/` directory. Cache these to avoid re-generation when `Project.swift` and the `Tuist/` configuration directory haven't changed:

```yaml
- name: Cache Tuist
  uses: actions/cache@v4
  with:
    path: |
      ~/.tuist
      Derived
    key: tuist-${{ runner.os }}-xcode16.2-${{ hashFiles('Project.swift', 'Tuist/**') }}
    restore-keys: |
      tuist-${{ runner.os }}-xcode16.2-
```

The cache key includes:
- **`runner.os`** — macOS caches are not compatible with Linux
- **`xcode16.2`** — different Xcode versions produce different generated projects; include the version to prevent stale caches
- **`hashFiles('Project.swift', 'Tuist/**')`** — any change to project configuration or Tuist helpers invalidates the cache

### Dependency Caching

SPM packages are resolved into `SourcePackages` inside DerivedData. Cache them based on the lockfile hash:

```yaml
- name: Cache SPM packages
  uses: actions/cache@v4
  with:
    path: ~/Library/Developer/Xcode/DerivedData/**/SourcePackages
    key: spm-${{ runner.os }}-${{ hashFiles('Tuist/Package.resolved') }}
    restore-keys: |
      spm-${{ runner.os }}-
```

For projects using Bundler (fastlane), also cache gems:

```yaml
- name: Cache Ruby gems
  uses: actions/cache@v4
  with:
    path: vendor/bundle
    key: gems-${{ runner.os }}-${{ hashFiles('Gemfile.lock') }}
    restore-keys: |
      gems-${{ runner.os }}-
```

### Code Signing with fastlane match

[[20-fastlane-app-store-connect-publishing|fastlane match]] manages certificates and provisioning profiles in a private git repository, encrypted with a passphrase. In CI, match clones the repo, decrypts the certificates, and installs them into a temporary keychain.

Required GitHub Secrets:

| Secret | Purpose |
|--------|---------|
| `MATCH_PASSWORD` | Passphrase to decrypt the match certificates repo |
| `MATCH_GIT_URL` | URL of the private git repo containing certificates |
| `APP_STORE_CONNECT_API_KEY_ID` | App Store Connect API key ID |
| `APP_STORE_CONNECT_API_KEY_ISSUER_ID` | API key issuer ID |
| `APP_STORE_CONNECT_API_KEY_CONTENT` | Base64-encoded `.p8` key content |

The signing step in CI:

```yaml
- name: Setup code signing
  env:
    MATCH_PASSWORD: ${{ secrets.MATCH_PASSWORD }}
    MATCH_GIT_URL: ${{ secrets.MATCH_GIT_URL }}
    APP_STORE_CONNECT_API_KEY_ID: ${{ secrets.APP_STORE_CONNECT_API_KEY_ID }}
    APP_STORE_CONNECT_API_KEY_ISSUER_ID: ${{ secrets.APP_STORE_CONNECT_API_KEY_ISSUER_ID }}
    APP_STORE_CONNECT_API_KEY_CONTENT: ${{ secrets.APP_STORE_CONNECT_API_KEY_CONTENT }}
  run: bundle exec fastlane match appstore --readonly
```

The `--readonly` flag is critical in CI — it prevents match from creating new certificates or profiles, which would conflict with other CI runs. Certificates should only be created from a developer's machine.

The Fastfile lane for match:

```ruby
lane :certificates do
  app_store_connect_api_key(
    key_id: ENV["APP_STORE_CONNECT_API_KEY_ID"],
    issuer_id: ENV["APP_STORE_CONNECT_API_KEY_ISSUER_ID"],
    key_content: ENV["APP_STORE_CONNECT_API_KEY_CONTENT"],
    is_key_content_base64: true
  )

  match(
    type: "appstore",
    readonly: is_ci,
    git_url: ENV["MATCH_GIT_URL"]
  )
end
```

### Parallel Snapshot Testing

Snapshot tests run across a matrix of locales and devices, validating visual correctness for every combination. See [[17-snapshot-testing-with-swift-snapshot-testing]] for the snapshot testing library setup and [[05-swift-testing-and-tdd-patterns]] for the underlying test patterns.

```yaml
strategy:
  fail-fast: false
  matrix:
    locale: [en, ar, de, ja, tr]
    device: ["iPhone SE", "iPhone 16 Pro", "iPhone 16 Pro Max"]
```

This produces 15 parallel jobs (5 locales x 3 devices). Each job runs:

```yaml
- name: Run snapshot tests
  run: make snapshots LOCALE=${{ matrix.locale }} DEVICE="${{ matrix.device }}"
```

The corresponding Makefile target passes locale and device to `xcodebuild`:

```makefile
LOCALE ?= en
DEVICE ?= iPhone 16 Pro

.PHONY: snapshots
snapshots: _ensure-workspace
	@for scheme in DesignSystem ChatUI InsightsUI OnboardingUI SettingsUI; do \
		echo "  -> $${scheme}SnapshotTests [$(LOCALE) / $(DEVICE)]"; \
		xcodebuild test \
			-workspace $(WORKSPACE) \
			-scheme $$scheme \
			-destination 'platform=iOS Simulator,name=$(DEVICE)' \
			-only-testing:$${scheme}SnapshotTests \
			-testLanguage $(LOCALE) \
			2>&1 | grep -E "Test run" | tail -1; \
	done
```

Setting `fail-fast: false` ensures all 15 jobs complete even if one fails — you want to see all broken snapshots, not just the first one.

### macOS Runner Selection

Pin the runner to `macos-15` (Sequoia) for a consistent Xcode version. GitHub-hosted macOS runners come with multiple Xcode versions pre-installed:

```yaml
jobs:
  test:
    runs-on: macos-15
    steps:
      - name: Select Xcode
        run: sudo xcode-select -s /Applications/Xcode_16.2.app/Contents/Developer

      - name: Verify Xcode version
        run: xcodebuild -version
```

Always pin the full Xcode path (e.g., `Xcode_16.2.app`) rather than relying on the default. GitHub updates the default Xcode version on runners without notice — this breaks builds silently.

### Cost Optimization: Linux Pre-checks

macOS runners cost 10x Linux runners on GitHub Actions. Move non-Xcode checks to Linux:

```yaml
jobs:
  lint:
    runs-on: ubuntu-latest    # $0.008/min vs $0.08/min for macOS
    steps:
      - uses: actions/checkout@v4
      - name: SwiftLint
        uses: norio-nomura/action-swiftlint@3.2.1
        with:
          args: --strict

  unit-tests:
    runs-on: macos-15
    needs: lint              # Only run if lint passes
```

By gating macOS jobs behind Linux pre-checks, a failing lint saves the full cost of a macOS runner spin-up.

## Edge Cases

- **Xcode version mismatch between local and CI** — Pin the exact version in the workflow with `sudo xcode-select -s /Applications/Xcode_16.2.app/Contents/Developer`. Document the required Xcode version in the project README so developers install the same version locally.
- **Simulator not available on runner** — GitHub runners may not have every simulator pre-installed. Verify with `xcrun simctl list devices available` and create the missing device if needed:
  ```yaml
  - name: Verify simulator
    run: |
      xcrun simctl list devices available | grep -q "iPhone SE" || \
        xcrun simctl create "iPhone SE" \
          "com.apple.CoreSimulator.SimDeviceType.iPhone-SE-3rd-generation" \
          "com.apple.CoreSimulator.SimRuntime.iOS-18-2"
  ```
- **Snapshot diffs on CI vs local** — macOS runners may render fonts slightly differently than local machines due to different display scaling and font hinting. Use `perceptualPrecision: 0.98` in snapshot assertions to tolerate sub-pixel differences. See [[17-snapshot-testing-with-swift-snapshot-testing]] for precision configuration.
- **GitHub Actions macOS runner costs (10x Linux)** — Minimize macOS minutes by running SwiftLint, Danger, and other non-Xcode checks on `ubuntu-latest` first. Use `needs: lint` to gate macOS jobs behind passing Linux checks.
- **Secrets not available in forks** — PRs from forked repositories cannot access repository secrets. This means signed builds and snapshot tests that require API keys will fail. Use `if: github.event.pull_request.head.repo.full_name == github.repository` to skip signing steps on fork PRs, and run only unsigned test builds.
- **Cache invalidation** — Include the Xcode version string in every cache key (e.g., `tuist-macOS-xcode16.2-...`). Without this, a runner Xcode update silently corrupts cached DerivedData and Tuist artifacts, causing cryptic build failures.

## Why This Matters

- **Automated testing prevents regressions from reaching main** — Every PR must pass unit tests and snapshot validation before merge. No human can manually verify 15 locale/device combinations on every change.
- **Snapshot validation catches visual bugs across 15 locale x device combinations** — Arabic RTL mirroring, Japanese text truncation, and SE screen overflow are caught automatically rather than in production.
- **Automated submission reduces human error in release process** — Tagging a version triggers the full pipeline: archive, sign, upload. No manual Xcode Organizer clicking, no forgotten build numbers, no certificate confusion.
- **Cost optimization: caching Tuist and SPM saves 5-10 minutes per run** — A full `tuist install` + `tuist generate` takes 3-5 minutes. SPM resolution adds another 2-5 minutes. Caching reduces these to seconds on cache hits, saving both time and macOS runner costs.
- **Consistency between local and CI** — Because CI calls the same [[18-makefile-for-ios-project-workflows|Makefile targets]] that developers run locally, there are no "works on my machine" surprises. `make test` behaves identically everywhere.

## Anti-Patterns

- Don't run `xcodebuild` directly in CI workflow files — use [[18-makefile-for-ios-project-workflows|Makefile targets]] for consistency with local development. When the build command changes, you update the Makefile once, not every workflow file.
- Don't store certificates in the repository — use [[20-fastlane-app-store-connect-publishing|fastlane match]] with an encrypted private repo. Raw certificates in the main repo are a security incident waiting to happen.
- Don't skip simulator pinning — different simulators produce different snapshot pixels. An iPhone SE and iPhone 16 Pro have different screen dimensions, safe areas, and dynamic type defaults. Always pass the device name explicitly.
- Don't run all tests on every PR if the test suite is slow — use path filters to skip unrelated modules:
  ```yaml
  on:
    pull_request:
      paths:
        - 'Features/ChatUI/**'
        - 'Core/**'
  ```
  This avoids burning 45 minutes of macOS runner time when someone edits a README.
- Don't use `self-hosted` runners without security hardening — PR authors can execute arbitrary code in workflow steps. On self-hosted runners, this means arbitrary code on your infrastructure. Use GitHub-hosted runners for public repos, or restrict self-hosted runners to protected branches only.
- Don't rely on the default Xcode version on runners — GitHub updates it without notice. Always pin with `xcode-select -s /Applications/Xcode_X.Y.app`.
- Don't pass secrets as command-line arguments — they appear in process listings and logs. Use environment variables instead (`env:` block in the workflow step).
