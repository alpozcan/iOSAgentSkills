---
title: "Pagination Task Lifecycle Management"
description: "Replacing fire-and-forget Task creation in onAppear with .task(id:) modifier for automatic cancellation, and ViewModel-level task tracking for complex pagination scenarios with multiple data streams."
---

# Pagination Task Lifecycle Management

## Context

Infinite scroll in SwiftUI typically triggers pagination via `onAppear` on the last item. The common pattern — `onAppear { Task { await viewModel.loadMore() } }` — creates unstructured Tasks with no reference tracking. These fire-and-forget Tasks cannot be cancelled when the user changes filters, performs a search, or navigates away. In feeds with multiple content streams (stories, activities, profiles), untracked pagination Tasks can produce race conditions, duplicate requests, and steadily growing memory usage.

## Pattern

### .task(id:) for View-Driven Pagination

Replace `onAppear { Task { } }` with `.task(id:)` to bind pagination to SwiftUI's structured concurrency:

```swift
ForEach(items) { item in
    ItemRow(item: item)
        .task(id: item.id) {
            if item == items.last {
                await viewModel.loadMore()
            }
        }
}
```

Key differences from `onAppear` + `Task`:
- **Automatic cancellation** — When the View leaves the hierarchy, the Task is cancelled. Navigation, filter changes, and tab switches all trigger cleanup.
- **Deduplication via id** — The Task only restarts when `item.id` changes. If `onAppear` fires multiple times for the same item (orientation change, Dynamic Type adjustment), the Task is not recreated.
- **Cooperative cancellation** — `loadMore()` can check `Task.isCancelled` to bail out early if the user navigated away mid-request.

### ViewModel Task Tracking for Complex Scenarios

When pagination involves multiple streams or needs to be cancelled from business logic (filter change, refresh), track the Task in the ViewModel:

```swift
@Observable @MainActor
final class FeedViewModel {
    nonisolated(unsafe) private var paginationTask: Task<Void, Never>?

    func loadMore() async {
        guard paginationTask == nil else { return }
        paginationTask = Task {
            defer { paginationTask = nil }
            guard let cursor = currentCursor else { return }
            do {
                let response = try await client.fetchItems(cursor: cursor)
                items.append(contentsOf: response.data)
                currentCursor = response.cursor
            } catch {
                if !Task.isCancelled {
                    self.error = error.localizedDescription
                }
            }
        }
        await paginationTask?.value
    }

    func refresh() async {
        paginationTask?.cancel()
        paginationTask = nil
        items = []
        currentCursor = nil
        await load()
    }

    deinit {
        paginationTask?.cancel()
    }
}
```

The `guard paginationTask == nil` check prevents duplicate requests. `defer { paginationTask = nil }` clears the reference on completion, enabling the next pagination call. `refresh()` cancels any in-flight pagination before resetting state.

### Combining Both Approaches

Use `.task(id:)` at the View layer for lifecycle management, and ViewModel task tracking for business logic:

```swift
// View: lifecycle-aware trigger
.task(id: item.id) {
    if item == items.last {
        await viewModel.loadMore()
    }
}

// ViewModel: business logic guard + cancellation on refresh
func loadMore() async {
    guard paginationTask == nil else { return }
    // ...
}
```

## Edge Cases

- **onAppear firing multiple times** — SwiftUI may call `onAppear` more than once for the same View (cell recycling in LazyVStack, orientation changes). `.task(id:)` handles this by not restarting if the id hasn't changed. If using `onAppear`, add a guard in the ViewModel.
- **Rapid filter switching** — User taps "Stories", then immediately "Activities". The Stories pagination Task should be cancelled before Activities pagination begins. With ViewModel task tracking, `setFilter()` can cancel `paginationTask` before loading new data.
- **Empty cursor** — Always guard against nil cursors before starting pagination. An API that returns no cursor means there are no more pages.
- **Search + pagination interaction** — Search results and feed content use different pagination streams. Ensure search pagination cancels feed pagination and vice versa, or use separate Task properties for each stream.

## Why This Matters

- **Network efficiency** — Cancelled Tasks that propagate cancellation to URLSession prevent completing requests whose results will be discarded.
- **Data consistency** — Untracked Tasks completing after a filter change can append stale data to the wrong list.
- **Memory predictability** — Each active Task holds response buffers and decoded objects. Cancelling unnecessary Tasks keeps the allocation count stable during long scroll sessions.

## Anti-Patterns

- **`onAppear { Task { } }` without any tracking** — Creates Tasks that cannot be cancelled, deduplicated, or debugged. Use `.task(id:)` instead.
- **Using `isLoading` flag as the sole deduplication guard** — Race conditions between Task creation and flag setting can still allow duplicate requests. Combine with nil-checking the Task reference.
- **Cancelling pagination on every scroll event** — Over-aggressive cancellation prevents pagination from ever completing. Cancel only on meaningful state changes (filter, search, refresh, navigation).
- **Ignoring `Task.isCancelled` in error handling** — A cancelled network request throws `URLError.cancelled`. Without filtering, this error appears as a user-visible error message. Check `Task.isCancelled` before setting error state.
