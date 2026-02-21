---
title: "Accessibility Patterns: VoiceOver, Dynamic Type, and Reduced Motion"
description: "Enforcing VoiceOver traversal order, Dynamic Type with @ScaledMetric, reduced motion fallbacks, WCAG color contrast audits, accessibility identifiers, and automated accessibility testing in a modular Tuist app with a custom design system."
---

# Accessibility Patterns: VoiceOver, Dynamic Type, and Reduced Motion

## Context

iOS apps must support VoiceOver, Dynamic Type, and reduced motion to serve all users and meet App Store accessibility guidelines. In a modular Tuist app with a custom design system ([[04-design-system-as-core-module]]), accessibility must be enforced at the component level — not bolted on after the fact. Because the app uses a custom tab bar ([[13-swiftui-custom-tab-bar-and-navigation]]) instead of the system `TabView`, VoiceOver traversal, traits, and labels must be configured manually for every interactive element.

## Pattern

### 1. VoiceOver Traversal Order

Use `.accessibilityElement(children:)` to control how VoiceOver groups and reads elements:

```swift
// Containers: VoiceOver steps into children individually
VStack {
    headerView
    bodyView
    footerView
}
.accessibilityElement(children: .contain)

// Compound elements (cards): VoiceOver reads as a single unit
HStack {
    Image(systemName: "brain.head.profile")
    VStack(alignment: .leading) {
        Text("Weekly Insight")
        Text("Your focus improved 23%")
    }
}
.accessibilityElement(children: .combine)
// VoiceOver reads: "Weekly Insight, Your focus improved 23%"
```

For the custom tab bar, each tab button needs explicit labels and traits:

```swift
Button {
    selectedTab = tab
} label: {
    VStack(spacing: NyxSpacing.xs) {
        Image(systemName: tab.icon)
        Text(tab.label)
            .font(NyxTypography.caption)
    }
}
.accessibilityLabel("\(tab.label) tab\(isSelected ? ", selected" : "")")
.accessibilityAddTraits(isSelected ? [.isButton, .isSelected] : .isButton)
.accessibilityRemoveTraits(isSelected ? [] : .isSelected)
```

Control reading order with `.accessibilitySortPriority()` — higher values are read first:

```swift
VStack {
    // Error banner should be read first, even though it's visually below the nav bar
    NavigationBar()
        .accessibilitySortPriority(1)

    ErrorBanner("Connection lost")
        .accessibilitySortPriority(2) // Read before nav bar

    ContentView()
        .accessibilitySortPriority(0)
}
```

### 2. Dynamic Type with @ScaledMetric

Wrap design system sizing in `@ScaledMetric(relativeTo:)` so font sizes and spacing scale with the user's preferred text size:

```swift
struct NyxScaledTypography {
    @ScaledMetric(relativeTo: .body) private var bodySize: CGFloat = 15
    @ScaledMetric(relativeTo: .caption) private var captionSize: CGFloat = 12
    @ScaledMetric(relativeTo: .title) private var titleSize: CGFloat = 22
    @ScaledMetric(relativeTo: .largeTitle) private var displaySize: CGFloat = 34
}
```

For compact layouts where extreme scaling would break the design, clamp the range:

```swift
// Limit Dynamic Type range for the tab bar labels
Text(tab.label)
    .font(.system(size: captionSize))
    .dynamicTypeSize(.large ... .accessibility3)
```

For labels that might truncate at large sizes, use `.minimumScaleFactor`:

```swift
Text(insight.title)
    .font(.system(size: titleSize, weight: .semibold))
    .minimumScaleFactor(0.8)
    .lineLimit(2)
```

Apply `@ScaledMetric` to spacing and icon sizes too — not just text:

```swift
struct NyxAccessibleCard<Content: View>: View {
    @ScaledMetric(relativeTo: .body) private var cardPadding: CGFloat = 16
    @ScaledMetric(relativeTo: .body) private var iconSize: CGFloat = 24

    let content: Content

    var body: some View {
        content
            .padding(cardPadding)
            .background(NyxColors.surfaceElevated)
            .clipShape(RoundedRectangle(cornerRadius: NyxCornerRadius.md))
    }
}
```

### 3. Reduced Motion

Respect the user's reduced motion preference with `@Environment(\.accessibilityReduceMotion)`:

```swift
struct SomeAnimatedView: View {
    @Environment(\.accessibilityReduceMotion) private var reduceMotion

    var body: some View {
        content
            .animation(reduceMotion ? .linear(duration: 0.1) : NyxAnimation.spring, value: isExpanded)
    }
}
```

