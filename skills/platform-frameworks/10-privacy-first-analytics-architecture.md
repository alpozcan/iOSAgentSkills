---
title: "Privacy-First Analytics Architecture"
description: "Pluggable multi-backend analytics actor with typed event factory methods, strict privacy filter (no PII), nonisolated fire-and-forget tracking, buffer+flush pattern, and OSLog fallback."
---

# Privacy-First Analytics Architecture

## Context
You need analytics to understand user behavior and optimize the trial-to-conversion funnel, but you cannot send any personally identifiable information (PII) — no calendar titles, no event locations, no personal insights. The system must use a pluggable backend architecture to support multiple providers.

## Pattern

### Multi-Backend Analytics Service

```swift
public protocol AnalyticsBackend: Sendable {
    func track(_ event: AnalyticsEvent) async
    func flush() async
}

// Actor pattern — see [[06-actor-based-concurrency-patterns]]
public actor AnalyticsService: AnalyticsServiceProtocol {
    private var backends: [AnalyticsBackend]
    private var eventBuffer: [AnalyticsEvent] = []
    private let bufferSize: Int
    
    // Fire-and-forget from any context (nonisolated)
    public nonisolated func track(_ event: AnalyticsEvent) {
        Task { await _track(event) }
    }
    
    private func _track(_ event: AnalyticsEvent) async {
        eventBuffer.append(event)
        if eventBuffer.count >= bufferSize { await flush() }
    }

    public func flush() async {
        let events = eventBuffer
        eventBuffer.removeAll()
        for backend in backends {
            do {
                for event in events { await backend.track(event) }
                await backend.flush()
            } catch {
                // Re-buffer failed events for retry (up to max buffer size)
                let remaining = maxBufferSize - eventBuffer.count
                if remaining > 0 {
                    eventBuffer.append(contentsOf: events.prefix(remaining))
                }
            }
        }
    }
}
```

### Typed Event System (No Free-Form Strings)

```swift
public struct AnalyticsEvent: Sendable {
    public let name: String
    public let parameters: [String: String]
    public let timestamp: Date
}

// All events are factory methods — prevents typos and enforces parameter shape
public extension AnalyticsEvent {
    static func screenOpen(_ screen: String) -> AnalyticsEvent {
        .init(name: "screen_open", parameters: ["screen": screen])
    }
    
    static func buttonTap(_ button: String, screen: String) -> AnalyticsEvent {
        .init(name: "button_tap", parameters: ["button": button, "screen": screen])
    }
    
    // Tracks [[08-on-device-llm-with-apple-foundation-models|on-device LLM]] performance
    static func llmInference(durationMs: Int, tokenCount: Int) -> AnalyticsEvent {
        .init(name: "llm_inference", parameters: [
            "duration_ms": "\(durationMs)",
            "token_count": "\(tokenCount)"
        ])
    }
    
    // Trial/conversion funnel (see [[07-storekit2-intelligence-based-trial]])
    static func trialMaturityReached(days: Int, interactions: Int, patterns: Int) -> AnalyticsEvent { ... }
    static func paywallViewed(source: String) -> AnalyticsEvent { ... }
    static func subscriptionStarted(productID: String, trialDay: Int, source: String) -> AnalyticsEvent { ... }
    static func freeLimitHit(type: String) -> AnalyticsEvent { ... }
}
```

### Privacy Filter (What We NEVER Send)

```swift
// NEVER sent to any backend:
// - Calendar event titles (see [[12-eventkit-coredata-sync-architecture]])
// - Event locations
// - Event notes
// - User names
// - Personal AI insights
// - Calendar source names

// ALWAYS safe to send:
// - screen_open (screen name only)
// - button_tap (button identifier only)
// - llm_inference (duration, token count — no prompt content)
// - subscription_events (product ID, no user info)
// - app_lifecycle (launch, background, active)
// - memory_warning (used MB)
// - error (generic message, no user context)
```

### Graceful Backend Configuration

