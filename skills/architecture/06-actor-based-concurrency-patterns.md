---
title: "Actor-Based Concurrency Patterns for iOS Services"
description: "When to use actor vs @MainActor vs @unchecked Sendable with NSLock. Covers actor-isolated stores, nonisolated escape hatches, Sendable value types, and app lifecycle integration."
---

# Actor-Based Concurrency Patterns for iOS Services

## Context
Modern iOS apps have multiple services (calendar store, notification service, subscription service, analytics) that must be accessed safely from multiple threads. Swift's structured concurrency with actors provides compile-time safety, but you need deliberate architecture to avoid deadlocks and performance bottlenecks.

## Pattern

### When to Use `actor` vs `@MainActor` vs `@unchecked Sendable`

**Use `actor` for data stores and services with mutable state** (e.g., the [[12-eventkit-coredata-sync-architecture|CalendarStore]]):
```swift
public actor CalendarStore: CalendarStoreProtocol {
    private let container: NSPersistentContainer
    
    public func saveEvents(_ events: [CalendarEvent]) async throws {
        let context = container.newBackgroundContext()
        try await context.perform {
            // CoreData operations on background context
        }
    }
    
    public func fetchEvents(from: Date, to: Date) async throws -> [CalendarEvent] {
        let context = container.newBackgroundContext()
        return try await context.perform {
            // Fetch on background thread, return Sendable values
        }
    }
}
```

**Use `actor` for notification/analytics services with preference state:**
```swift
public actor NotificationService: NotificationServiceProtocol {
    private let preferencesStore: NotificationPreferencesStore
    
    public func scheduleWeeklySummary() async throws {
        let prefs = await preferences  // Actor-isolated access
        guard prefs.weeklySummaryEnabled else { return }
        // ...
    }
}

public actor AnalyticsService: AnalyticsServiceProtocol {
    private var eventBuffer: [AnalyticsEvent] = []
    
    // nonisolated for fire-and-forget from any context
    public nonisolated func track(_ event: AnalyticsEvent) {
        Task { await _track(event) }
    }
    
    private func _track(_ event: AnalyticsEvent) async {
        eventBuffer.append(event)
        if eventBuffer.count >= bufferSize { await flush() }
    }
}
```

**Use `@MainActor` for ViewModels** (created by [[03-ui-composer-pattern-for-feature-modules|UIComposers]]):
```swift
@MainActor
public final class ChatViewModel: ObservableObject {
    @Published public var messages: [ChatMessage] = []
    @Published public var isGenerating: Bool = false
    
    public func sendMessage() async {
        isGenerating = true
        // await crosses isolation boundaries safely
        let events = try await store.fetchEvents(from: start, to: end)
        let response = try await llmEngine.generateResponse(prompt: query)
        messages.append(ChatMessage(content: response, isUser: false))
        isGenerating = false
    }
}
```

**Use `@unchecked Sendable` with `NSLock` for lightweight thread-safe containers (e.g., the [[02-protocol-driven-service-catalog|Catalog]]):**
```swift
public final class Catalog: Assembling, @unchecked Sendable {
    private let lock = NSLock()
    private var builders: [ObjectIdentifier: AnyFactory] = [:]

    public func supply<Spec>(_ specType: Spec.Type, factory: @escaping (Catalog) -> B.Output) {
        lock.lock()
        builders[ObjectIdentifier(specType)] = factory
        lock.unlock()
    }
}

// Also for engines that need internal mutation but no actor isolation:
public final class DynamicPromptEngine: @unchecked Sendable {
    private let lock = NSLock()
    private var _interactionCount: Int = 0
    
    public var interactionCount: Int {
        lock.lock(); defer { lock.unlock() }
        return _interactionCount
    }
}
```

### The `nonisolated` Escape Hatch

For actor methods that don't touch mutable state and need to be called from synchronous contexts:

```swift
public actor SafetyAwareLLMEngine: LLMEngineProtocol {
    // Must be nonisolated because AsyncThrowingStream construction is synchronous
    nonisolated public func generateStream(prompt: String) -> AsyncThrowingStream<String, Error> {
        AsyncThrowingStream { continuation in
            Task {
                // Enter actor isolation inside the Task
                let classification = await self.safetyGuardrail.classify(prompt, locale: .current)
                switch classification {
                case .safe:
                    let stream = self.baseEngine.generateStream(prompt: prompt)
                    for try await chunk in stream { continuation.yield(chunk) }
                    continuation.finish()
                case .offTopic:
                    continuation.yield(self.safetyGuardrail.generateRefusal(for: .offTopic, locale: .current))
                    continuation.finish()
                // ...
                }
            }
        }
    }
    
    // Pure computation — no mutable state access
    nonisolated private func handleHarmfulRequest(type: RefusalType) -> String {
        safetyGuardrail.generateRefusal(for: type, locale: .current)
    }
}
```

### Sendable Data Models

All models that cross actor boundaries must be `Sendable`:

