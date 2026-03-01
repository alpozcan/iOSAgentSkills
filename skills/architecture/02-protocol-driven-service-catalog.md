---
title: "Protocol-Driven Service Catalog Without Third-Party Frameworks"
description: "A complete service catalog in ~80 lines of Swift with KeyPath-based type-safe obtainment, NSLock thread safety, shared vs transient supply, and clean clear() for test isolation."
---

# Protocol-Driven Service Catalog Without Third-Party Frameworks

## Context
You need compile-time safe service assembly in a modular iOS app where each module defines its own protocol, but registration and obtainment happen at the [[01-tuist-modular-architecture|app composition root]]. You want zero third-party frameworks.

## Pattern

### Three-Layer Catalog Architecture

**Layer 1: Protocol** (in the module that owns the service)
```swift
// CalendarService/Sources/CalendarService.swift
public protocol CalendarServiceProtocol: Sendable {
    func fetchEvents(from: Date, to: Date) async throws -> [CalendarEvent]
    func requestAccess() async -> Bool
}
```

**Layer 2: Blueprint** (in the same module — describes how to assemble the service)
```swift
// CalendarService/Sources/CalendarServiceBlueprint.swift
public struct CalendarServiceBlueprint: Blueprint {
    public typealias Output = CalendarServiceProtocol
    public init() {}

    public static func assemble(from catalog: Assembling) -> CalendarServiceProtocol {
        CalendarService()
    }
}

// KeyPath extension for ergonomic obtainment
extension Catalog {
    public var calendarService: CalendarServiceBlueprint.Type { CalendarServiceBlueprint.self }
}
```

**Layer 3: Preparation** (in the app target — the composition root)
```swift
// App/Sources/Catalog+Entries.swift
extension Catalog {
    public func prepareAll() {
        allBlueprints().forEach { blueprint in installBlueprint(blueprint) }
    }

    private func allBlueprints() -> [any Blueprint] {
        [
            CalendarServiceBlueprint(),
            CalendarStoreBlueprint(),
            LLMEngineBlueprint(),
            // [[03-ui-composer-pattern-for-feature-modules|Feature composers]]
            ChatUIBlueprint(),
            InsightsUIBlueprint(),
        ]
    }

    private func installBlueprint<B: Blueprint>(_ blueprint: B) {
        supply(B.self) { _ in B.assemble(from: self) }
    }
}
```

### The Catalog Core

```swift
public protocol Blueprint {
    associatedtype Output
    static func assemble(from catalog: Assembling) -> Output
}

public protocol Assembling: AnyObject {
    func obtain<B: Blueprint>(_ keyPath: KeyPath<Catalog, B.Type>) throws -> B.Output
}

public enum CatalogError: Error, Sendable {
    case missingEntry(String)
    case typeMismatch(expected: String, actual: String)
    case circularReference(chain: [String])
}

public final class Catalog: Assembling, @unchecked Sendable {
    public static let main = Catalog()

    private let lock = NSLock()
    private var builders: [ObjectIdentifier: @Sendable (Catalog) throws -> Any] = [:]
    private var sharedInstances: [ObjectIdentifier: Any] = [:]
    private var assemblyChain: [String] = []  // Circular reference detection

    public func supply<B: Blueprint>(
        _ blueprintType: B.Type,
        builder: @escaping @Sendable (Catalog) throws -> B.Output
    ) {
        let key = ObjectIdentifier(blueprintType)
        lock.lock()
        defer { lock.unlock() }
        builders[key] = { catalog in try builder(catalog) }
    }

    public func supplyShared<B: Blueprint>(
        _ blueprintType: B.Type,
        builder: @escaping @Sendable (Catalog) throws -> B.Output
    ) {
        let key = ObjectIdentifier(blueprintType)
        lock.lock()
        defer { lock.unlock() }
        builders[key] = { [weak self] catalog in
            guard let self else { throw CatalogError.missingEntry(String(describing: blueprintType)) }
            // Shared check + creation inside lock to prevent race conditions
            if let existing = self.sharedInstances[key] { return existing }
            let instance = try builder(catalog)
            self.sharedInstances[key] = instance
            return instance
        }
    }

    public func obtain<B: Blueprint>(
        _ keyPath: KeyPath<Catalog, B.Type>
    ) throws -> B.Output {
        let blueprintType = self[keyPath: keyPath]
        let key = ObjectIdentifier(blueprintType)
        let blueprintName = String(describing: blueprintType)

        lock.lock()
        defer { lock.unlock() }

        // Circular reference detection
        if assemblyChain.contains(blueprintName) {
            let chain = assemblyChain + [blueprintName]
            throw CatalogError.circularReference(chain: chain)
        }
        assemblyChain.append(blueprintName)
        defer { assemblyChain.removeLast() }

        guard let builder = builders[key] else {
            throw CatalogError.missingEntry(blueprintName)
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

    public subscript<B: Blueprint>(keyPath: KeyPath<Catalog, B.Type>) -> B.Output {
        get throws {
            try obtain(keyPath)
        }
    }

    public func clear() {
        lock.lock()
        defer { lock.unlock() }
        builders.removeAll()
        sharedInstances.removeAll()
        assemblyChain.removeAll()
    }
}
```

