---
title: "Object Lifecycle, Memory Leak, and Thread Safety Audit"
description: "Systematic audit patterns for iOS apps: retain cycle detection, memory leak prevention, thread safety enforcement, and crash prevention. Covers @Observable, MKMapView coordinators, NSCache, Timer cleanup, @MainActor isolation, and force-unwrap elimination."
---

# Object Lifecycle, Memory Leak, and Thread Safety Audit

## Context
Production iOS apps accumulate subtle lifecycle bugs, memory leaks, and thread safety violations as features are added. This skill provides a systematic audit checklist and fix patterns for the three categories that cause the most crashes and performance degradation.

## Phase 1: Object Lifecycle & Memory Leaks

### 1.1 Retain Cycle Detection

**Closures stored on objects must use `[weak self]`:**
```swift
// BAD: Strong capture creates cycle if object holds the closure
viewModel.onComplete = { self.dismiss() }

// GOOD: Weak capture breaks the cycle
viewModel.onComplete = { [weak self] in self?.dismiss() }
```

**Check these common patterns:**
- `DispatchQueue.main.asyncAfter` closures on ViewModels
- `Timer.scheduledTimer` closures on services
- Stored closures (`var onTap: (() -> Void)?`) on @Observable classes
- `NotificationCenter.addObserver` with closure API
- Combine `sink` without storing `AnyCancellable`

**UIViewRepresentable Coordinators:**
```swift
// BAD: Coordinator holds strong ref to ViewModel
final class Coordinator: NSObject {
    var viewModel: MapViewModel  // Strong ref

// GOOD: Weak ref since SwiftUI owns the ViewModel's lifecycle
final class Coordinator: NSObject {
    weak var viewModel: MapViewModel?  // Weak ref
```

### 1.2 Image Memory Management

**NSCache over Dictionary for image caches:**
```swift
// BAD: Unbounded, never evicted
private var imageCache: [String: UIImage] = [:]

// GOOD: Auto-evicts under memory pressure
private let imageCache: NSCache<NSString, UIImage> = {
    let cache = NSCache<NSString, UIImage>()
    cache.countLimit = 15
    cache.totalCostLimit = 10 * 1024 * 1024  // 10MB
    return cache
}()
```

**Asset catalog images:**
```swift
// UIImage(named:) caches permanently — fine for small UI assets
// For large images (maps, backgrounds), use file-based loading:
UIImage(contentsOfFile: Bundle.main.path(forResource: "LargeMap", ofType: "png")!)
// This allows the system to reclaim memory under pressure
```

### 1.3 Timer Cleanup

```swift
@Observable @MainActor
final class PlaybackService {
    // nonisolated(unsafe) allows deinit access from any isolation
    nonisolated(unsafe) private var timer: Timer?

    deinit {
        timer?.invalidate()
    }

    func start() {
        timer = Timer.scheduledTimer(withTimeInterval: 3.0, repeats: true) { [weak self] _ in
            DispatchQueue.main.async { self?.advance() }
        }
    }

    func stop() {
        timer?.invalidate()
        timer = nil
    }
}
```

### 1.4 Singleton/Shared Instance Closures

Even singletons should use `[weak self]` in dispatched closures:
```swift
// Follows best practices even though singleton won't dealloc
DispatchQueue.global(qos: .utility).async { [weak self] in
    self?.configureSDK()
}
```

## Phase 2: Thread Safety

### 2.1 @MainActor Isolation Rules

| Type | Isolation | Reason |
|------|-----------|--------|
| **ViewModel** | `@MainActor` | Mutates UI-bound @Observable properties |
| **Service with @Observable** | `@MainActor` | Properties observed by SwiftUI views |
| **Data service (struct)** | None (Sendable) | Immutable data, no mutation |
| **Network service** | None + async | Network calls are inherently async |
| **ServiceProvider** | `@MainActor` | Creates @MainActor-isolated services |

### 2.2 Static Data Caches

```swift
// SAFE: Swift guarantees static let runs exactly once, thread-safely
struct DataService: Sendable {
    private static let _cached: [Item] = buildItems()
    let items: [Item]
    init() { items = Self._cached }
}
```