Extend `NyxAnimation` to provide accessible variants automatically:

```swift
extension NyxAnimation {
    /// Returns a reduced-motion-safe animation.
    /// When Reduce Motion is enabled, returns a near-instant linear transition.
    public static func adaptive(
        _ animation: Animation = .spring(response: 0.35, dampingFraction: 0.8),
        reduceMotion: Bool
    ) -> Animation {
        reduceMotion ? .linear(duration: 0.1) : animation
    }
}

// Usage
.animation(NyxAnimation.adaptive(reduceMotion: reduceMotion), value: trigger)
```

Disable the breathing glow animation when reduced motion is on:

```swift
struct NyxBreathingGlow: ViewModifier {
    let color: Color
    @Environment(\.accessibilityReduceMotion) private var reduceMotion
    @State private var ringScale: CGFloat = 1.0

    func body(content: Content) -> some View {
        content.overlay(
            Circle()
                .stroke(color.opacity(0.4), lineWidth: 2)
                .scaleEffect(ringScale)
                .blur(radius: 3)
        )
        .onAppear {
            guard !reduceMotion else { return }
            withAnimation(.easeInOut(duration: 3).repeatForever(autoreverses: true)) {
                ringScale = 1.15
            }
        }
    }
}
```

### 4. Color Contrast and Differentiate Without Color

WCAG 2.1 AA requires a contrast ratio of 4.5:1 for normal text and 3:1 for large text (18pt+ or 14pt bold). Audit the design tokens from [[04-design-system-as-core-module]]:

