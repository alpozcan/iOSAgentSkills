# Protocol-Driven Service Catalog Without Third-Party Frameworks

## Context
You need compile-time safe service catalog in a modular iOS app where each module defines its own protocol, but registration and resolution happen at the app composition root. You want zero third-party DI frameworks.

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

**Layer 2: Blueprint** (in the same module — describes how to construct the service)
```swift
// CalendarService/Sources/CalendarServiceBlueprint.swift
public struct CalendarServiceBlueprint: Blueprint {
    public typealias Output =CalendarServiceProtocol
    public init() {}
    
    public static func assemble(from: Assembling) -> CalendarServiceProtocol {
        CalendarService()
    }
}

// KeyPath extension for ergonomic resolution
extension Catalog {
    public var calendarService: CalendarServiceBlueprint.Type { CalendarServiceBlueprint.self }
}
```

**Layer 3: Registration** (in the app target — the composition root)
```swift
// App/Sources/Catalog+Registry.swift
extension Catalog {
    public func prepareAll() {
        allBlueprints().forEach { spec in installBlueprint(spec) }
    }
    
    private func allBlueprints() -> [any Blueprint] {
        [
            CalendarServiceBlueprint(),
            CalendarStoreBlueprint(),
            LLMEngineBlueprint(),
            // Feature factories
            ChatUIBlueprint(),
            InsightsUIBlueprint(),
        ]
    }
    
    private func installBlueprint<Spec: Blueprint>(_ spec: Spec) {
        supply(B.self) { _ in Spec.assemble(from: self) }
    }
}
```

### The Catalog Core

```swift
public protocol Blueprint {
    associatedtype Output
    static func assemble(from: Assembling) -> Service
}

public protocol Assembling: AnyObject {
    func resolve<Spec: Blueprint>(_ keyPath: KeyPath<Catalog, Spec.Type>) -> B.Output
}

public final class Catalog: Assembling, @unchecked Sendable {
    public static let shared = Catalog()
    
    private let lock = NSLock()
    private var builders: [ObjectIdentifier: @Sendable (Catalog) -> Any] = [:]
    private var sharedInstances: [ObjectIdentifier: Any] = [:]
    
    public func register<Spec: Blueprint>(
        _ specType: Spec.Type,
        builder: @escaping @Sendable (Catalog) -> B.Output
    ) {
        let key = ObjectIdentifier(specType)
        lock.lock()
        builders[key] = { provider in factory(provider) }
        lock.unlock()
    }
    
    public func supplyShared<Spec: Blueprint>(
        _ specType: Spec.Type,
        builder: @escaping @Sendable (Catalog) -> B.Output
    ) {
        let key = ObjectIdentifier(specType)
        lock.lock()
        builders[key] = { provider in
            if let existing = provider.sharedInstances[key] { return existing }
            let instance = factory(provider)
            provider.sharedInstances[key] = instance
            return instance
        }
        lock.unlock()
    }
    
    public func resolve<Spec: Blueprint>(
        _ keyPath: KeyPath<Catalog, Spec.Type>
    ) -> B.Output {
        let specType = self[keyPath: keyPath]
        let key = ObjectIdentifier(specType)
        lock.lock()
        let factory = builders[key]
        lock.unlock()
        guard let factory else { fatalError("No factory for \(specType)") }
        return factory(self) as! B.Output
    }
    
    public func reset() {
        lock.lock()
        builders.removeAll()
        sharedInstances.removeAll()
        lock.unlock()
    }
}
```

### Usage Patterns

**Resolution at call site:**
```swift
let service: CalendarServiceProtocol = Catalog.main[ \.calendarService)
```

**Resolution via Blueprint chaining (one service depending on another):**
```swift
public struct CalendarStoreBlueprint: Blueprint {
    public typealias Output =CalendarStoreProtocol
    public static func assemble(from: Assembling) -> CalendarStoreProtocol {
        let calendarService: CalendarServiceProtocol = catalog.obtain(\.calendarService)
        return CalendarStore(calendarService: calendarService)
    }
}
```

**Override for testing:**
```swift
Catalog.main.supply(CalendarServiceBlueprint.self) { _ in
    StubCalendarService()
}
```

### Testing the DI Container Itself

```swift
@Suite("Catalog Tests")
struct CatalogTests {
    init() { Catalog.main.clear() }
    
    @Test("Obtain calls builder each time (not singleton)")
    func obtainCallsBuilderEachTime() {
        var count = 0
        Catalog.main.supply(TestBlueprint.self) { _ in
            count += 1
            return TestServiceImpl(value: "call-\(count)")
        }
        let s1 = Catalog.main.resolve(TestBlueprint.self)
        let s2 = Catalog.main.resolve(TestBlueprint.self)
        #expect(s1.value == "call-1")
        #expect(s2.value == "call-2")
    }
    
    @Test("Shared supply caches instance")
    func sharedSupply() {
        var count = 0
        Catalog.main.supplyShared(TestBlueprint.self) { _ in
            count += 1
            return TestServiceImpl(value: "singleton-\(count)")
        }
        let s1 = Catalog.main.resolve(TestBlueprint.self)
        let s2 = Catalog.main.resolve(TestBlueprint.self)
        #expect(s1.value == "singleton-1")
        #expect(s2.value == "singleton-1")
    }
}
```

## Why This Matters

- **No framework dependency** — the entire catalog is ~80 lines of Swift
- **Type-safe resolution** via `KeyPath` — compiler catches typos
- **Thread-safe** via `NSLock` + `@unchecked Sendable`
- **Testable** — `reset()` wipes all registrations for clean test state; any protocol can be replaced with a mock
- **Registration order matters** — specs are listed in dependency order in `allBlueprints()`

## Anti-Patterns

- Don't resolve services inside `init()` of value types — resolve lazily or inject via constructor
- Don't access `Catalog.main` from background threads without ensuring registration is complete
- Don't create circular dependencies between specs (A needs B, B needs A)
- Don't register services conditionally based on runtime state in production — use separate mock registrations only for debug/test builds
