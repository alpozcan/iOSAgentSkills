---
title: "Protocol-Driven Service Catalog Without Third-Party Frameworks"
description: "A complete service catalog in ~80 lines of Swift with KeyPath-based type-safe resolution, NSLock thread safety, shared vs transient registration, and clean reset for test isolation."
---

# Protocol-Driven Service Catalog Without Third-Party Frameworks

## Context
You need compile-time safe service catalog in a modular iOS app where each module defines its own protocol, but registration and resolution happen at the [[01-tuist-modular-architecture|app composition root]]. You want zero third-party DI frameworks.

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
            // [[03-ui-composer-pattern-for-feature-modules|Feature factories]]
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
    func resolve<Spec: Blueprint>(_ keyPath: KeyPath<Catalog, Spec.Type>) throws -> B.Output
}

public enum CatalogError: Error, Sendable {
    case missingEntry(String)
    case typeMismatch(expected: String, actual: String)
    case circularReference(chain: [String])
}

public final class Catalog: Assembling, @unchecked Sendable {
    public static let shared = Catalog()

    private let lock = NSLock()
    private var builders: [ObjectIdentifier: @Sendable (Catalog) throws -> Any] = [:]
    private var sharedInstances: [ObjectIdentifier: Any] = [:]
    private var assemblyChain: [String] = []  // Circular dependency detection

    public func register<Spec: Blueprint>(
        _ specType: Spec.Type,
        builder: @escaping @Sendable (Catalog) throws -> B.Output
    ) {
        let key = ObjectIdentifier(specType)
        lock.lock()
        defer { lock.unlock() }
        builders[key] = { provider in try factory(provider) }
    }

    public func supplyShared<Spec: Blueprint>(
        _ specType: Spec.Type,
        builder: @escaping @Sendable (Catalog) throws -> B.Output
    ) {
        let key = ObjectIdentifier(specType)
        lock.lock()
        defer { lock.unlock() }
        builders[key] = { [weak self] provider in
            guard let self else { throw CatalogError.missingEntry(String(describing: specType)) }
            // Singleton check + creation inside lock to prevent race conditions
            if let existing = self.sharedInstances[key] { return existing }
            let instance = try builder(provider)
            self.sharedInstances[key] = instance
            return instance
        }
    }

    public func resolve<Spec: Blueprint>(
        _ keyPath: KeyPath<Catalog, Spec.Type>
    ) throws -> B.Output {
        let specType = self[keyPath: keyPath]
        let key = ObjectIdentifier(specType)
        let specName = String(describing: specType)

        lock.lock()
        defer { lock.unlock() }

        // Circular dependency detection
        if assemblyChain.contains(specName) {
            let chain = assemblyChain + [specName]
            throw CatalogError.circularReference(chain: chain)
        }
        assemblyChain.append(specName)
        defer { assemblyChain.removeLast() }

        guard let builder = builders[key] else {
            throw CatalogError.missingEntry(specName)
        }
        let instance = try builder(self)
        guard let typed = instance as? B.Output else {
            throw CatalogError.typeMismatch(
                expected: String(describing: B.Output.self),
                actual: String(describing: type(of: instance))
            )
        }
        return typed
    }

    public func reset() {
        lock.lock()
        defer { lock.unlock() }
        builders.removeAll()
        sharedInstances.removeAll()
        assemblyChain.removeAll()
    }
}
```

### Usage Patterns

**Resolution at call site:**
```swift
let service: CalendarServiceProtocol = try Catalog.main[ \.calendarService)
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

## Edge Cases

- **Thread contention at startup:** If `prepareAll()` and `resolve()` race, the lock ensures consistency but resolution will throw `missingEntry` until registration completes. Call `prepareAll()` synchronously in `App.init()` before any view resolves services.
- **Double registration (last-write-wins):** Registering the same spec twice silently overwrites. This is intentional for test overrides but can mask bugs. Add `#if DEBUG` assertions if needed.
- **Circular dependencies:** A resolves B which resolves A. The `assemblyChain` detects this and throws `circularReference(chain:)` instead of stack overflow.
- **Missing registration before `prepareAll()` completes:** If a service is resolved before its spec is registered, you get `missingEntry`. Ensure all registrations happen before the first `resolve()` call.
- **Singleton race condition (fixed above):** The singleton check and creation now happen inside the factory closure while the lock is held, preventing duplicate instance creation under concurrent access.

## Why This Matters

- **No framework dependency** — the entire catalog is ~80 lines of Swift
- **Type-safe resolution** via `KeyPath` — compiler catches typos
- **Thread-safe** via `NSLock` + `@unchecked Sendable` (see [[06-actor-based-concurrency-patterns]])
- **Testable** — `reset()` wipes all registrations for [[05-swift-testing-and-tdd-patterns|clean test state]]; any protocol can be replaced with a mock
- **Registration order matters** — specs are listed in dependency order in `allBlueprints()`

## Anti-Patterns

- Don't resolve services inside `init()` of value types — resolve lazily or inject via constructor
- Don't access `Catalog.main` from background threads without ensuring registration is complete
- Don't create circular dependencies between specs (A needs B, B needs A)
- Don't register services conditionally based on runtime state in production — use separate mock registrations only for debug/test builds
