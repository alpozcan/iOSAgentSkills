---
title: "Design System as a Core Framework Module"
description: "Four-layer design system as a Core module: tokens (colors, typography, spacing), reusable components, animation primitives, and centralized haptic feedback. Dark-mode-only, 4pt grid, OLED-optimized."
---

# Design System as a Core Framework Module

## Context
Every Feature module needs consistent colors, typography, spacing, animations, and reusable components. Without a centralized design system, visual inconsistency creeps in and designers lose trust. The design system must be a compile-time dependency, not a runtime theme.

## Pattern

Build the design system as a **Core framework module** (see [[01-tuist-modular-architecture]]) with four layers: tokens, components, animations, and haptics.

### Layer 1: Design Tokens (Static Enums, No Instances)

```swift
// NyxDesignTokens.swift

public enum NyxColors {
    // Surface hierarchy (z-axis layering)
    public static let surfacePrimary = Color(red: 0, green: 0, blue: 0)           // L0: screen bg
    public static let surfaceSecondary = Color(red: 0.07, green: 0.07, blue: 0.09) // L1: tab bars
    public static let surfaceElevated = Color(red: 0.10, green: 0.10, blue: 0.13)  // L2: cards, sheets
    
    // Signal colors (accent palette)
    public static let signalPrimary = Color(red: 0.0, green: 1.0, blue: 0.533)   // AI signature green
    public static let signalWarm = Color(red: 1.0, green: 0.42, blue: 0.208)      // Warnings, errors
    public static let signalTertiary = Color(red: 0.545, green: 0.361, blue: 0.965) // Evolution/intelligence
    
    // Text hierarchy
    public static let textPrimary = Color.white.opacity(0.95)
    public static let textSecondary = Color.white.opacity(0.6)
    public static let textTertiary = Color.white.opacity(0.4) // WCAG AA threshold
    
    // Semantic borders
    public static let border = Color(red: 0.102, green: 0.102, blue: 0.141)
}

public enum NyxTypography {
    public static let display = Font.system(size: 34, weight: .bold, design: .default)
    public static let title = Font.system(size: 22, weight: .semibold, design: .default)
    public static let headline = Font.system(size: 17, weight: .medium, design: .default)
    public static let body = Font.system(size: 15, weight: .regular, design: .default)
    public static let caption = Font.system(size: 12, weight: .regular, design: .default)
    public static let data = Font.system(size: 14, weight: .medium, design: .monospaced) // Terminal feel
}

public enum NyxSpacing {
    public static let xs: CGFloat = 4
    public static let sm: CGFloat = 8
    public static let md: CGFloat = 16
    public static let lg: CGFloat = 24
    public static let xl: CGFloat = 32
}

public enum NyxCornerRadius {
    public static let sm: CGFloat = 8
    public static let md: CGFloat = 14
    public static let lg: CGFloat = 20
    public static let pill: CGFloat = 999
}

public enum NyxAnimation {
    public static let spring = Animation.spring(response: 0.35, dampingFraction: 0.8)
    public static let typedCharacter: TimeInterval = 0.03
    public static let staggerDelay: TimeInterval = 0.08
}
```

### Layer 2: Reusable Components

```swift
// NyxCard — consistent elevated container
public struct NyxCard<Content: View>: View {
    let content: Content
    public init(@ViewBuilder content: () -> Content) { self.content = content() }
    
    public var body: some View {
        content
            .padding(NyxSpacing.md)
            .background(NyxColors.surface)
            .clipShape(RoundedRectangle(cornerRadius: NyxCornerRadius.md))
            .overlay(
                RoundedRectangle(cornerRadius: NyxCornerRadius.md)
                    .stroke(NyxColors.border.opacity(0.5), lineWidth: 0.5)
            )
    }
}

// NyxErrorCard — typed error states with recovery actions (see [[14-error-handling-and-typed-error-system]])
public struct NyxErrorCard: View {
    let error: WythnosError
    let onRecovery: ((RecoveryAction) -> Void)?
    // Each error type maps to: title, message, icon, recovery action
}

// NyxFeedbackToggle — thumbs up/down with tri-state (none/positive/negative)
public struct NyxFeedbackToggle: View {
    public enum State { case none, positive, negative }
    let state: State
    let onPositive: () -> Void
    let onNegative: () -> Void
}

// AutoSizingTextEditor — grows from 1 to 4 lines with spring animation
public struct AutoSizingTextEditor: View {
    @Binding var text: String
    // min 40px, max 100px, springs to new height
}

// FlowLayout — custom Layout protocol for chip/tag wrapping
public struct FlowLayout: Layout {
    var spacing: CGFloat
    // Wraps children horizontally, breaking to next line when exceeding width
}
```

