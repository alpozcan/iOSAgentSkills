---
title: "UI Composer Pattern for Feature Module Composition"
description: "Feature modules expose a single UIComposer that accepts protocol dependencies via constructor injection and produces concrete SwiftUI Views -- no AnyView, no service locator."
---

# UI Composer Pattern for Feature Module Composition

## Context
In a [[01-tuist-modular-architecture|modular iOS app]], Feature modules need to construct Views with ViewModels that depend on multiple Core services. The Feature module cannot access the catalog directly (that would create a circular dependency), and Views should not know about service resolution.

## Pattern

Each Feature module exposes a **UIFactory** that accepts pre-resolved dependencies via constructor injection, and provides a `compose*View()` method that constructs the full View + ViewModel stack.

### Three-Part Feature Module Structure

**1. The Spec ([[02-protocol-driven-service-catalog|DI integration point]])**
```swift
// Chat/Sources/ChatUIComposer.swift
public struct ChatUIBlueprint: Blueprint {
    public typealias Output =ChatUIComposer
    public init() {}
    
    public static func assemble(from: Assembling) -> ChatUIComposer {
        ChatUIComposer(
            store: catalog.obtain(\.calendarStore),
            llmEngine: catalog.obtain(\.llmEngine),
            subscriptionService: catalog.obtain(\.subscriptionService),
            analytics: catalog.obtain(\.analyticsService)
        )
    }
}

extension Catalog {
    public var chatUI: ChatUIBlueprint.Type { ChatUIBlueprint.self }
}
```

**2. The Factory (dependencies → View)**
```swift
public struct ChatUIComposer {
    private let store: CalendarStoreProtocol
    private let llmEngine: LLMEngineProtocol
    private let subscriptionService: SubscriptionServiceProtocol
    private let analytics: AnalyticsServiceProtocol

    public init(
        store: CalendarStoreProtocol,
        llmEngine: LLMEngineProtocol,
        subscriptionService: SubscriptionServiceProtocol,
        analytics: AnalyticsServiceProtocol
    ) {
        self.store = store
        self.llmEngine = llmEngine
        self.subscriptionService = subscriptionService
        self.analytics = analytics
    }

    @MainActor
    public func composeChatView() -> ChatView {
        ChatView(viewModel: ChatViewModel(
            store: store,
            llmEngine: llmEngine,
            subscriptionService: subscriptionService,
            analytics: analytics
        ))
    }
}
```

**3. Usage at the call site (zero knowledge of dependencies)**
```swift
// MainTabView.swift
case .chat:
    Catalog.main[ \.chatUI).composeChatView()
```

### Factory Variations

**Factory with callback parameter (onboarding):**
```swift
public struct OnboardingUIComposer {
    private let calendarService: CalendarServiceProtocol
    private let userProfileService: UserProfileServiceProtocol
    private let analytics: AnalyticsServiceProtocol

    @MainActor
    public func composeOnboardingView(onComplete: @escaping () -> Void) -> RootOnboardingContainer {
        let viewModel = OnboardingViewModel(
            calendarService: calendarService,
            userProfileService: userProfileService,
            analytics: analytics,
            onComplete: onComplete
        )
        return RootOnboardingContainer(onboardingView: OnboardingView(viewModel: viewModel))
    }
}
```

**Factory with runtime data (insights with user name):**
```swift
public struct InsightsUIComposer {
    private let store: CalendarStoreProtocol
    private let llmEngine: LLMEngineProtocol
    private let userName: String?  // Resolved at spec creation time

    @MainActor
    public func composeInsightsView() -> InsightsView {
        let viewModel = InsightsViewModel(
            store: store,
            llmEngine: llmEngine,
            userName: userName
        )
        return InsightsView(viewModel: viewModel)
    }
}
```

### ViewModel as Constructor-Injected Dependency Hub

```swift
@MainActor
public final class ChatViewModel: ObservableObject {
    @Published public var messages: [ChatMessage] = []
    @Published public var isGenerating: Bool = false
    @Published public var isPremium: Bool = false

    private let store: CalendarStoreProtocol
    private let llmEngine: LLMEngineProtocol
    private let subscriptionService: SubscriptionServiceProtocol
    private let analytics: AnalyticsServiceProtocol

    public init(
        store: CalendarStoreProtocol,
        llmEngine: LLMEngineProtocol,
        subscriptionService: SubscriptionServiceProtocol,
        analytics: AnalyticsServiceProtocol
    ) {
        self.store = store
        self.llmEngine = llmEngine
        self.subscriptionService = subscriptionService
        self.analytics = analytics
    }
}
```

### Split Factory Pattern (SRP for Large Features)

When a feature has many screens (5+), split the factory into sub-factories:

```swift
public struct ChatUIComposer {
    // Sub-factories for distinct flows within Chat
    public let conversationFactory: ConversationSubComposer
    public let historyFactory: HistorySubComposer

    public init(store: CalendarStoreProtocol, llmEngine: LLMEngineProtocol, ...) {
        self.conversationFactory = ConversationSubComposer(store: store, llmEngine: llmEngine)
        self.historyFactory = HistorySubComposer(store: store)
    }
}

// Each sub-factory owns its own screen construction
public struct ConversationSubComposer {
    private let store: CalendarStoreProtocol
    private let llmEngine: LLMEngineProtocol

    @MainActor
    public func composeConversationView() -> ConversationView { ... }

    @MainActor
    public func composeDetailView(for message: ChatMessage) -> MessageDetailView { ... }
}
```

**Rule:** Callbacks (navigation, completion) go in `compose*()` parameters. Dependencies (services, stores) go in the factory constructor. This keeps the factory's constructor stable while allowing per-screen customization.

## Edge Cases

- **Stale dependencies after factory creation:** If a factory is created during `prepareAll()` and a dependency is later replaced (e.g., mock override in tests), the factory still holds the old reference. Mitigation: create factories lazily at call sites, not eagerly at registration.
- **Thread safety:** Factories are created during service registration (any thread) but `makeView()` must run on `@MainActor`. The `@MainActor` annotation on `compose*View()` methods enforces this at compile time.
- **Memory leaks from retained closures:** Callbacks passed to `make*View(onComplete:)` can retain the calling ViewModel, creating retain cycles. Use `[weak self]` in callbacks or make factories value types (structs) so they don't participate in cycles.
- **Factory explosion:** Don't create a factory for every screen — only for feature entry points. Internal navigation within a feature can use standard SwiftUI `NavigationStack` without factories.

## Why This Matters

- **Feature modules are self-contained** — they expose a single `UIComposer` and nothing else
- **No service locator calls inside Views or ViewModels** — all dependencies are constructor-injected
- **[[06-actor-based-concurrency-patterns|`@MainActor`]] on `compose*View()`** ensures Views are created on the main thread
- **Testable** — create a `ChatUIComposer` with mock services directly, no catalog needed
- **Views are concrete types** (not `AnyView`) — better SwiftUI performance and type safety

## Anti-Patterns

- Don't pass `Catalog` into ViewModels — inject resolved protocols instead
- Don't use `@EnvironmentObject` for core service dependencies — it's invisible and crashes at runtime if missing
- Don't create ViewModels inside Views — the Factory creates the ViewModel and passes it to the View
- Don't return `AnyView` from factories unless absolutely necessary (e.g., conditional views) — prefer concrete types
- Don't eagerly create all factories at registration time — create them lazily to avoid stale dependency references
- Don't put navigation logic inside factories — factories create views, navigation coordinators handle flow
