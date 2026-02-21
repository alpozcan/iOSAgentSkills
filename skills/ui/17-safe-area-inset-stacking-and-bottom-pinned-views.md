---
title: "safeAreaInset Stacking and Bottom-Pinned Views in Custom Tab Bar Apps"
description: "Why nested safeAreaInset(edge: .bottom) hides inner views and how to fix it. Documents five failing approaches and the solution: a single safeAreaInset block with conditional per-tab content."
---

# safeAreaInset Stacking and Bottom-Pinned Views in Custom Tab Bar Apps

## Context

When you replace the system `TabView` with a custom tab bar using `safeAreaInset(edge: .bottom)` (per [[13-swiftui-custom-tab-bar-and-navigation]]), you encounter a critical layout challenge: **how to pin additional chrome (like a chat input bar) above the tab bar on specific tabs**. This is one of the most common and difficult layout problems in SwiftUI apps with custom navigation, and incorrect approaches silently fail — the view renders but is invisible, with no compiler errors or runtime warnings.

This skill documents what works, what doesn't, and why — based on real debugging sessions against iOS 26 / Xcode 26.

## The Problem

```
┌──────────────────────┐
│   Navigation Bar     │
├──────────────────────┤
│                      │
│   ScrollView         │
│   (messages)         │
│                      │
├──────────────────────┤  ← input bar should appear here
│   Custom Tab Bar     │  ← safeAreaInset(edge: .bottom) from MainTabView
├──────────────────────┤
│   Home Indicator     │
└──────────────────────┘
```

The custom tab bar is injected at the `MainTabView` level:

```swift
// MainTabView.swift
var body: some View {
    tabContent
        .safeAreaInset(edge: .bottom) {
            CustomTabBar(selectedTab: $selectedTab)
                .ignoresSafeArea(.keyboard, edges: .bottom)
        }
}
```

Inside the chat tab, you need a persistent input bar (styled with [[04-design-system-as-core-module|design system components]]) pinned above the tab bar. The naive approaches all fail.

## What Does NOT Work

### ❌ Approach 1: VStack with ScrollView + InputBar

```swift
// ChatView.swift — DOES NOT WORK
var body: some View {
    VStack(spacing: 0) {
        ScrollView { /* messages */ }
        inputBar  // ← renders behind tab bar, invisible
    }
}
```

**Why it fails:** The `safeAreaInset(edge: .bottom)` from the tab bar adjusts the **safe area insets** of the content view, but the VStack still extends behind the tab bar in Z-order. The tab bar paints on top. The ScrollView respects the safe area for its content inset (scrollable area), but the VStack's non-scrollable children (like `inputBar`) are laid out in the full frame — which extends behind the tab bar. The tab bar's background (`NyxColors.surfaceSecondary.ignoresSafeArea(edges: .bottom)`) covers the input bar.

**Evidence:** Adding `Rectangle().fill(Color.red).frame(height: 60)` as a VStack child shows a thin red strip just above the tab bar — proving the view IS rendered, but the tab bar's background paints over most/all of it.

### ❌ Approach 2: safeAreaInset on the inner ScrollView

```swift
// ChatView.swift — DOES NOT WORK
var body: some View {
    ScrollView { /* messages */ }
        .safeAreaInset(edge: .bottom) {
            inputBar  // ← invisible, eaten by outer safeAreaInset
        }
}
```

**Why it fails:** When `safeAreaInset` is nested (inner on ScrollView, outer on MainTabView), the inner inset's view is positioned relative to the content view's safe area — which has already been reduced by the outer inset. The inner `safeAreaInset` view ends up rendering in the same Z-layer as the content, and the outer tab bar's `safeAreaInset` paints on top of it. The inner view is completely hidden.

### ❌ Approach 3: overlay(alignment: .bottom) on ScrollView

```swift
// ChatView.swift — DOES NOT WORK
var body: some View {
    ScrollView { /* messages */ }
        .overlay(alignment: .bottom) {
            inputBar  // ← invisible, behind tab bar
        }
}
```

**Why it fails:** Overlays are positioned within the view's frame. The view's frame extends behind the tab bar. The overlay renders but the tab bar (which is in a higher `safeAreaInset` layer) covers it.

### ❌ Approach 4: layoutPriority / fixedSize on InputBar in VStack

```swift
inputBar
    .layoutPriority(1)
    .fixedSize(horizontal: false, vertical: true)
    .frame(minHeight: 52)
```

**Why it fails:** The input bar IS being laid out correctly with the right height — but it's still behind the tab bar in Z-order. `layoutPriority` affects how the VStack distributes space, not Z-order or safe area stacking.

### ❌ Approach 5: toolbar(.bottomBar)

```swift
.toolbar {
    ToolbarItem(placement: .bottomBar) {
        inputBar
    }
}
```

**Why it fails:** The system `bottomBar` toolbar placement conflicts with the custom tab bar. SwiftUI either hides it or renders it behind the custom tab bar's `safeAreaInset`.

## What Works

### ✅ Solution: Move the input bar into MainTabView's safeAreaInset

The only reliable approach is to place the input bar **at the same level as the tab bar** — inside the `safeAreaInset(edge: .bottom)` in `MainTabView`. This ensures both the tab bar and the input bar are in the same Z-layer, with the input bar stacked above the tab bar.

