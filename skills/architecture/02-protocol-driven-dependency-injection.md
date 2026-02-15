# Protocol-Driven Dependency Injection Without Third-Party Frameworks

## Context
You need compile-time safe dependency injection in a modular iOS app where each module defines its own protocol, but registration and resolution happen at the app composition root. You want zero third-party DI frameworks.

## Pattern

### Three-Layer DI Architecture

**Layer 1: Protocol** (in the module that owns the service)
```swift
// CalendarService/Sources/CalendarService.swift
public protocol CalendarServiceProtocol: Sendable {
    func fetchEvents(from: Date, to: Date) async throws -> [CalendarEvent]
    func requestAccess() async -> Bool
}
```

**Layer 2: DependencySpec** (in the same module — describes how to construct the service)
```swift
// CalendarService/Sources/CalendarServiceSpec.swift
public struct CalendarServiceSpec: DependencySpec {
    public typealias Service = CalendarServiceProtocol
    public init() {}
    
    public static func make(resolver: DependencyResolving) -> CalendarServiceProtocol {
        CalendarService()
    }
}

// KeyPath extension for ergonomic resolution
extension ServiceProvider {
    public var calendarService: CalendarServiceSpec.Type { CalendarServiceSpec.self }
}
```

**Layer 3: Registration** (in the app target — the composition root)
```swift
// App/Sources/ServiceProvider+Registry.swift
extension ServiceProvider {
    public func registerAll() {
        allSpecs().forEach { spec in registerSpec(spec) }
    }
    
    private func allSpecs() -> [any DependencySpec] {
        [
            CalendarServiceSpec(),
            CalendarStoreSpec(),
            LLMEngineSpec(),
            // Feature factories
            ChatUISpec(),
            InsightsUISpec(),
        ]
    }
    
    private func registerSpec<Spec: DependencySpec>(_ spec: Spec) {
        register(Spec.self) { _ in Spec.make(resolver: self) }
    }
}
```

### The ServiceProvider Core

```swift
public protocol DependencySpec {
    associatedtype Service
    static func make(resolver: DependencyResolving) -> Service
}

public protocol DependencyResolving: AnyObject {
    func resolve<Spec: DependencySpec>(_ keyPath: KeyPath<ServiceProvider, Spec.Type>) -> Spec.Service
}

public final class ServiceProvider: DependencyResolving, @unchecked Sendable {
    public static let shared = ServiceProvider()
    
    private let lock = NSLock()
    private var factories: [ObjectIdentifier: @Sendable (ServiceProvider) -> Any] = [:]
    private var singletons: [ObjectIdentifier: Any] = [:]
    
    public func register<Spec: DependencySpec>(
        _ specType: Spec.Type,
        factory: @escaping @Sendable (ServiceProvider) -> Spec.Service
    ) {
        let key = ObjectIdentifier(specType)
        lock.lock()
        factories[key] = { provider in factory(provider) }
        lock.unlock()
    }
    
    public func registerSingleton<Spec: DependencySpec>(
        _ specType: Spec.Type,
        factory: @escaping @Sendable (ServiceProvider) -> Spec.Service
    ) {
        let key = ObjectIdentifier(specType)
        lock.lock()
        factories[key] = { provider in
            if let existing = provider.singletons[key] { return existing }
            let instance = factory(provider)
            provider.singletons[key] = instance
            return instance
        }
        lock.unlock()
    }
    
    public func resolve<Spec: DependencySpec>(
        _ keyPath: KeyPath<ServiceProvider, Spec.Type>
    ) -> Spec.Service {
        let specType = self[keyPath: keyPath]
        let key = ObjectIdentifier(specType)
        lock.lock()
        let factory = factories[key]
        lock.unlock()
        guard let factory else { fatalError("No factory for \(specType)") }
        return factory(self) as! Spec.Service
    }
    
    public func reset() {
        lock.lock()
        factories.removeAll()
        singletons.removeAll()
        lock.unlock()
    }
}
```

### Usage Patterns

**Resolution at call site:**
```swift
let service: CalendarServiceProtocol = ServiceProvider.shared.resolve(\.calendarService)
```

**Resolution via DependencySpec chaining (one service depending on another):**
```swift
public struct CalendarStoreSpec: DependencySpec {
    public typealias Service = CalendarStoreProtocol
    public static func make(resolver: DependencyResolving) -> CalendarStoreProtocol {
        let calendarService: CalendarServiceProtocol = resolver.resolve(\.calendarService)
        return CalendarStore(calendarService: calendarService)
    }
}
```

**Override for testing:**
```swift
ServiceProvider.shared.register(CalendarServiceSpec.self) { _ in
    MockCalendarService()
}
```

### Testing the DI Container Itself

```swift
@Suite("ServiceProvider Tests")
struct ServiceProviderTests {
    init() { ServiceProvider.shared.reset() }
    
    @Test("Resolve calls factory each time (not singleton)")
    func resolveCallsFactoryEachTime() {
        var count = 0
        ServiceProvider.shared.register(TestSpec.self) { _ in
            count += 1
            return TestServiceImpl(value: "call-\(count)")
        }
        let s1 = ServiceProvider.shared.resolve(TestSpec.self)
        let s2 = ServiceProvider.shared.resolve(TestSpec.self)
        #expect(s1.value == "call-1")
        #expect(s2.value == "call-2")
    }
    
    @Test("Singleton registration caches instance")
    func singletonRegistration() {
        var count = 0
        ServiceProvider.shared.registerSingleton(TestSpec.self) { _ in
            count += 1
            return TestServiceImpl(value: "singleton-\(count)")
        }
        let s1 = ServiceProvider.shared.resolve(TestSpec.self)
        let s2 = ServiceProvider.shared.resolve(TestSpec.self)
        #expect(s1.value == "singleton-1")
        #expect(s2.value == "singleton-1")
    }
}
```

## Why This Matters

- **No framework dependency** — the entire DI system is ~80 lines of Swift
- **Type-safe resolution** via `KeyPath` — compiler catches typos
- **Thread-safe** via `NSLock` + `@unchecked Sendable`
- **Testable** — `reset()` wipes all registrations for clean test state; any protocol can be replaced with a mock
- **Registration order matters** — specs are listed in dependency order in `allSpecs()`

## Anti-Patterns

- Don't resolve services inside `init()` of value types — resolve lazily or inject via constructor
- Don't access `ServiceProvider.shared` from background threads without ensuring registration is complete
- Don't create circular dependencies between specs (A needs B, B needs A)
- Don't register services conditionally based on runtime state in production — use separate mock registrations only for debug/test builds