`nonisolated(unsafe)` is acceptable here because `static let` initialization is atomic. However, a regular `static let` with a closure initializer is equally safe and doesn't require the `unsafe` annotation:

```swift
private static let _cached: [Item] = {
    buildItems()
}()
```

### 2.3 NSCache Thread Safety

`NSCache` is thread-safe — no external locking needed for `object(forKey:)` / `setObject(_:forKey:)`. But auxiliary state (like in-flight tracking) still needs locking:

```swift
final class ImageService: @unchecked Sendable {
    private let cache = NSCache<NSString, UIImage>()  // Thread-safe
    private let lock = NSLock()
    private var inFlight: Set<String> = []  // Needs lock

    func load(name: String) async {
        if cache.object(forKey: name as NSString) != nil { return }  // No lock needed

        lock.lock()
        let alreadyLoading = inFlight.contains(name)
        if !alreadyLoading { inFlight.insert(name) }
        lock.unlock()
        guard !alreadyLoading else { return }

        defer { lock.withLock { inFlight.remove(name) } }
        // ... download and cache
    }
}
```

### 2.4 DispatchQueue.main.asyncAfter Patterns

```swift
// Always use [weak self] — the delayed work may fire after deallocation
DispatchQueue.main.asyncAfter(deadline: .now() + 0.6) { [weak self] in
    self?.selectedItem = item
    self?.showDetail = true
}
```

## Phase 3: Crash Prevention

### 3.1 Force Unwrap Elimination

Search for and fix all `!` that operate on user data or external input:

```swift
// BAD: Crash if array is empty
let first = items[0]
let forced = optionalValue!

// GOOD: Safe access
guard let first = items.first else { return }
if let value = optionalValue { ... }
```

**Acceptable force unwraps:**
- `Bundle.main.url(forResource:)!` for known bundled resources
- `UIImage(systemName:)!` for known SF Symbols
- `try! JSONDecoder().decode(...)` for compile-time-verified data

### 3.2 Array Bounds Checking

```swift
// BAD: Index might be out of range
func jumpToIndex(_ index: Int) {
    currentIndex = index  // Crash if index >= count
}

// GOOD: Clamp to valid range
func jumpToIndex(_ index: Int) {
    guard !items.isEmpty else { return }
    currentIndex = min(max(index, 0), items.count - 1)
}
```

### 3.3 Optional Chaining Through Weak References

After making a reference `weak`, update ALL access sites:
```swift
// When changing `var viewModel: VM` to `weak var viewModel: VM?`
// Every `viewModel.foo` must become `viewModel?.foo`
```

## Audit Checklist

Run through this checklist for every @Observable class and service:

- [ ] **Stored closures** — all use `[weak self]` or `[weak target]`
- [ ] **Timer** — invalidated in `deinit`, closure uses `[weak self]`
- [ ] **Coordinator** — holds weak refs to ViewModel and parent view
- [ ] **Image cache** — uses NSCache with limits, not unbounded Dictionary
- [ ] **@MainActor** — on all ViewModels and @Observable services that bind to UI
- [ ] **deinit** — can access all properties it needs (nonisolated(unsafe) for @MainActor timers)
- [ ] **Static caches** — initialized once, read-only after init
- [ ] **Force unwraps** — none on user data or external input
- [ ] **Array access** — bounds-checked for all index-based access
- [ ] **async closures** — results dispatched to @MainActor when mutating observable state

## Test Integration

Add `@MainActor` to test suites that create `@MainActor`-isolated services:
```swift
@Suite("Service Tests")
@MainActor
struct ServiceTests {
    @Test func testService() {
        let service = MainActorService()  // Works because test is @MainActor
    }
}
```

## Connections

- [[06-actor-based-concurrency-patterns]] — actor isolation rules
- [[25-performance-monitoring-and-profiling-patterns]] — memory profiling with Instruments
- [[34-solid-mvvm-view-decomposition]] — ViewModel boundaries affect lifecycle ownership
- [[05-swift-testing-and-tdd-patterns]] — testing @MainActor services
