---
title: "Debug Modes and Stub Service Strategy for iOS Apps"
description: "Three-tier mock system: UI testing (minimal), rich debug (realistic 30-day data), and developer mode (gesture-activated with SHA256). Launch arguments control which tier is active."
---

# Debug Modes and Stub Service Strategy for iOS Apps

## Context
A real iOS app depends on system frameworks (EventKit, StoreKit, UNUserNotificationCenter, Apple Foundation Models) that are unavailable or unreliable in simulators, CI, and Xcode Previews. You need a layered mock strategy that supports UI testing, developer debugging, and preview rendering — all without polluting production code.

## Pattern

### Launch Argument Flags

```swift
// App entry point checks flags before anything else
let isUITesting = ProcessInfo.processInfo.arguments.contains("--uitesting")
let isProDebug = ProcessInfo.processInfo.arguments.contains("--pro-debug")
let skipOnboarding = ProcessInfo.processInfo.arguments.contains("--skip-onboarding")
let showOnboarding = ProcessInfo.processInfo.arguments.contains("--show-onboarding")
```

### Three-Tier Mock System

**Tier 1: UI Testing Mocks (minimal, fast, predictable)**

Used when `--uitesting` is passed. Mocks return fixed data instantly:

```swift
struct StubLLMEngine: LLMEngineProtocol {
    func generateResponse(prompt: String) async throws -> String {
        try? await Task.sleep(for: .milliseconds(200))
        return "Mock AI response for UI testing."
    }
    func isAvailable() async -> Bool { true }
}

actor StubCalendarStore: CalendarStoreProtocol {
    private var events: [CalendarEvent] = []
    func fetchEvents(from: Date, to: Date) async throws -> [CalendarEvent] {
        if events.isEmpty {
            return try await StubCalendarService().fetchEvents(from: from, to: to)
        }
        return events.filter { $0.startDate >= from && $0.startDate <= to }
    }
}

actor StubSubscriptionService: SubscriptionServiceProtocol {
    func currentStatus() async -> SubscriptionStatus { .free }
    var isPremium: Bool { false }
}
```

**Tier 2: Rich Debug Mocks (realistic data, Pro access)**

Used when `--pro-debug` is passed OR developer mode is activated:

```swift
actor PreviewCalendarStore: CalendarStoreProtocol {
    init() { self.events = generatePreviewEvents() }
    
    private func generatePreviewEvents() -> [CalendarEvent] {
        // 30 days of realistic work/personal data:
        // - Standups Mon/Wed/Fri at 9:30
        // - Deep Work blocks 10:00-12:00
        // - 1:1s every week
        // - Weekend gym sessions
        // - All-day events (retreats)
    }
}

struct PreviewLLMEngine: LLMEngineProtocol {
    private let responses = [
        "Your day looks well-balanced with 3 meetings and 2 hours of focus time...",
        "Pattern detected: You tend to schedule personal activities on weekends...",
        // 5+ varied, realistic responses
    ]
    
    func generateResponse(prompt: String) async throws -> String {
        try? await Task.sleep(for: .milliseconds(300))
        return responses.randomElement()!
    }
}

actor StubProSubscriptionService: SubscriptionServiceProtocol {
    func currentStatus() async -> SubscriptionStatus {
        SubscriptionStatus(tier: .pro, expirationDate: Date() + 365*24*3600, isActive: true)
    }
    var isPremium: Bool { true }
}
```

**Tier 3: Developer Mode (activated via secret gesture)**

A hidden activation flow for testers who don't have Xcode:

```swift
@MainActor
public final class DeveloperModeActivator: ObservableObject {
    static let requiredTapCount = 5
    static let tapWindowSeconds: TimeInterval = 3.0
    
    @Published public private(set) var isCodeEntryEnabled = false
    
    public func registerTap() {
        tapTimestamps.append(Date())
        tapTimestamps = tapTimestamps.filter { Date().timeIntervalSince($0) <= Self.tapWindowSeconds }
        if tapTimestamps.count >= Self.requiredTapCount {
            enableCodeEntry()
        }
    }
    
    private func validateCode(_ code: String) {
        let codeHash = SHA256.hash(data: Data(code.utf8))
        if codeHash == Self.expectedCodeHash {
            activateDeveloperMode()
        }
    }
}
```

### Supply Flow in App Entry Point