```swift
// MainTabView.swift
var body: some View {
    tabContent
        .safeAreaInset(edge: .bottom) {
            VStack(spacing: 0) {
                // Chat input bar — only shown on chat tab
                if selectedTab == .chat {
                    chatInputBar
                }

                // Tab bar — always shown
                CustomTabBar(selectedTab: $selectedTab)
            }
            .ignoresSafeArea(.keyboard, edges: .bottom)
        }
}
```

This approach requires lifting the input bar state (text binding, send action) to the `MainTabView` level or using a shared `ObservableObject`:

```swift
// Shared state between MainTabView and ChatView
@MainActor
final class ChatInputState: ObservableObject {
    @Published var inputText: String = ""
    @Published var isInputVisible: Bool = true
    var onSend: ((String) -> Void)?
}
```

### ✅ Alternative: Expose input bar via ChatView's factory

If lifting state feels too invasive, the ChatView factory can return both the view and the input bar as separate components:

```swift
public struct ChatUIComposer {
    @MainActor
    public func composeChatView() -> ChatView { /* ... */ }

    @MainActor
    public func composeChatInputBar() -> ChatInputBar { /* ... */ }
}
```

Then in MainTabView:

```swift
.safeAreaInset(edge: .bottom) {
    VStack(spacing: 0) {
        if selectedTab == .chat {
            chatUIComposer.composeChatInputBar()
        }
        CustomTabBar(selectedTab: $selectedTab)
    }
    .ignoresSafeArea(.keyboard, edges: .bottom)
}
```

### ✅ Alternative: Use a single safeAreaInset with conditional content

```swift
.safeAreaInset(edge: .bottom) {
    VStack(spacing: 0) {
        // Per-tab bottom chrome
        switch selectedTab {
        case .chat:
            ChatInputBar(text: $chatText, onSend: handleSend)
        default:
            EmptyView()
        }

        // Universal tab bar
        CustomTabBar(selectedTab: $selectedTab)
    }
    .ignoresSafeArea(.keyboard, edges: .bottom)
}
```

## The Root Cause: safeAreaInset Z-Order

`safeAreaInset(edge: .bottom)` does two things:
1. **Adjusts the safe area** of the modified view so content doesn't overlap the inset view
2. **Renders the inset view on top** of the modified view in Z-order

When nested, the outer `safeAreaInset` always wins in Z-order. The inner `safeAreaInset` adjusts the content's safe area correctly, but its view is rendered **underneath** the outer inset view. Since the outer tab bar has an opaque background, it covers the inner input bar completely.

```
Z-order (back to front):
  1. Content view (ScrollView) — furthest back
  2. Inner safeAreaInset view (input bar) — middle
  3. Outer safeAreaInset view (tab bar) — front ← covers #2
```

This is not a bug — it's how `safeAreaInset` composes. The API is designed for a single bottom chrome element per view hierarchy level.

## Rules

- **One `safeAreaInset(edge: .bottom)` per view level.** If you need multiple bottom-pinned elements, stack them inside a single `safeAreaInset`.
- **The outermost `safeAreaInset` always wins in Z-order.** Inner insets adjust spacing but their views are hidden behind the outer one.
- **VStack children below a ScrollView render behind `safeAreaInset` chrome.** The VStack extends behind the inset area; the inset view covers it.
- **Never rely on overlays or inner `safeAreaInset`s to place views above an outer `safeAreaInset`.** They will be invisible.
- **When you add a custom tab bar via `safeAreaInset`, ALL per-tab bottom chrome must live in the same `safeAreaInset`.** This means the tab bar's `safeAreaInset` block must be aware of per-tab input bars, toolbars, or action sheets.
- **Test bottom-pinned views with bright debug colors** (`Color.red`, `Color.orange`) to verify visibility before styling — see [[18-ui-testing-regression-and-smoke]] for regression testing these layouts. If a bright-colored view is invisible, the problem is Z-order, not color/contrast.

## Why This Matters

- **`safeAreaInset` stacking is the #1 custom tab bar layout bug** — it silently hides views with no compiler errors or runtime warnings, wasting hours of debugging
- **Understanding the Z-order model** prevents trial-and-error approaches that never converge on a solution
- **The single-inset pattern** scales to any number of per-tab bottom chrome elements (input bars, toolbars, action sheets) by composing them in one `VStack`
- **iPad multitasking:** In Split View and Slide Over, the safe area changes dynamically. Test bottom-pinned views in all multitasking configurations — `safeAreaInset` handles this correctly, but fixed-height assumptions may break when the window width changes.

## Anti-Patterns

- Don't add `safeAreaInset(edge: .bottom)` inside a view that's already wrapped by another `safeAreaInset(edge: .bottom)` — the inner one will be hidden.
- Don't use VStack to place fixed chrome below a ScrollView when a custom tab bar uses `safeAreaInset` — the chrome renders behind the tab bar.
- Don't assume `.overlay(alignment: .bottom)` renders above `safeAreaInset` chrome — it doesn't.
- Don't spend time debugging height, `layoutPriority`, `fixedSize`, or `frame(minHeight:)` when the view is invisible — the issue is Z-order, not sizing.
- Don't use `.ignoresSafeArea(edges: .bottom)` on the tab bar background hoping to "make room" — this only affects the background fill direction, not Z-order.