| Token | On Surface | Contrast Ratio | WCAG AA |
|-------|-----------|---------------|---------|
| `textPrimary` (white 0.95) | `surfacePrimary` (#000) | 18.1:1 | Pass |
| `textSecondary` (white 0.6) | `surfacePrimary` (#000) | 10.4:1 | Pass |
| `textTertiary` (white 0.4) | `surfacePrimary` (#000) | 5.3:1 | Pass |
| `textTertiary` (white 0.4) | `surfaceElevated` (0.10) | 4.1:1 | Fail for normal text |
| `signalPrimary` (green) | `surfacePrimary` (#000) | 12.8:1 | Pass |

When users enable "Differentiate Without Color", supplement color-only indicators with shapes or icons:

```swift
struct StatusIndicator: View {
    let status: MessageStatus
    @Environment(\.accessibilityDifferentiateWithoutColor) private var differentiateWithoutColor

    var body: some View {
        HStack(spacing: NyxSpacing.xs) {
            Circle()
                .fill(status.color)
                .frame(width: 8, height: 8)

            if differentiateWithoutColor {
                Image(systemName: status.iconName)
                    .font(.caption2)
                    .foregroundColor(status.color)
            }
        }
    }
}

extension MessageStatus {
    var color: Color {
        switch self {
        case .sent: return NyxColors.signalPrimary
        case .failed: return NyxColors.signalWarm
        case .pending: return NyxColors.textSecondary
        }
    }

    var iconName: String {
        switch self {
        case .sent: return "checkmark.circle.fill"
        case .failed: return "exclamationmark.triangle.fill"
        case .pending: return "clock.fill"
        }
    }
}
```

### 5. Accessibility Identifiers

Use the convention `"module.screen.element"` for all accessibility identifiers to enable reliable UI testing:

```swift
// In ChatFeature
sendButton
    .accessibilityIdentifier("chat.input.sendButton")

messageList
    .accessibilityIdentifier("chat.conversation.messageList")

// In InsightsFeature
weeklyCard
    .accessibilityIdentifier("insights.dashboard.weeklyCard")
```

Integrate with XCUITest for automated element discovery (see [[18-ui-testing-regression-and-smoke]]):

```swift
func testChatScreenAccessibilityElements() {
    let app = XCUIApplication()
    app.launch()

    // Navigate to chat tab
    app.buttons["chat.tabBar.chatTab"].tap()

    // Verify critical elements exist and are accessible
    XCTAssertTrue(app.textFields["chat.input.textField"].exists)
    XCTAssertTrue(app.buttons["chat.input.sendButton"].exists)
    XCTAssertTrue(app.scrollViews["chat.conversation.messageList"].exists)

    // Verify send button has an accessibility label (not just identifier)
    let sendButton = app.buttons["chat.input.sendButton"]
    XCTAssertFalse(sendButton.label.isEmpty, "Send button must have an accessibility label")
}
```

### 6. Accessibility Audit in Tests

Use Xcode's `performAccessibilityAudit()` API to run automated WCAG checks in XCUITests:

```swift
func testChatScreenPassesAccessibilityAudit() throws {
    let app = XCUIApplication()
    app.launch()

    // Navigate to the screen under test
    app.buttons["chat.tabBar.chatTab"].tap()

    // Run the full accessibility audit — fails the test if any issues are found
    try app.performAccessibilityAudit()
}

func testInsightsScreenPassesAccessibilityAudit() throws {
    let app = XCUIApplication()
    app.launch()

    app.buttons["chat.tabBar.insightsTab"].tap()

    // Audit with specific categories if needed
    try app.performAccessibilityAudit(for: [
        .dynamicType,
        .contrast,
        .elementDetection,
        .hitRegion
    ])
}

func testAllTabsPassAccessibilityAudit() throws {
    let app = XCUIApplication()
    app.launch()

    let tabs = ["insightsTab", "chatTab", "settingsTab"]
    for tab in tabs {
        app.buttons["chat.tabBar.\(tab)"].tap()
        try app.performAccessibilityAudit()
    }
}
```

Run these in every CI test pass to catch regressions early. See [[18-ui-testing-regression-and-smoke]] for CI integration details.

## Edge Cases

- **Custom shapes (WGlyphView) need `.accessibilityLabel`** — VoiceOver cannot describe custom `Shape` or `Path` content. The center tab glyph must have an explicit label: `.accessibilityLabel("Wythnos assistant")`.
- **ScrollView with `.accessibilityScrollAction`** — For custom scroll behavior (e.g., paginated insights carousel), implement `.accessibilityScrollAction` so VoiceOver's three-finger swipe triggers the correct scroll direction and page snapping.
- **Modal presentations need `.accessibilityAddTraits(.isModal)`** — Without this trait, VoiceOver can read content behind the modal sheet, confusing users. Apply it to the root view of every modal presentation.
- **`@ScaledMetric` clamping** — Use `.dynamicTypeSize(.large ... .accessibility3)` on layouts that break at extreme sizes (e.g., tab bar, compact toolbars). This prevents the view from scaling below Large or above Accessibility 3.
- **RTL + VoiceOver** — VoiceOver traversal order mirrors automatically for RTL languages, but custom `accessibilitySortPriority` values are absolute. If you set explicit sort priorities, verify traversal order in both LTR and RTL by testing with Arabic or Hebrew locales (see [[16-localization-and-multi-language-patterns]]).
- **`textTertiary` on `surfaceElevated`** — This combination fails WCAG AA for normal text (4.1:1 < 4.5:1 required). Use `textSecondary` or increase the opacity of `textTertiary` to 0.5 when placed on elevated surfaces.

## Why This Matters

- **App Store Review may reject apps** with poor accessibility — Apple has increasingly flagged apps that lack VoiceOver support or fail basic Dynamic Type compliance.
- **15-20% of users rely on some accessibility feature** — this includes Dynamic Type (the most common), bold text, reduce motion, and VoiceOver.
- **VoiceOver traversal in custom tab bars** (vs system `TabView`) must be manually configured — the system `TabView` handles traits, labels, and traversal for free; the custom implementation from [[13-swiftui-custom-tab-bar-and-navigation]] does not.
- **Dynamic Type compliance is required** for government and enterprise distribution — many enterprise procurement contracts mandate WCAG 2.1 AA compliance.
- **Automated accessibility audits in CI** catch regressions before they reach users — a missing label on an icon button is invisible in manual testing but caught immediately by `performAccessibilityAudit()`.

## Anti-Patterns

- Don't use `.accessibilityHidden(true)` to "fix" VoiceOver issues — investigate the traversal order with Accessibility Inspector instead. Hiding elements removes them from VoiceOver entirely, making the app less usable.
- Don't hardcode font sizes without `@ScaledMetric` — use the design system tokens from [[04-design-system-as-core-module]] that already support scaling. A `Font.system(size: 15)` that never scales violates Dynamic Type compliance.
- Don't skip `.accessibilityLabel` on icon-only buttons — a button with only an SF Symbol and no label is announced as "button" by VoiceOver, giving the user no information about its purpose.
- Don't test only with VoiceOver off — run `performAccessibilityAudit()` in every CI test pass ([[18-ui-testing-regression-and-smoke]]). Manual VoiceOver testing is valuable but insufficient for catching regressions.
- Don't ignore `.accessibilityDifferentiateWithoutColor` — red/green status indicators are indistinguishable for color-blind users. Always provide a shape or icon fallback when this environment value is true.