```swift
// Value types are naturally Sendable
public struct CalendarEvent: Sendable, Identifiable, Hashable {
    public let id: String
    public let title: String
    public let startDate: Date
    public let endDate: Date
    public let location: String?
    // All stored properties are Sendable (String, Date, Bool, Optional<Sendable>)
}

// Enums are naturally Sendable
// Enums are naturally Sendable (see also [[14-error-handling-and-typed-error-system]])
public enum SafetyClassification: Sendable, Equatable {
    case safe
    case offTopic
    case harmful(RefusalType)
    case emergency
}
```

### Actor-to-Actor Communication

When one actor needs data from another, use `await` chains:

```swift
public actor CalendarSyncManager {
    private let calendarService: CalendarServiceProtocol
    private let store: CalendarStoreProtocol
    
    public func sync() async -> SyncState {
        do {
            // Cross into CalendarService's isolation
            let events = try await calendarService.fetchEvents(from: startDate, to: endDate)
            // Cross into CalendarStore's isolation
            try await store.saveEvents(events)
            await store.setLastSyncDate(Date())
            return .completed(eventCount: events.count)
        } catch {
            return .failed(error)
        }
    }
}
```

### App Lifecycle Integration with Actors

```swift
// WythnosApp.swift
.onChange(of: scenePhase) { _, newPhase in
    Task {
        switch newPhase {
        case .active:
            await lifecycleTrainer.onAppBecameActive()  // Actor method
            await appBecameActive()
        case .background:
            await lifecycleTrainer.onAppMovedToBackground()
        default: break
        }
    }
}
```

### Decision Tree: Which Isolation Pattern?

```
Is the type a ViewModel?
  → YES: Use @MainActor (required for @Published + SwiftUI)
  → NO: Does it have mutable state?
    → NO: Make it a struct/enum conforming to Sendable
    → YES: Does it need synchronous access (no await)?
      → YES: Use @unchecked Sendable + NSLock (e.g., Catalog)
      → NO: Does it manage a resource with its own queue (CoreData, URLSession)?
        → YES: Use actor with nonisolated escape hatches
        → NO: Use actor (default safe choice)
```

## Edge Cases

- **Actor reentrancy:** When an actor method suspends at an `await` point, another call to the same actor can execute before the first resumes. This can cause unexpected state changes mid-method. Mitigation: capture state into local variables before `await`, then validate state hasn't changed after resuming.
  ```swift
  func withdraw(_ amount: Decimal) async throws {
      let currentBalance = balance  // Capture before await
      let fee = await feeService.calculateFee(for: amount)
      guard balance == currentBalance else { throw BankError.stateChanged }
      guard balance >= amount + fee else { throw BankError.insufficientFunds }
      balance -= (amount + fee)
  }
  ```
- **Actor deadlock:** Actor A awaits Actor B which awaits Actor A. Swift actors use cooperative threading, so this won't deadlock in the traditional sense — but it can cause unexpected interleaving due to reentrancy. Design unidirectional data flow (A → B, never B → A).
- **`nonisolated` property access:** Accessing actor properties from `nonisolated` methods requires the property to be `Sendable` and either `let` (immutable) or computed from immutable state. Marking a `var` property as `nonisolated` is a compiler error.
- **`@unchecked Sendable` abuse:** Classes marked `@unchecked Sendable` without proper locking are time bombs. Every mutable stored property must be guarded by the same lock. Code review rule: grep for `@unchecked Sendable` and verify lock coverage.
- **MainActor starvation:** Long-running `@MainActor` work blocks UI updates. Offload computation to a detached task or background actor, then hop back to MainActor for UI updates:
  ```swift
  @MainActor func loadData() async {
      isLoading = true
      let result = await Task.detached { await self.heavyComputation() }.value
      self.data = result  // Back on MainActor
      isLoading = false
  }
  ```
- **Task cancellation in actors:** Actor methods should check `Task.isCancelled` during long operations and throw `CancellationError` to support structured concurrency cancellation.

## Why This Matters

- **Compile-time data race prevention** — Swift actors make race conditions impossible
- **`nonisolated` + `Task`** pattern lets you return synchronous types (like `AsyncThrowingStream`) while deferring actor work
- **`@unchecked Sendable` with `NSLock`** is the right choice for service catalogs that need synchronous access without `await`
- **All cross-boundary data is `Sendable`** — no accidental reference sharing

## Anti-Patterns

- Don't mark classes as `@unchecked Sendable` unless you've manually verified thread safety with locks
- Don't hold actor locks across `await` points — this causes deadlocks
- Don't make ViewModels actors — they must be `@MainActor` for `@Published` to work with SwiftUI
- Don't use `DispatchQueue.main.async` in an actor-based codebase — use `@MainActor.run` or `Task { @MainActor in }`
- Don't pass mutable reference types across actor boundaries — use value types or `Sendable` types
- Don't ignore actor reentrancy — always validate state after `await` points if the method depends on pre-await state
- Don't create bidirectional actor dependencies — design unidirectional data flow to prevent logical deadlocks
