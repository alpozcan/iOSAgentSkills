---
title: "Task Cancellation and deinit Patterns"
description: "Preventing memory leaks from uncancelled Task references in @Observable ViewModels, including deinit cleanup, nonisolated(unsafe) for @MainActor classes, and weak reference verification in tests."
---

# Task Cancellation and deinit Patterns

## Context

In SwiftUI apps using `@Observable` ViewModels, it's common to create unstructured `Task` instances for debounced search, polling, or background operations. When a ViewModel is deallocated — because the user navigated away or the View was removed from the hierarchy — any active `Task` references must be explicitly cancelled. Swift's unstructured concurrency does not cancel Tasks when their reference is dropped; the Task continues running on the cooperative thread pool until completion. This creates orphaned Tasks that hold URLSession buffers, decoded model objects, and closure captures in memory.

## Pattern

### deinit Cancellation with nonisolated(unsafe)

`@MainActor` classes have a nonisolated `deinit` — Swift 6.0 strict concurrency prevents accessing actor-isolated properties from `deinit`. Mark Task properties as `nonisolated(unsafe)` to allow deinit access:

```swift
@Observable @MainActor
final class SearchViewModel {
    nonisolated(unsafe) private var searchTask: Task<Void, Never>?
    nonisolated(unsafe) private var pollingTask: Task<Void, Never>?

    deinit {
        searchTask?.cancel()
        pollingTask?.cancel()
    }

    func onSearchChanged(query: String) {
        searchTask?.cancel()
        searchTask = Task {
            try? await Task.sleep(for: .milliseconds(400))
            guard !Task.isCancelled else { return }
            await performSearch(query: query)
        }
    }
}
```

`nonisolated(unsafe)` is safe here because `deinit` is the final access point — no concurrent code can read the property after deallocation begins.

### Verifying Cleanup in Tests

Use weak references to verify ViewModels are deallocated when expected:

```swift
@Test("ViewModel deallocates after searchTask cancel")
func viewModelDeallocation() async {
    var vm: SearchViewModel? = SearchViewModel(client: MockClient())
    vm?.onSearchChanged(query: "test")

    weak var weakVM = vm
    vm = nil

    // If searchTask holds a strong reference, weakVM won't be nil
    #expect(weakVM == nil)
}
```

### Structured vs Unstructured Decision

| Scenario | Recommended approach |
|---|---|
| View-lifecycle-bound work | `.task { }` or `.task(id:)` modifier |
| User-triggered debounce in ViewModel | `Task { }` + deinit cancel |
| Background polling in ViewModel | `Task { }` + deinit cancel |
| One-shot load on appear | `.task { }` modifier |

## Edge Cases

- **Task<Void, Error> vs Task<Void, Never>** — If the Task can throw, cancellation propagates through `CancellationError`. If the Task uses `try? await Task.sleep(...)`, the sleep returns early on cancel but the error is silenced — add an explicit `guard !Task.isCancelled` after the sleep.
- **Multiple Task properties** — When a ViewModel has several Task references (search, pagination, refresh), cancel all of them in deinit. Consider grouping them in an array if the count grows.
- **Capture semantics** — A Task created inside a `@MainActor` class implicitly captures `self`. This is fine as long as deinit cancels the Task. If you need the Task to outlive the ViewModel (rare), capture `[weak self]` explicitly.
- **Swift 6.0 strict mode** — Without `nonisolated(unsafe)`, accessing a `@MainActor`-isolated property from `deinit` produces a compiler error. This is a Swift 6.0 strictness improvement that makes this class of bugs more visible.

## Why This Matters

- **Orphaned Tasks accumulate** — Each uncancelled Task holds its closure captures, URLSession response buffers, and decoded objects until the Task completes. In rapid navigation scenarios (search → detail → back → search), Tasks pile up.
- **Instruments visibility** — Orphaned Tasks don't appear as leaks in the Leaks instrument because there's no retain cycle. They show up as "transient" allocations in the Allocations instrument with a growing trend line.
- **Network waste** — An orphaned Task completing a network request consumes bandwidth and server resources for data that will never be displayed.

## Anti-Patterns

- **Dropping Task references without cancelling** — Setting `searchTask = nil` does NOT cancel the Task. Always call `.cancel()` before releasing the reference.
- **Using `weak self` as a substitute for cancellation** — `[weak self]` prevents the closure from retaining the ViewModel, but the Task still runs to completion, consuming CPU and network. Cancel is the correct mechanism.
- **Cancelling in `onDisappear` instead of deinit** — `onDisappear` fires when a View leaves the visible hierarchy but not necessarily when it's deallocated (e.g., tab switches in a TabView keep Views alive). deinit is the definitive cleanup point.
