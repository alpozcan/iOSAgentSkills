# UI Factory Pattern for Feature Module Composition

## Context
In a modular iOS app, Feature modules need to construct Views with ViewModels that depend on multiple Core services. The Feature module cannot access the DI container directly (that would create a circular dependency), and Views should not know about service resolution.

## Pattern

Each Feature module exposes a **UIFactory** that accepts pre-resolved dependencies via constructor injection, and provides a `make*View()` method that constructs the full View + ViewModel stack.

### Three-Part Feature Module Structure

**1. The Spec (DI integration point)**
```swift
// Chat/Sources/ChatUIFactory.swift
public struct ChatUISpec: DependencySpec {
    public typealias Service = ChatUIFactory
    public init() {}
    
    public static func make(resolver: DependencyResolving) -> ChatUIFactory {
        ChatUIFactory(
            store: resolver.resolve(\.calendarStore),
            llmEngine: resolver.resolve(\.llmEngine),
            subscriptionService: resolver.resolve(\.subscriptionService),
            analytics: resolver.resolve(\.analyticsService)
        )
    }
}

extension ServiceProvider {
    public var chatUI: ChatUISpec.Type { ChatUISpec.self }
}
```

**2. The Factory (dependencies → View)**
```swift
public struct ChatUIFactory {
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
    public func makeChatView() -> ChatView {
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
    ServiceProvider.shared.resolve(\.chatUI).makeChatView()
```

### Factory Variations

**Factory with callback parameter (onboarding):**
```swift
public struct OnboardingUIFactory {
    private let calendarService: CalendarServiceProtocol
    private let userProfileService: UserProfileServiceProtocol
    private let analytics: AnalyticsServiceProtocol

    @MainActor
    public func makeOnboardingView(onComplete: @escaping () -> Void) -> RootOnboardingContainer {
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
public struct InsightsUIFactory {
    private let store: CalendarStoreProtocol
    private let llmEngine: LLMEngineProtocol
    private let userName: String?  // Resolved at spec creation time

    @MainActor
    public func makeInsightsView() -> InsightsView {
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

## Why This Matters

- **Feature modules are self-contained** — they expose a single `UIFactory` and nothing else
- **No service locator calls inside Views or ViewModels** — all dependencies are constructor-injected
- **`@MainActor` on `make*View()`** ensures Views are created on the main thread
- **Testable** — create a `ChatUIFactory` with mock services directly, no DI container needed
- **Views are concrete types** (not `AnyView`) — better SwiftUI performance and type safety

## Anti-Patterns

- Don't pass `ServiceProvider` into ViewModels — inject resolved protocols instead
- Don't use `@EnvironmentObject` for core service dependencies — it's invisible and crashes at runtime if missing
- Don't create ViewModels inside Views — the Factory creates the ViewModel and passes it to the View
- Don't return `AnyView` from factories unless absolutely necessary (e.g., conditional views) — prefer concrete types
