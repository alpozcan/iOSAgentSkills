# Typed Error System with Recovery Actions

## Context
In a SwiftUI app with multiple failure modes (calendar access denied, AI unavailable, sync failed, subscription errors), you need a structured error system where each error type maps to a specific title, message, icon, and recovery action — all rendered through a reusable design system component.

## Pattern

### Typed Error Enum with Semantic Properties

```swift
public enum WythnosError: Error, Sendable {
    case calendarAccessDenied
    case aiUnavailable
    case syncFailed(String)
    case subscriptionError(String)
    case networkError
    case unknown

    public var title: String {
        switch self {
        case .calendarAccessDenied: return "Calendar Access Required"
        case .aiUnavailable: return "AI Temporarily Unavailable"
        case .syncFailed: return "Sync Failed"
        case .subscriptionError: return "Subscription Issue"
        case .networkError: return "Connection Error"
        case .unknown: return "Something Went Wrong"
        }
    }

    public var message: String {
        switch self {
        case .calendarAccessDenied:
            return "Wythnos needs calendar access to provide insights. Please enable it in Settings."
        case .aiUnavailable:
            return "The AI is taking longer than expected. Your calendar data is still available."
        case .syncFailed(let reason):
            return "Couldn't sync your calendar: \(reason)"
        // ...
        }
    }

    public var icon: String {
        switch self {
        case .calendarAccessDenied: return "calendar.badge.exclamationmark"
        case .aiUnavailable: return "brain.head.profile"
        case .syncFailed: return "arrow.clockwise.circle"
        // ...
        }
    }

    public var recoveryAction: RecoveryAction? {
        switch self {
        case .calendarAccessDenied: return .openSettings
        case .aiUnavailable: return .retry
        case .syncFailed: return .retry
        case .subscriptionError: return .contactSupport
        case .networkError: return .retry
        case .unknown: return .dismiss
        }
    }
}

public enum RecoveryAction: Sendable {
    case openSettings
    case retry
    case contactSupport
    case dismiss
    
    public var label: String {
        switch self {
        case .openSettings: return "Open Settings"
        case .retry: return "Try Again"
        case .contactSupport: return "Contact Support"
        case .dismiss: return "Dismiss"
        }
    }
}
```

### Reusable Error Card Component

```swift
public struct NyxErrorCard: View {
    let error: WythnosError
    let onRecovery: ((RecoveryAction) -> Void)?

    public var body: some View {
        VStack(spacing: NyxSpacing.md) {
            // Icon
            Image(systemName: error.icon)
                .font(.system(size: 32))
                .foregroundColor(NyxColors.signalWarm)

            // Title & Message
            VStack(spacing: NyxSpacing.xs) {
                Text(error.title)
                    .font(NyxTypography.headline)
                    .foregroundColor(NyxColors.textPrimary)
                Text(error.message)
                    .font(NyxTypography.body)
                    .foregroundColor(NyxColors.textSecondary)
                    .multilineTextAlignment(.center)
            }

            // Recovery button
            if let action = error.recoveryAction {
                Button(action.label) { onRecovery?(action) }
                    .font(NyxTypography.headline)
                    .foregroundColor(.black)
                    .padding(.horizontal, NyxSpacing.lg)
                    .padding(.vertical, NyxSpacing.sm)
                    .background(NyxColors.signalPrimary)
                    .clipShape(RoundedRectangle(cornerRadius: NyxCornerRadius.sm))
            }
        }
        .padding(NyxSpacing.lg)
        .background(NyxColors.surface)
        .overlay(
            Rectangle().fill(NyxColors.signalWarm.opacity(0.5)).frame(width: 2),
            alignment: .leading
        )
        .clipShape(RoundedRectangle(cornerRadius: NyxCornerRadius.md))
    }
}
```

### Safety Classification System (AI-Specific Errors)

```swift
public enum SafetyClassification: Sendable, Equatable {
    case safe
    case offTopic
    case harmful(RefusalType)
    case emergency       // Mental health crisis → provide resources
}

public enum RefusalType: String, Sendable, CaseIterable {
    case violence, illegal, hateSpeech, selfHarm, explicit, misinformation, offTopic
}

// Localized refusal responses (English + Turkish)
public struct RefusalResponseGenerator: Sendable {
    private let responses: [String: [RefusalType: String]] = [
        "en": [
            .violence: "I can't help with requests involving violence. How can I help with your schedule?",
            .selfHarm: "I'm not equipped to help with this, but support is available. Contact your local crisis line.",
            .offTopic: "I'm Wythnos, your calendar intelligence assistant. What would you like to know about your calendar?",
        ],
        "tr": [
            .offTopic: "Ben Wythnos, takvim zekası asistanınızım. Takviminizle ilgili ne bilmek istersiniz?",
        ]
    ]
    
    public func generate(for type: RefusalType, locale: Locale) -> String {
        let lang = locale.language.languageCode?.identifier ?? "en"
        return responses[lang]?[type] ?? responses["en"]?[type] ?? defaultResponse
    }
}
```

### Emergency Response Handling

```swift
case .emergency:
    let resource = EmergencyResources.resource(for: .current)
    return "I'm not equipped to help with this, but please know that support is available. Contact \(resource). You matter."
```

## Why This Matters

- **Every error has exactly one recovery path** — no ambiguity for the user
- **Typed enum prevents stringly-typed errors** — compiler enforces handling of every case
- **The error card is a design system component** — consistent across all features
- **Safety classification** integrates error handling with AI safety at the architecture level
- **Localized refusals** respect the user's language preference
- **`Sendable` conformance** ensures errors can safely cross actor boundaries

## Anti-Patterns

- Don't catch errors with generic `catch` blocks — always pattern match on specific error types
- Don't show raw error messages to users — map every error to a human-readable title + message
- Don't hardcode error strings in Feature modules — define them in the error enum's computed properties
- Don't show alerts for errors — use inline error cards that don't interrupt the user's flow
- Don't forget to handle `.emergency` classification — always provide crisis resources for self-harm queries