```swift
// Configuration protocol instead of direct Bundle.main access (DIP)
public protocol AnalyticsConfiguration: Sendable {
    var telemetryDeckAppID: String? { get }
    var maxBufferSize: Int { get }
}

public struct BundleAnalyticsConfiguration: AnalyticsConfiguration, Sendable {
    public var telemetryDeckAppID: String? {
        Bundle.main.object(forInfoDictionaryKey: "TELEMETRYDECK_APP_ID") as? String
    }
    public var maxBufferSize: Int { 100 }
}

public struct AnalyticsServiceBlueprint: Blueprint {
    public typealias Output = AnalyticsServiceProtocol

    public static func assemble(from catalog: Assembling) -> AnalyticsServiceProtocol {
        let config: AnalyticsConfiguration = BundleAnalyticsConfiguration()
        var backends: [AnalyticsBackend] = [OSLogAnalyticsBackend()]

        if let appID = config.telemetryDeckAppID, !appID.isEmpty {
            backends.append(TelemetryDeckBackend(appID: appID))
        }

        return AnalyticsService(backends: backends, maxBufferSize: config.maxBufferSize)
    }
}
```

### OSLog Backend (Always Available, Zero Dependencies)

```swift
public struct OSLogAnalyticsBackend: AnalyticsBackend {
    private let logger = Logger(subsystem: "app.wythnos.ios", category: "analytics")
    
    public func track(_ event: AnalyticsEvent) async {
        // Privacy: event names are safe to log publicly, parameters may contain identifiers
        logger.info("[\(event.name, privacy: .public)] \(event.parameters, privacy: .private)")
    }
    
    public func flush() async { /* os_log flushes automatically */ }
}
```

### Usage at Call Sites

```swift
// ViewModel
analytics.track(.screenOpen("chat"))
analytics.track(.buttonTap("send_message", screen: "chat"))

// After LLM response
analytics.track(.llmInference(durationMs: elapsed, tokenCount: tokens))

// Trial/subscription events
analytics.track(.trialMaturityReached(days: 14, interactions: 52, patterns: 3))
analytics.track(.paywallViewed(source: "chat_limit"))
```

## Edge Cases

- **Buffer overflow:** If events arrive faster than backends can flush, the buffer grows unbounded. Set a `maxBufferSize` and drop oldest events when exceeded (drop policy). Log dropped event counts for monitoring.
- **Backend failure handling:** If a remote backend (e.g., TelemetryDeck) is unreachable, failed events should be re-buffered for retry with exponential backoff. Don't block other backends — flush each independently.
- **App termination during flush:** If the app is killed mid-flush, buffered events are lost. For critical events (subscription, trial maturity), persist to disk before flushing. Use `UIApplication.beginBackgroundTask` for flush on background transition.
- **Privacy filter bypass prevention:** The typed event system prevents accidental PII leakage, but string parameters (e.g., `screen` name) could carry user data if misused. Code review rule: all `AnalyticsEvent` factory methods must use hardcoded string literals for parameter names.
- **Log injection via os.Logger:** Always use structured `os.Logger` parameters with `privacy:` annotations. Never interpolate user-provided strings into log messages at `.public` privacy level — this prevents log forging attacks.

## Why This Matters

- **Privacy by architecture** — the typed event system makes it impossible to accidentally send PII
- **Backend-agnostic** — swap TelemetryDeck for PostHog, Mixpanel, or custom OpenTelemetry without changing call sites
- **Graceful degradation** — if TelemetryDeck API key is missing (CI, local dev), analytics still work via OSLog
- **Buffer + flush** prevents excessive network requests
- **`nonisolated func track()`** means analytics calls never block the caller

## Anti-Patterns

- Don't pass free-form strings to analytics — always use typed factory methods
- Don't log calendar event titles, even to OSLog in production — they're PII
- Don't make analytics calls `await`-able in ViewModels — fire-and-forget is correct
- Don't add analytics as a direct external dependency in Feature modules — inject via protocol
- Don't interpolate user-provided data into `os.Logger` messages at `.public` privacy level — use `.private` or `.auto`
- Don't let the event buffer grow unbounded — enforce a max size with a drop-oldest policy
