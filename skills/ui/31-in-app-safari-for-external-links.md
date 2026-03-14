---
title: "In-App Safari for External Links with SFSafariViewController"
description: "Always open external URLs inside the app using SFSafariViewController instead of ejecting to Safari. Covers a reusable SwiftUI wrapper, presentation patterns, and customization."
---

# In-App Safari for External Links

## Context
When an app opens a URL with `Link(destination:)` or `UIApplication.shared.open(_:)`, the user is ejected to Safari — losing context, breaking immersion, and requiring a manual swipe back. `SFSafariViewController` keeps the browsing experience inside the app while still offering Safari's full rendering, autofill, and Reader mode. **Default rule: every external URL in the app should open via SFSafariViewController, never via `Link` or `openURL`.**

## Pattern

### 1. Reusable `SafariView` wrapper

```swift
import SwiftUI
import SafariServices

struct SafariView: UIViewControllerRepresentable {
    let url: URL

    func makeUIViewController(context: Context) -> SFSafariViewController {
        let config = SFSafariViewController.Configuration()
        config.entersReaderIfAvailable = false
        config.barCollapsingEnabled = true
        let vc = SFSafariViewController(url: url, configuration: config)
        // Tint the toolbar to match your design system accent color
        vc.preferredControlTintColor = UIColor(YourDesignSystem.accentColor)
        return vc
    }

    func updateUIViewController(_ uiViewController: SFSafariViewController, context: Context) {}
}
```

### 2. Make `URL` work with `item:`-based presentation

SwiftUI's `.sheet(item:)` and `.fullScreenCover(item:)` require `Identifiable`. Add a one-time retroactive conformance:

```swift
extension URL: @retroactive Identifiable {
    public var id: String { absoluteString }
}
```

> Place this in a shared Helpers or Extensions file so it's available app-wide.

### 3. Presenting from any view

```swift
struct SomeView: View {
    @State private var safariURL: URL?

    var body: some View {
        Button("Open Link") {
            safariURL = URL(string: "https://example.com")
        }
        .fullScreenCover(item: $safariURL) { url in
            SafariView(url: url)
                .ignoresSafeArea()
        }
    }
}
```

**Prefer `.fullScreenCover`** over `.sheet` — it gives the Safari chrome more room and avoids drag-to-dismiss conflicts with the web content scroll view.

### 4. Design system integration

Customize the toolbar tint to match your app's design tokens:

```swift
vc.preferredControlTintColor = UIColor(MEColors.bronze)      // accent color
vc.preferredBarTintColor = UIColor(MEColors.backgroundPrimary) // bar background
```

`dismissButtonStyle` can also be set if you prefer `.close` vs `.done`:

```swift
vc.dismissButtonStyle = .close
```

## When NOT to use SFSafariViewController

- **Authentication flows** that need redirect interception — use `ASWebAuthenticationSession` instead (see [[07-storekit2-intelligence-based-trial]] for related patterns)
- **Rendering custom HTML** you control — use `WKWebView`
- **Deep links back into the app** — use Universal Links or `ASWebAuthenticationSession`

## Anti-patterns

| Don't | Do |
|---|---|
| `Link(destination: url)` | `Button { safariURL = url }` + `.fullScreenCover` with `SafariView` |
| `UIApplication.shared.open(url)` | Present `SFSafariViewController` |
| `WKWebView` for third-party pages | `SFSafariViewController` — inherits cookies, autofill, content blockers |
| `.sheet` presentation | `.fullScreenCover` — avoids scroll/drag conflicts |

## Checklist

- [ ] `SafariView` wrapper exists in a shared location (e.g., `Views/Common/` or a Core module)
- [ ] `URL: Identifiable` conformance is in a single shared file
- [ ] Every `Link(destination:)` in the codebase has been replaced with a `Button` + `safariURL` state + `.fullScreenCover`
- [ ] Toolbar tint matches the design system accent color
- [ ] No external URLs open via `UIApplication.shared.open`

## Related Skills

- [[04-design-system-as-core-module]] — design tokens for bar tint customization
- [[13-swiftui-custom-tab-bar-and-navigation]] — navigation context that SafariView lives within
- [[21-accessibility-voiceover-dynamic-type-patterns]] — SFSafariViewController automatically supports VoiceOver and Dynamic Type