Stub services are supplied via [[02-protocol-driven-service-catalog|Catalog]] overrides at launch:

```swift
@main
struct MyApp: App {
    init() {
        // 1. Prepare all production services
        Catalog.main.prepareAll()
        
        // 2. Override with stubs based on mode
        if isUITesting {
            Catalog.main.supply(LLMEngineBlueprint.self) { _ in StubLLMEngine() }
            Catalog.main.supply(CalendarStoreBlueprint.self) { _ in StubCalendarStore() }
            Catalog.main.supply(SubscriptionServiceBlueprint.self) { _ in StubSubscriptionService() }
        }
        
        if isProDebug || isDeveloperMode {
            Catalog.main.supply(LLMEngineBlueprint.self) { _ in PreviewLLMEngine() }
            Catalog.main.supply(CalendarStoreBlueprint.self) { _ in PreviewCalendarStore() }
            Catalog.main.supply(SubscriptionServiceBlueprint.self) { _ in StubProSubscriptionService() }
        }
        
        // 3. Re-supply dependent services after overrides
        if isUITesting || isProDebug || isDeveloperMode {
            Catalog.main.supply(CalendarSyncManagerBlueprint.self) { catalog in
                CalendarSyncManager(
                    calendarService: catalog.obtain(\.calendarService),
                    store: catalog.obtain(\.calendarStore)
                )
            }
        }
    }
}
```

### UI Test Base Class

The base class passes launch arguments to the app, enabling [[18-ui-testing-regression-and-smoke|UI regression tests]] with deterministic state:

```swift
class WythnosUITestCase: XCTestCase {
    var app: XCUIApplication!
    
    // Subclasses override for different modes
    var launchArguments: [String] { ["--uitesting", "--skip-onboarding"] }
    
    override func setUpWithError() throws {
        continueAfterFailure = false
        app = XCUIApplication()
        app.launchArguments = launchArguments
        app.launch()
    }
    
    // Pro debug variant:
    // var launchArguments: [String] { ["--pro-debug", "--skip-onboarding"] }
}
```

### Clean State for Testing

```swift
if isUITesting || isProDebug {
    if let bundleId = Bundle.main.bundleIdentifier {
        UserDefaults.standard.removePersistentDomain(forName: bundleId)
    }
}
```

## Edge Cases

- **Launch arguments in release builds:** `ProcessInfo.processInfo.arguments` is accessible in release builds too. A user could inject `--uitesting` via URL scheme or MDM profile. Gate stub service registration behind `#if DEBUG`:
  ```swift
  #if DEBUG
  if isUITesting { /* supply stubs */ }
  #endif
  ```
- **Developer mode code brute force:** The SHA256-validated developer code has no rate limiting. Add a delay (e.g., 2 seconds) after each failed attempt and lock out after 10 failures to prevent brute force.
- **`removePersistentDomain` clearing user data:** The clean state pattern `UserDefaults.standard.removePersistentDomain(forName: bundleId)` removes ALL user preferences — including onboarding completion, notification preferences, and trial state. This is intentional for UI testing but catastrophic if accidentally triggered in production. Always guard with `#if DEBUG`.
- **Mock data determinism:** `responses.randomElement()!` in `PreviewLLMEngine` makes debug sessions non-reproducible. For bug reports, seed the random generator or use indexed cycling instead.

## Why This Matters

- **UI tests are deterministic** — fixed mock data means no flaky tests from calendar permissions or StoreKit sandbox
- **Developer mode** enables QA testers to see Pro features without Xcode or TestFlight
- **Rich mocks** provide 30 days of realistic patterns — essential for testing AI insights, pattern discovery
- **Clean state on launch** prevents test pollution from previous runs
- **Re-supplying dependent services** ensures the service graph is consistent (e.g., `CalendarSyncManager` uses the stubbed store, not the real one)

## Anti-Patterns

- Don't leave stub code in production paths — stubs are only supplied when debug flags are present
- Don't use `#if DEBUG` for mock selection — use launch arguments so the same binary works in all modes
- Don't skip re-supplying dependent services after overriding their dependencies
- Don't hardcode the developer code — hash it and store the hash in a `.gitignore`d file
- Don't use stub services for unit tests — inject stubs directly via constructor (see [[05-swift-testing-and-tdd-patterns]] for unit test patterns and [[17-snapshot-testing-with-swift-snapshot-testing]] for snapshot stub factories)