### Layer 3: Animation Components

```swift
// Typed text (character-by-character with cursor)
public struct NyxTypedText: View {
    let text: String
    // Types each character with configurable speed, shows blinking cursor
}

// Count-up number animation
public struct NyxCountUpText: View {
    let value: Double
    let formatter: (Double) -> String
    // Animates from 0 to target value over duration
}

// View modifiers
public extension View {
    func nyxSlideUp(delay: TimeInterval = 0) -> some View {
        modifier(NyxSlideUp(delay: delay))
    }
    
    func nyxBreathingGlow(color: Color = NyxColors.glowSignal) -> some View {
        modifier(NyxBreathingGlow(color: color))
    }
}
```

### Layer 4: Haptic Feedback System

```swift
public enum NyxHaptics {
    // Pre-initialized generators (eliminate latency)
    private static let selectionGenerator = UISelectionFeedbackGenerator()
    private static let mediumImpactGenerator = UIImpactFeedbackGenerator(style: .medium)
    private static let notificationGenerator = UINotificationFeedbackGenerator()
    
    public static func prepare() { /* prepare all generators */ }
    
    // Semantic haptic events
    public static func tabSwitch() { selectionGenerator.selectionChanged() }
    public static func sendMessage() { mediumImpactGenerator.impactOccurred(intensity: 0.8) }
    public static func aiResponseComplete() { notificationGenerator.notificationOccurred(.success) }
    public static func error() { notificationGenerator.notificationOccurred(.error) }
    public static func onboardingComplete() {
        heavyImpactGenerator.impactOccurred(intensity: 1.0)
        // Double-tap feel
        DispatchQueue.main.asyncAfter(deadline: .now() + 0.1) {
            mediumImpactGenerator.impactOccurred(intensity: 0.6)
        }
    }
}

// SwiftUI integration
public extension View {
    func nyxPrepareHaptics() -> some View {
        self.onAppear { NyxHaptics.prepare() }
    }
}
```

### Custom ButtonStyle for Consistent Press Feedback

```swift
public struct NyxPressableStyle: ButtonStyle {
    let pressedScale: CGFloat
    let pressedOpacity: Double
    
    public func makeBody(configuration: Configuration) -> some View {
        configuration.label
            .scaleEffect(configuration.isPressed ? pressedScale : 1.0)
            .opacity(configuration.isPressed ? pressedOpacity : 1.0)
            .animation(.spring(response: 0.25, dampingFraction: 0.6), value: configuration.isPressed)
    }
}
```

## Design Principles Enforced by Code

1. **Dark mode only** — no light mode variants, `preferredColorScheme(.dark)` set at app root
2. **4pt grid** — all spacing tokens are multiples of 4
3. **44pt+ touch targets** — enforced by minimum frame sizes
4. **SF Symbols only** — no custom icon assets
5. **OLED optimized** — true black (#000000) as primary background
6. **Surface hierarchy** — 3-level z-axis: Primary (black) → Secondary (0.07) → Elevated (0.10)

## Why This Matters

- **Compile-time guarantee** — every Feature module imports `DesignSystem` and uses the same tokens
- **Enum-based tokens** — no instances, no allocation, no runtime lookups
- **Haptic consistency** — centralized enum prevents scattered `UIImpactFeedbackGenerator` allocations; these haptics are used throughout the [[13-swiftui-custom-tab-bar-and-navigation|custom tab bar]]
- **Testable components** — components like `NyxErrorCard` can be tested for correct error-to-action mapping (see [[14-error-handling-and-typed-error-system]])

## Anti-Patterns

- Don't hardcode colors as `.init(red:green:blue:)` in Feature modules — always use tokens
- Don't create new `UIImpactFeedbackGenerator()` instances locally — use `NyxHaptics`
- Don't add light mode overrides — the design system is dark-only by specification
- Don't use `AnyView` wrapping for design components — keep concrete types for SwiftUI diffing
