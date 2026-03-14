---
title: "AsyncImage Cache and Memory Management"
description: "Configuring URLCache limits for AsyncImage in infinite scroll feeds, understanding default cache behavior, and choosing between AsyncImage and custom image loading for high-volume scenarios."
---

# AsyncImage Cache and Memory Management

## Context

SwiftUI's `AsyncImage` uses `URLSession.shared` internally, which means all image loading shares the system's default `URLCache.shared`. In infinite scroll feeds where users scroll through hundreds of images, the default cache configuration can lead to unbounded memory growth. The cache stores both API responses and image data in the same pool, and eviction timing is not deterministic. Explicitly configuring cache limits prevents memory warnings and jetsam kills on older devices.

## Pattern

### Setting URLCache Limits

Configure `URLCache.shared` early in the app lifecycle — typically in the API client initializer or the `@main` App struct:

```swift
// Set once at app startup or API client init
URLCache.shared.memoryCapacity = 50 * 1024 * 1024   // 50 MB
URLCache.shared.diskCapacity = 200 * 1024 * 1024     // 200 MB
```

These values provide a balance between cache hit rate and memory control:
- **50 MB memory** — Holds ~100-250 compressed images in memory for instant display on re-scroll
- **200 MB disk** — Enables offline/re-launch cache hits without re-downloading

### Stable Placeholders to Prevent Layout Thrashing

`AsyncImage` placeholders must match the final image dimensions to prevent layout recalculation during loading:

```swift
AsyncImage(url: url, transaction: Transaction(animation: .easeIn(duration: 0.2))) { phase in
    switch phase {
    case .success(let image):
        image
            .resizable()
            .aspectRatio(contentMode: .fill)
    case .failure:
        fallbackView
    default:
        if url != nil {
            Rectangle()
                .fill(Color(.systemGray6))
        } else {
            fallbackView
        }
    }
}
.clipShape(RoundedRectangle(cornerRadius: 8))
```

Key details:
- **Rectangle placeholder** fills the parent frame without intrinsic size — no layout shift when the image loads
- **Transaction animation** provides a smooth fade-in without triggering a full layout pass
- **Nil URL check** avoids entering the loading state when there's nothing to load

### Monitoring Cache Usage

Add debug logging to track cache behavior during development:

```swift
#if DEBUG
func logCacheUsage() {
    let cache = URLCache.shared
    let memMB = cache.currentMemoryUsage / 1024 / 1024
    let diskMB = cache.currentDiskUsage / 1024 / 1024
    print("URLCache — memory: \(memMB) MB / \(cache.memoryCapacity / 1024 / 1024) MB")
    print("URLCache — disk: \(diskMB) MB / \(cache.diskCapacity / 1024 / 1024) MB")
}
#endif
```

## Edge Cases

- **Shared cache contention** — `URLCache.shared` serves both API JSON responses and image data. If API responses are large (paginated lists with embedded content), they compete with images for cache space. Consider using separate URLSession configurations for API and image traffic if cache hit rates drop.
- **Cache eviction timing** — URLCache uses LRU eviction but does not evict on every request. Temporary memory usage may exceed the configured `memoryCapacity` before eviction runs. The configured limit is a target, not a hard ceiling.
- **AsyncImage does not expose cache policy** — You cannot set per-request cache policies with `AsyncImage`. If you need `returnCacheDataElseLoad` or `reloadIgnoringLocalCacheData`, build a custom image loader using `URLSession` directly.
- **Duplicate cache entries for resized images** — If the same image URL is used at different sizes in different Views, URLCache stores the full response once but `AsyncImage` decodes it separately each time. This is a CPU cost, not a memory cost, since decoded `UIImage`/`NSImage` instances are not cached by URLCache.
- **Low-memory pressure** — When iOS sends memory warnings, URLCache automatically evicts memory-cached entries. Disk cache persists. Apps should not manually call `removeAllCachedResponses()` on memory warning — let the system manage eviction.

## Why This Matters

- **Jetsam prevention** — On devices with 3-4 GB RAM (iPhone SE, older iPads), unbounded image caching can push total app memory past jetsam limits, causing silent termination.
- **Scroll performance** — Cache hits display images instantly without network delay or decode overhead. Proper cache sizing directly improves perceived scroll smoothness.
- **Network efficiency** — Disk cache persists across app launches. Images loaded in a previous session display from cache on relaunch, reducing cellular data usage.

## Anti-Patterns

- **Not setting any URLCache limits** — Relying on system defaults works for small apps but causes unpredictable memory behavior in image-heavy feeds. Always set explicit limits.
- **Setting memoryCapacity to 0** — This forces every image to be loaded from disk or network, destroying scroll performance. Keep a reasonable memory cache.
- **Calling `URLCache.shared.removeAllCachedResponses()` on refresh** — This nukes the entire cache including images that are still visible on screen. If you need to invalidate specific responses, use `removeCachedResponse(for:)` with targeted URLRequests.
- **Using ProgressView as AsyncImage placeholder** — ProgressView has a small intrinsic size (~20pt) that causes layout thrashing when replaced by the full-size image. Use a Rectangle or Color placeholder that fills the available space.