### Usage Patterns

**Obtainment at call site:**
```swift
let service: CalendarServiceProtocol = try Catalog.main[\.calendarService]
```

**Blueprint chaining (one service depending on another):**
```swift
public struct CalendarStoreBlueprint: Blueprint {
    public typealias Output = CalendarStoreProtocol
    public static func assemble(from catalog: Assembling) -> CalendarStoreProtocol {
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

### Testing the Catalog Itself

```swift
@Suite("Catalog Tests")
struct CatalogTests {
    init() { Catalog.main.clear() }

    @Test("Obtain calls builder each time (not shared)")
    func obtainCallsBuilderEachTime() {
        var count = 0
        Catalog.main.supply(TestBlueprint.self) { _ in
            count += 1
            return TestServiceImpl(value: "call-\(count)")
        }
        let s1 = Catalog.main[\.testService]
        let s2 = Catalog.main[\.testService]
        #expect(s1.value == "call-1")
        #expect(s2.value == "call-2")
    }

    @Test("Shared supply caches instance")
    func sharedSupply() {
        var count = 0
        Catalog.main.supplyShared(TestBlueprint.self) { _ in
            count += 1
            return TestServiceImpl(value: "shared-\(count)")
        }
        let s1 = Catalog.main[\.testService]
        let s2 = Catalog.main[\.testService]
        #expect(s1.value == "shared-1")
        #expect(s2.value == "shared-1")
    }
}
```

## Edge Cases

- **Thread contention at startup:** If `prepareAll()` and `obtain()` race, the lock ensures consistency but obtainment will throw `missingEntry` until preparation completes. Call `prepareAll()` synchronously in `App.init()` before any view obtains services.
- **Double supply (last-write-wins):** Supplying the same blueprint twice silently overwrites. This is intentional for test overrides but can mask bugs. Add `#if DEBUG` assertions if needed.
- **Circular references:** A assembles B which assembles A. The `assemblyChain` detects this and throws `circularReference(chain:)` instead of stack overflow.
- **Missing supply before `prepareAll()` completes:** If a service is obtained before its blueprint is supplied, you get `missingEntry`. Ensure all supplies happen before the first `obtain()` call.
- **Shared instance race condition (fixed above):** The shared check and creation now happen inside the builder closure while the lock is held, preventing duplicate instance creation under concurrent access.

## Why This Matters

- **No framework dependency** — the entire catalog is ~80 lines of Swift
- **Type-safe obtainment** via `KeyPath` — compiler catches typos
- **Thread-safe** via `NSLock` + `@unchecked Sendable` (see [[06-actor-based-concurrency-patterns]])
- **Testable** — `clear()` wipes all entries for [[05-swift-testing-and-tdd-patterns|clean test state]]; any protocol can be replaced with a stub
- **Preparation order matters** — blueprints are listed in assembly order in `allBlueprints()`

## Anti-Patterns

- Don't obtain services inside `init()` of value types — obtain lazily or inject via constructor
- Don't access `Catalog.main` from background threads without ensuring preparation is complete
- Don't create circular references between blueprints (A needs B, B needs A)
- Don't supply services conditionally based on runtime state in production — use separate stub supplies only for debug/test builds
