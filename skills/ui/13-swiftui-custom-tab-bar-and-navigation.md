---
title: "Custom SwiftUI Tab Bar with Design System Integration"
description: "Replacing TabView with a custom tab bar using safeAreaInset, re-tap scroll-to-top, spring-physics ButtonStyle, breathing glow animations, keyboard avoidance, and splash screen transitions."
---

# Custom SwiftUI Tab Bar with Design System Integration

## Context
The default `TabView` in SwiftUI is limited in customization — you can't add custom glyph animations, breathing glow rings, or spring-physics press feedback. For a premium dark-mode app, you need a fully custom tab bar that integrates with your [[04-design-system-as-core-module|design system tokens]].

## Pattern

### Custom Tab Bar Architecture

```swift
struct MainTabView: View {
    @State private var selectedTab: Tab = .chat
    @State private var tabReTapTrigger: UUID? = nil

    enum Tab: Int, CaseIterable, Identifiable {
        case insights = 0
        case chat = 1      // Center position
        case settings = 2

        var id: Int { rawValue }
        var icon: String { /* SF Symbol names */ }
        var label: String { /* Localized strings — see [[16-localization-and-multi-language-patterns]] */ }
        var analyticsName: String { /* screen identifiers */ }
    }

    var body: some View {
        tabContent
            .safeAreaInset(edge: .bottom) {
                NyxCustomTabBar(
                    selectedTab: $selectedTab,
                    onTabReTap: { tab in
                        NyxHaptics.scrollToBottom()
                        tabReTapTrigger = UUID()
                    }
                )
                .ignoresSafeArea(.keyboard, edges: .bottom)
            }
    }

    @ViewBuilder
    private var tabContent: some View {
        switch selectedTab {
        case .insights:
            ServiceProvider.shared.resolve(\.insightsUI).makeInsightsView()
                .id("insights_\(tabReTapTrigger?.uuidString ?? "")")
        case .chat:
            ServiceProvider.shared.resolve(\.chatUI).makeChatView()
                .id("chat_\(tabReTapTrigger?.uuidString ?? "")")
        case .settings:
            ServiceProvider.shared.resolve(\.settingsUI).makeSettingsView()
                .id("settings_\(tabReTapTrigger?.uuidString ?? "")")
        }
    }
}
```

### Key Techniques

**1. `safeAreaInset(edge: .bottom)` instead of `overlay` or `ZStack`:**
```swift
tabContent
    .safeAreaInset(edge: .bottom) {
        NyxCustomTabBar(...)
            .ignoresSafeArea(.keyboard, edges: .bottom)
    }
```
This correctly pushes content up to avoid the tab bar, and `.ignoresSafeArea(.keyboard)` prevents the tab bar from jumping when the keyboard appears.

**2. Tab re-tap triggers scroll-to-top via `.id()` change:**
```swift
.id("chat_\(tabReTapTrigger?.uuidString ?? "")")
```
Changing the view's `.id()` forces SwiftUI to recreate it, which resets scroll position. The `UUID()` triggers are fired when the user taps the already-selected tab.

**3. Custom pressable button style with spring physics:**
```swift
struct NyxPressableStyle: ButtonStyle {
    let pressedScale: CGFloat
    func makeBody(configuration: Configuration) -> some View {
        configuration.label
            .scaleEffect(configuration.isPressed ? pressedScale : 1.0)
            .opacity(configuration.isPressed ? 0.8 : 1.0)
            .animation(.spring(response: 0.25, dampingFraction: 0.6), value: configuration.isPressed)
    }
}
```

**4. Custom glyph with breathing animation:**
```swift
struct WGlyphView: View {
    let isActive: Bool
    @State private var ringScale: CGFloat = 1.0
    
    var body: some View {
        ZStack {
            Circle()
                .stroke(NyxColors.signalPrimary.opacity(ringOpacity), lineWidth: 2)
                .scaleEffect(ringScale)
                .blur(radius: 3)
            
            WGlyphShape()
                .fill(isActive ? NyxColors.signalPrimary : NyxColors.textSecondary)
                .shadow(color: isActive ? NyxColors.signalPrimary.opacity(0.5) : .clear, radius: 8)
        }
        .onChange(of: isActive) { _, active in
            if active {
                withAnimation(.easeInOut(duration: 3).repeatForever(autoreverses: true)) {
                    ringScale = 1.15
                }
            }
        }
    }
}
```

**5. Analytics tracking on tab switch:**
```swift
Button {
    if isSelected {
        NyxHaptics.scrollToBottom()
        onTabReTap?(tab)
    } else {
        NyxHaptics.tabSwitch()
        withAnimation(NyxAnimation.spring) { selectedTab = tab }
        analytics.track(.screenOpen(tab.analyticsName))
    }
}
```

### Splash Screen → Content Transition

```swift
var body: some Scene {
    WindowGroup {
        ZStack {
            if !splashScreenComplete {
                SplashScreenView {
                    withAnimation(.easeOut(duration: 0.6)) { splashScreenComplete = true }
                }
                .zIndex(1)
            }
            
            if splashScreenComplete {
                contentView
                    .modifier(FallingFromAbove3DAnimation())
                    .zIndex(0)
            }
        }
        .preferredColorScheme(.dark)
        .environment(\.colorScheme, .dark)
    }
}

struct FallingFromAbove3DAnimation: ViewModifier {
    @State private var opacity: Double = 0
    func body(content: Content) -> some View {
        content.opacity(opacity)
            .onAppear { withAnimation(.easeOut(duration: 0.4)) { opacity = 1 } }
    }
}
```

### Onboarding → Main App Transition

```swift
@ViewBuilder
private var contentView: some View {
    if hasCompletedOnboarding {
        MainTabView().environmentObject(deepLinkRouter)
    } else {
        ServiceProvider.shared.resolve(\.onboardingUI)
            .makeOnboardingView {
                withAnimation(.easeInOut(duration: 0.5)) { hasCompletedOnboarding = true }
            }
    }
}
```

## Why This Matters

- **`safeAreaInset`** is the correct API for custom tab bars — it respects keyboard avoidance and safe areas automatically (but beware of [[17-safe-area-inset-stacking-and-bottom-pinned-views|stacking issues]] when nesting insets)
- **Re-tap detection** (tap already-selected tab) enables scroll-to-top via view identity change
- **Haptic feedback on every tab switch** creates a premium tactile feel
- **No `TabView`** means full control over animations, glyphs, and layout
- **`@ViewBuilder` switch** avoids `AnyView` type erasure — better SwiftUI diffing performance

## Anti-Patterns

- Don't use `overlay` for tab bars — it doesn't push content up, causing overlap
- Don't use padding for keyboard avoidance — `.ignoresSafeArea(.keyboard)` on the tab bar is the correct approach (see also [[17-safe-area-inset-stacking-and-bottom-pinned-views]])
- Don't animate tab content transitions — it causes input field shifting during keyboard interactions
- Don't use `AnyView` for tab content — use `@ViewBuilder` with `switch`
- Don't resolve services inside the View body — resolve once in the factory and inject
