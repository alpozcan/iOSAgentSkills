---
title: "Scroll Performance and Layout Stability"
description: "Preventing layout thrashing during async image loading in LazyVStack/LazyVGrid, using stable placeholders, Transaction-based animations, and measuring frame drops with Animation Hitches instrument."
---

# Scroll Performance and Layout Stability

## Context

In SwiftUI feeds using `LazyVStack` or `LazyVGrid`, scroll jank occurs when Views change size during async content loading. The most common cause is `AsyncImage` placeholders that have different dimensions than the loaded image — when the image arrives, SwiftUI's layout engine recalculates the entire parent View hierarchy. In a grid with dozens of cards loading simultaneously, these cascading relayouts exceed the frame budget (16.6ms at 60 FPS, 8.3ms at 120 FPS) and produce visible hitches.

## Pattern

### Stable-Size Placeholders

Ensure placeholder Views occupy exactly the same space as the final content:

```swift
AsyncImage(url: url, transaction: Transaction(animation: .easeIn(duration: 0.2))) { phase in
    switch phase {
    case .success(let image):
        image
            .resizable()
            .aspectRatio(contentMode: .fill)
    case .failure:
        Rectangle()
            .fill(Color(.systemGray5))
            .overlay {
                Image(systemName: "photo")
                    .foregroundStyle(.secondary)
            }
    default:
        Rectangle()
            .fill(Color(.systemGray6))
    }
}
.frame(height: 200)  // Fixed height — no layout shift
.clipShape(RoundedRectangle(cornerRadius: 12))
```

The `Rectangle` placeholder fills the parent-proposed size. When the image loads, it replaces the Rectangle at the same dimensions — zero layout recalculation.

### Transaction-Based Fade-In

`Transaction(animation:)` on `AsyncImage` animates the phase transition (placeholder → image) without triggering a layout invalidation:

```swift
AsyncImage(url: url, transaction: Transaction(animation: .easeIn(duration: 0.2))) { phase in
    // ...
}
```

This is lighter than wrapping the image in `.animation()` or `.withAnimation()`, which can trigger broader View tree invalidation.

### Equal-Height Grid Cards

In `LazyVGrid`, cards in the same row should have equal height to prevent row-level relayout when one card's content loads:

```swift
VStack(alignment: .leading, spacing: 8) {
    // Image area — fixed height
    RemoteImage(urlString: imageUrl)
        .frame(height: 140)

    // Text area — fills remaining space
    VStack(alignment: .leading, spacing: 4) {
        Text(title)
            .font(.headline)
            .lineLimit(2)
        Text(summary)
            .font(.caption)
            .lineLimit(3)
    }
    .frame(maxHeight: .infinity, alignment: .top)
}
```

`.frame(maxHeight: .infinity, alignment: .top)` on the text container ensures all cards in a grid row stretch to the same height, regardless of how many lines of text they contain.

### Measuring Frame Drops

Use Instruments' **Animation Hitches** to quantify scroll performance:

1. Profile the app in **Release** configuration (Debug builds disable optimizations and inflate timing)
2. Open the Animation Hitches instrument
3. Scroll through the feed for 30 seconds
4. Check the **Hitch Ratio** — total hitch time divided by total scroll time

| Hitch Ratio | User Perception |
|---|---|
| < 1% | Smooth, no visible jank |
| 1-5% | Occasional subtle stutters |
| > 5% | Clearly janky, needs fixing |

Compare hitch ratio before and after placeholder changes to validate the optimization.

## Edge Cases

- **Variable-height content below images** — Text with different line counts causes row height variation in grids. Use `lineLimit` on all text elements and `.frame(maxHeight: .infinity)` on the text container to equalize row heights.
- **ProMotion displays (120 FPS)** — Frame budget halves to 8.3ms. Layout thrashing that's invisible at 60 FPS becomes noticeable at 120 FPS. Test on Pro-model iPhones.
- **Dynamic Type size changes** — When the user changes text size in Settings, all Views are re-laid-out. This is expected and unavoidable — but it should happen once, not on every scroll frame. Ensure placeholder sizes don't depend on dynamic text metrics.
- **`.drawingGroup()` tradeoffs** — Rasterizes the View subtree into a single Metal texture. Helps with complex Views (gradients, shadows, blurs) but adds overhead for simple Views. Measure before applying broadly.
- **LazyVStack vs List** — `List` has built-in cell recycling and prefetching optimized by Apple. `LazyVStack` gives more layout control but lacks these optimizations. For very long lists (1000+ items), `List` may perform better despite less visual flexibility.

## Why This Matters

- **User perception** — Scroll smoothness is one of the first things users notice. Even small hitches feel "cheap" compared to native Apple apps that consistently hit 120 FPS.
- **App Store review** — Performance issues during review can lead to rejection, especially on older test devices that Apple uses.
- **Battery impact** — Layout thrashing keeps the CPU busy during scroll, increasing energy usage. Instruments' Energy Log confirms this: sustained high CPU during scroll correlates with faster battery drain.

## Anti-Patterns

- **ProgressView as image placeholder** — `ProgressView()` has ~20pt intrinsic size. Replacing it with a 200pt image triggers full parent relayout. Use `Rectangle` or `Color` that fills proposed size.
- **`.animation()` on individual AsyncImage** — Broad animation modifiers can invalidate the parent View's layout. Use `Transaction` on `AsyncImage` for scoped animation.
- **Unbounded text without lineLimit** — Text Views without `lineLimit` expand to fit content, causing row height changes as data loads. Always set `lineLimit` in feed cards.
- **Loading indicators inside card frames** — A spinner that appears and disappears changes the card's content size. If you need a loading indicator, overlay it on a fixed-size placeholder rather than inserting it into the layout flow.
- **Profiling in Debug configuration** — Debug builds disable optimizations (`-Onone`), making layout code 2-10x slower. Performance measurements in Debug are not representative. Always profile in Release.
