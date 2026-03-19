---
title: "Minimal Onboarding Flow"
description: "Reusable 2-4 page onboarding with paged TabView, hero illustrations, permission requests, and AppStorage completion tracking. Designed for minimal apps that need a light-touch first launch."
---

# Minimal Onboarding Flow

## Context

Minimal apps need a minimal onboarding — 2-4 pages max that communicate the core value, request necessary permissions (camera, notifications), and get out of the way. This pattern uses a paged `TabView` with `@AppStorage` to track completion so it only shows once. No sign-up, no accounts.

## Pattern

### Onboarding Data Model

```swift
struct OnboardingPage: Identifiable {
    let id = UUID()
    let systemImage: String      // SF Symbol name
    let title: String
    let subtitle: String
    let accentColor: Color
}
```

### Onboarding View

```swift
import SwiftUI

struct OnboardingView: View {
    let pages: [OnboardingPage]
    let onComplete: () -> Void

    @State private var currentPage = 0

    var body: some View {
        VStack(spacing: 0) {
            TabView(selection: $currentPage) {
                ForEach(Array(pages.enumerated()), id: \.element.id) { index, page in
                    OnboardingPageView(page: page)
                        .tag(index)
                }
            }
            .tabViewStyle(.page(indexDisplayMode: .never))
            .animation(.easeInOut(duration: 0.3), value: currentPage)

            // Page indicator + button
            VStack(spacing: 24) {
                // Dots
                HStack(spacing: 8) {
                    ForEach(0..<pages.count, id: \.self) { index in
                        Circle()
                            .fill(index == currentPage ? Color.primary : Color.primary.opacity(0.2))
                            .frame(width: 8, height: 8)
                            .animation(.easeInOut(duration: 0.2), value: currentPage)
                    }
                }

                // Action button
                Button {
                    if currentPage < pages.count - 1 {
                        withAnimation { currentPage += 1 }
                    } else {
                        onComplete()
                    }
                } label: {
                    Text(currentPage < pages.count - 1 ? "Next" : "Get Started")
                        .font(.headline)
                        .frame(maxWidth: .infinity)
                        .padding(.vertical, 16)
                        .background(Color.primary)
                        .foregroundStyle(Color(.systemBackground))
                        .clipShape(RoundedRectangle(cornerRadius: 14))
                }
                .padding(.horizontal, 24)

                // Skip (not on last page)
                if currentPage < pages.count - 1 {
                    Button("Skip") {
                        onComplete()
                    }
                    .font(.subheadline)
                    .foregroundStyle(.secondary)
                }
            }
            .padding(.bottom, 48)
        }
    }
}

struct OnboardingPageView: View {
    let page: OnboardingPage

    var body: some View {
        VStack(spacing: 20) {
            Spacer()

            Image(systemName: page.systemImage)
                .font(.system(size: 72, weight: .thin))
                .foregroundStyle(page.accentColor)

            Text(page.title)
                .font(.title)
                .fontWeight(.semibold)
                .multilineTextAlignment(.center)

            Text(page.subtitle)
                .font(.body)
                .foregroundStyle(.secondary)
                .multilineTextAlignment(.center)
                .padding(.horizontal, 40)

            Spacer()
            Spacer()
        }
    }
}
```

### Root App Integration

```swift
@main
struct MyApp: App {
    @AppStorage("hasCompletedOnboarding") private var hasCompletedOnboarding = false

    var body: some Scene {
        WindowGroup {
            if hasCompletedOnboarding {
                ContentView()
            } else {
                OnboardingView(pages: onboardingPages) {
                    withAnimation {
                        hasCompletedOnboarding = true
                    }
                }
            }
        }
    }

    private var onboardingPages: [OnboardingPage] {
        [
            OnboardingPage(
                systemImage: "text.quote",
                title: "Capture Quotes",
                subtitle: "Point your camera at any book page or type quotes manually.",
                accentColor: .primary
            ),
            OnboardingPage(
                systemImage: "rectangle.stack",
                title: "Organize by Book",
                subtitle: "Quotes are automatically grouped by book and author.",
                accentColor: .primary
            ),
            OnboardingPage(
                systemImage: "widget.small",
                title: "Daily Rediscovery",
                subtitle: "Add a widget to see a random quote on your home screen every day.",
                accentColor: .primary
            ),
        ]
    }
}
```

### Permission Request Page (Optional)

For apps needing camera or notification access, add a permission page:

```swift
struct PermissionOnboardingPage: View {
    let title: String
    let subtitle: String
    let systemImage: String
    let buttonTitle: String
    let onRequest: () async -> Void

    @State private var granted = false

    var body: some View {
        VStack(spacing: 20) {
            Spacer()

            Image(systemName: systemImage)
                .font(.system(size: 72, weight: .thin))
                .foregroundStyle(granted ? .green : .primary)

            Text(title)
                .font(.title)
                .fontWeight(.semibold)

            Text(subtitle)
                .font(.body)
                .foregroundStyle(.secondary)
                .multilineTextAlignment(.center)
                .padding(.horizontal, 40)

            if !granted {
                Button(buttonTitle) {
                    Task {
                        await onRequest()
                        granted = true
                    }
                }
                .buttonStyle(.borderedProminent)
                .tint(.primary)
                .padding(.top, 8)
            } else {
                Label("Enabled", systemImage: "checkmark.circle.fill")
                    .foregroundStyle(.green)
            }

            Spacer()
            Spacer()
        }
    }
}
```

### Animation Presets (from MiddleEarth pattern)

For apps wanting polished onboarding transitions:

```swift
enum OnboardingAnimation {
    static let quick = Animation.easeInOut(duration: 0.2)
    static let standard = Animation.easeInOut(duration: 0.3)
    static let spring = Animation.spring(response: 0.3, dampingFraction: 0.7, blendDuration: 0)

    /// Respects system reduced motion preference
    static func adaptive(reduceMotion: Bool = false) -> Animation {
        reduceMotion ? .easeInOut(duration: 0.1) : spring
    }
}
```

## Rules

- **2-4 pages max.** More than 4 and users will skip anyway.
- **Show only once.** Use `@AppStorage("hasCompletedOnboarding")` — survives reinstall via iCloud KeyValue sync if needed.
- **Always include Skip.** Forcing users through onboarding creates resentment.
- **No sign-up.** This pattern is for apps without accounts. If auth is needed, that's a separate flow after onboarding.
- **Request permissions in context.** Don't front-load camera/notification permissions — request on the page that explains why.
- **Respect reduced motion.** Use `@Environment(\.accessibilityReduceMotion)` to tone down animations.

## Anti-Patterns

- ❌ DON'T use onboarding as a feature tour — it's a value proposition, not a manual
- ❌ DON'T animate every element — one subtle transition per page is enough
- ❌ DON'T request permissions on the first page — build context first
- ❌ DON'T store completion in UserDefaults without `@AppStorage` — you'll forget to check it somewhere
- ❌ DON'T show onboarding again after app updates — users hate it
