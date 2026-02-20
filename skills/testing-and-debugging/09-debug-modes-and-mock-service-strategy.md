---
title: "Debug Modes and Mock Service Strategy for iOS Apps"
description: "Three-tier mock system: UI testing (minimal), rich debug (realistic 30-day data), and developer mode (gesture-activated with SHA256). Launch arguments control which tier is active."
---

# Debug Modes and Mock Service Strategy for iOS Apps

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
struct MockLLMEngine: LLMEngineProtocol {
    func generateResponse(prompt: String) async throws -> String {
        try? await Task.sleep(for: .milliseconds(200))
        return "Mock AI response for UI testing."
    }
    func isAvailable() async -> Bool { true }
}

actor MockCalendarStore: CalendarStoreProtocol {
    private var events: [CalendarEvent] = []
    func fetchEvents(from: Date, to: Date) async throws -> [CalendarEvent] {
        if events.isEmpty {
            return try await MockCalendarService().fetchEvents(from: from, to: to)
        }
        return events.filter { $0.startDate >= from && $0.startDate <= to }
    }
}

actor MockSubscriptionService: SubscriptionServiceProtocol {
    func currentStatus() async -> SubscriptionStatus { .free }
    var isPremium: Bool { false }
}
```

**Tier 2: Rich Debug Mocks (realistic data, Pro access)**

Used when `--pro-debug` is passed OR developer mode is activated:

```swift
actor RichMockCalendarStore: CalendarStoreProtocol {
    init() { self.events = generateRichMockEvents() }
    
    private func generateRichMockEvents() -> [CalendarEvent] {
        // 30 days of realistic work/personal data:
        // - Standups Mon/Wed/Fri at 9:30
        // - Deep Work blocks 10:00-12:00
        // - 1:1s every week
        // - Weekend gym sessions
        // - All-day events (retreats)
    }
}

struct RichMockLLMEngine: LLMEngineProtocol {
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

actor MockProSubscriptionService: SubscriptionServiceProtocol {
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

### Registration Flow in App Entry Point

Mock services are registered via [[02-protocol-driven-dependency-injection|ServiceProvider]] overrides at launch:

```swift
@main
struct MyApp: App {
    init() {
        // 1. Register all production services
        ServiceProvider.shared.registerAll()
        
        // 2. Override with mocks based on mode
        if isUITesting {
            ServiceProvider.shared.register(LLMEngineSpec.self) { _ in MockLLMEngine() }
            ServiceProvider.shared.register(CalendarStoreSpec.self) { _ in MockCalendarStore() }
            ServiceProvider.shared.register(SubscriptionServiceSpec.self) { _ in MockSubscriptionService() }
        }
        
        if isProDebug || isDeveloperMode {
            ServiceProvider.shared.register(LLMEngineSpec.self) { _ in RichMockLLMEngine() }
            ServiceProvider.shared.register(CalendarStoreSpec.self) { _ in RichMockCalendarStore() }
            ServiceProvider.shared.register(SubscriptionServiceSpec.self) { _ in MockProSubscriptionService() }
        }
        
        // 3. Re-register dependent services after overrides
        if isUITesting || isProDebug || isDeveloperMode {
            ServiceProvider.shared.register(CalendarSyncManagerSpec.self) { resolver in
                CalendarSyncManager(
                    calendarService: resolver.resolve(\.calendarService),
                    store: resolver.resolve(\.calendarStore)
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

## Why This Matters

- **UI tests are deterministic** — fixed mock data means no flaky tests from calendar permissions or StoreKit sandbox
- **Developer mode** enables QA testers to see Pro features without Xcode or TestFlight
- **Rich mocks** provide 30 days of realistic patterns — essential for testing AI insights, pattern discovery
- **Clean state on launch** prevents test pollution from previous runs
- **Re-registration of dependent services** ensures the mock graph is consistent (e.g., `CalendarSyncManager` uses the mocked store, not the real one)

## Anti-Patterns

- Don't leave mock code in production paths — mocks are only registered when debug flags are present
- Don't use `#if DEBUG` for mock selection — use launch arguments so the same binary works in all modes
- Don't skip re-registering dependent services after overriding their dependencies
- Don't hardcode the developer code — hash it and store the hash in a `.gitignore`d file
- Don't use mock services for unit tests — inject mocks directly via constructor (see [[05-swift-testing-and-tdd-patterns]] for unit test patterns and [[17-snapshot-testing-with-swift-snapshot-testing]] for snapshot mock factories)
