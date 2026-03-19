---
title: "Maestro E2E Testing for iOS"
description: "End-to-end UI testing with Maestro — YAML-based flows, iOS simulator integration, CI pipeline setup, and reusable test patterns for critical user journeys."
---

# Maestro E2E Testing for iOS

## Context

XCUITest is powerful but brittle and slow to write. Maestro (`mobile-dev-inc/maestro`) provides declarative YAML-based E2E tests that are readable, fast to author, and tolerant of minor UI changes. It runs on iOS simulators and real devices, requires no code compilation, and integrates into CI. Use Maestro for **smoke tests and critical user journeys** — keep unit/integration tests in Swift Testing ([[05-swift-testing-and-tdd-patterns]]).

## Pattern

### Installation

```bash
# macOS (also works on Linux/WSL for Android)
curl -fsSL "https://get.maestro.mobile.dev" | bash

# Verify
maestro --version

# Requires: Java 17+, Xcode (for iOS simulator)
```

### Project Structure

```
e2e/
├── flows/
│   ├── onboarding.yaml
│   ├── capture-quote.yaml
│   ├── search.yaml
│   ├── widget-config.yaml
│   └── about-page.yaml
├── helpers/
│   └── login.yaml           # Reusable sub-flows
└── .maestro/
    └── config.yaml           # Global config
```

### Basic Flow Syntax

```yaml
# e2e/flows/capture-quote.yaml
appId: com.yourapp.papercut
---
- launchApp:
    clearState: true

# Skip onboarding
- tapOn: "Skip"

# Tap add button
- tapOn:
    id: "add-quote-button"     # accessibilityIdentifier

# Enter quote manually
- tapOn: "Type manually"
- tapOn:
    id: "quote-text-field"
- inputText: "The only way to do great work is to love what you do."

# Enter book info
- tapOn:
    id: "book-title-field"
- inputText: "Steve Jobs"

# Save
- tapOn: "Save"

# Verify quote appears in feed
- assertVisible: "The only way to do great work"
- assertVisible: "Steve Jobs"
```

### Waiting & Assertions

```yaml
# Smart waiting (Maestro auto-waits, but you can be explicit)
- waitForAnimationToEnd

# Assert element visible
- assertVisible: "Quote saved"

# Assert element NOT visible
- assertNotVisible: "Error"

# Wait with timeout
- extendedWaitUntil:
    visible: "Processing complete"
    timeout: 10000   # ms
```

### Reusable Sub-Flows

```yaml
# e2e/helpers/skip-onboarding.yaml
appId: com.yourapp.papercut
---
- tapOn: "Skip"
- assertVisible: "Papercut"
```

```yaml
# Reference in other flows
- runFlow: ../helpers/skip-onboarding.yaml
```

### Running Tests

```bash
# Single flow
maestro test e2e/flows/capture-quote.yaml

# All flows in directory
maestro test e2e/flows/

# Against specific simulator
maestro test --device "iPhone 16 Pro" e2e/flows/

# Record video of test run
maestro record e2e/flows/capture-quote.yaml

# Interactive — build flows step-by-step
maestro studio
```

### CI Integration (GitHub Actions)

```yaml
# .github/workflows/e2e.yml
name: E2E Tests
on:
  pull_request:
    branches: [main]

jobs:
  maestro:
    runs-on: macos-15
    steps:
      - uses: actions/checkout@v4

      - name: Install Maestro
        run: curl -fsSL "https://get.maestro.mobile.dev" | bash

      - name: Build app
        run: |
          xcodebuild -scheme YourApp \
            -destination 'platform=iOS Simulator,name=iPhone 16 Pro' \
            -derivedDataPath build/ \
            build

      - name: Boot simulator
        run: |
          xcrun simctl boot "iPhone 16 Pro"
          xcrun simctl install booted build/Build/Products/Debug-iphonesimulator/YourApp.app

      - name: Run E2E tests
        run: |
          export PATH="$PATH:$HOME/.maestro/bin"
          maestro test e2e/flows/

      - name: Upload test artifacts
        if: failure()
        uses: actions/upload-artifact@v4
        with:
          name: maestro-artifacts
          path: ~/.maestro/tests/
```

### Makefile Integration

```makefile
# Add to project Makefile
e2e:
	maestro test e2e/flows/

e2e-record:
	maestro record e2e/flows/

e2e-studio:
	maestro studio
```

### Critical Flow Examples

**Onboarding flow:**
```yaml
appId: com.yourapp.papercut
---
- launchApp:
    clearState: true
- assertVisible: "Capture Quotes"
- tapOn: "Next"
- assertVisible: "Organize by Book"
- tapOn: "Next"
- assertVisible: "Daily Rediscovery"
- tapOn: "Get Started"
- assertVisible: "Papercut"    # Home screen
```

**About page flow:**
```yaml
appId: com.yourapp.papercut
---
- launchApp
- runFlow: ../helpers/skip-onboarding.yaml
- tapOn:
    id: "settings-tab"
- tapOn: "About"
- assertVisible: "Version"
- assertVisible: "Privacy Policy"
- assertVisible: "Rate on App Store"
- assertVisible: "Support & Feedback"
```

### Accessibility Identifiers

Maestro can match by text or `accessibilityIdentifier`. Always set identifiers on interactive elements:

```swift
Button("Save") { ... }
    .accessibilityIdentifier("save-quote-button")

TextField("Enter quote", text: $text)
    .accessibilityIdentifier("quote-text-field")
```

## Rules

- **Maestro for E2E journeys, Swift Testing for logic.** Don't test business logic in Maestro.
- **`clearState: true`** on the first `launchApp` in each flow — ensures clean state.
- **Use `accessibilityIdentifier`** for reliable element matching — text matching breaks with localization.
- **Keep flows under 20 steps** — long flows are brittle and slow to debug.
- **One journey per file** — makes it easy to identify which flow failed in CI.
- **Run in CI on every PR** — E2E tests catch integration issues that unit tests miss.

## Anti-Patterns

- ❌ DON'T use Maestro for unit testing — it's for user journeys only
- ❌ DON'T match by text in localized apps — use `accessibilityIdentifier`
- ❌ DON'T add `sleep` commands — Maestro auto-waits. Use `extendedWaitUntil` if needed
- ❌ DON'T skip writing `assertVisible` — a flow without assertions is not a test
- ❌ DON'T run E2E tests on every commit — they're slow. Run on PR only
