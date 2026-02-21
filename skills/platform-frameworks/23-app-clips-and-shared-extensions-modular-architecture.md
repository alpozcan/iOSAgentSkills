---
title: "App Clips and Shared Extensions in Modular Architecture"
description: "Structuring App Clips, Share Extensions, and Notification Content Extensions as separate Tuist targets with filtered Core module dependencies, App Group data sharing, clip-to-app migration, and binary size optimization under the 15 MB App Clip limit."
---

# App Clips and Shared Extensions in Modular Architecture

## Context
App Clips and extensions (Share, Widget, Notification Content) need to share code with the main app while staying under strict binary size limits (App Clips: 15 MB). In a [[01-tuist-modular-architecture|Tuist modular app]], this means carefully selecting which Core modules to include in each target. Feature modules pull in too many transitive dependencies and must never be linked into clips or extensions. The challenge is maximizing code reuse (especially [[04-design-system-as-core-module|DesignSystem]] and networking layers) without blowing past size budgets or violating extension sandbox restrictions.

## Pattern

### App Clip as Tuist Target

Define the App Clip as a `.appClip` product type with filtered dependencies. Only include Core modules — never Feature modules:

```swift
// In Project.swift
.target(
    name: "MyAppClip",
    destinations: [.iPhone],
    product: .appClip,
    bundleId: "com.app.clip",
    sources: ["Targets/AppClip/Sources/**"],
    dependencies: [
        .target(name: "DesignSystem"),
        .target(name: "NetworkService"),
        // NO Feature modules — keep binary small
    ]
)
```

The App Clip target gets its own `Info.plist` with `NSAppClip` dictionary specifying the invocation URL and experience configuration. Tuist handles embedding the clip inside the main app's `.app` bundle automatically when the clip target is listed as a dependency of the app target.

### App Group Data Sharing

Share `UserDefaults` and files between the app, clip, and extensions via an App Group container. Define the identifier once in a shared Core module to avoid hardcoding across targets:

```swift
// In a shared Core module (e.g., SharedConstants)
public enum AppGroupConfig {
    public static let suiteName = "group.com.app.shared"
}

// Usage in any target (app, clip, extension)
guard let sharedDefaults = UserDefaults(suiteName: AppGroupConfig.suiteName) else {
    // Entitlement missing — fail gracefully
    return
}

guard let sharedContainer = FileManager.default.containerURL(
    forSecurityApplicationGroupIdentifier: AppGroupConfig.suiteName
) else {
    // Container is nil if entitlement is misconfigured
    return
}
```

Both the main app target and the clip/extension targets must have the App Groups entitlement enabled with the same identifier, or the container URL will be `nil` at runtime.

### Clip-to-App Transition

When a user installs the full app after using the App Clip, migrate data from the shared App Group container. The App Clip's local data outside the container is deleted on full app install, so anything worth preserving must be written to the App Group:

```swift
final class ClipDataMigrator {
    private let sharedDefaults: UserDefaults
    private let appDefaults: UserDefaults

    init() {
        self.sharedDefaults = UserDefaults(suiteName: AppGroupConfig.suiteName)!
        self.appDefaults = .standard
    }

    func migrateIfNeeded() {
        guard sharedDefaults.bool(forKey: "clipDidComplete") else { return }
        guard !appDefaults.bool(forKey: "clipDataMigrated") else { return }

        // Migrate user preferences saved during clip session
        if let clipUserName = sharedDefaults.string(forKey: "userName") {
            appDefaults.set(clipUserName, forKey: "userName")
        }

        // Migrate files from shared container
        if let sharedContainer = FileManager.default.containerURL(
            forSecurityApplicationGroupIdentifier: AppGroupConfig.suiteName
        ) {
            let clipCache = sharedContainer.appendingPathComponent("clip_cache")
            // Move files to app's documents directory...
        }

        appDefaults.set(true, forKey: "clipDataMigrated")
    }
}
```

Handle the invocation URL with `NSUserActivity` in the clip's scene delegate:

```swift
func scene(_ scene: UIScene, continue userActivity: NSUserActivity) {
    guard userActivity.activityType == NSUserActivityTypeBrowsingWeb,
          let url = userActivity.webpageURL else { return }
    // Route to the appropriate clip experience based on URL path
    router.handle(invocationURL: url)
}
```

For streamlined authentication in the clip, use `ASAuthorizationController` for Sign in with Apple — it provides a low-friction auth flow without requiring the user to create an account:

```swift
let provider = ASAuthorizationAppleIDProvider()
let request = provider.createRequest()
request.requestedScopes = [.email, .fullName]

let controller = ASAuthorizationController(authorizationRequests: [request])
controller.delegate = self
controller.performRequests()
```

### Share Extension

Define the Share Extension as a `.appExtension` Tuist target. It shares Core modules with the main app and uses a custom URL scheme to deep link back:

```swift
// In Project.swift
.target(
    name: "MyShareExtension",
    destinations: [.iPhone],
    product: .appExtension,
    bundleId: "com.app.share-extension",
    infoPlist: .extendingDefault(with: [
        "NSExtension": [
            "NSExtensionPointIdentifier": "com.apple.share-services",
            "NSExtensionPrincipalClass": "$(PRODUCT_MODULE_NAME).ShareViewController",
        ]
    ]),
    sources: ["Targets/ShareExtension/Sources/**"],
    dependencies: [
        .target(name: "DesignSystem"),
        .target(name: "NetworkService"),
    ]
)
```

Extract shared content from `NSExtensionContext` and handle the three major content types (URL, text, image):

```swift
class ShareViewController: UIViewController {
    override func viewDidLoad() {
        super.viewDidLoad()
        extractSharedContent()
    }

    private func extractSharedContent() {
        guard let items = extensionContext?.inputItems as? [NSExtensionItem] else {
            close()
            return
        }

        for item in items {
            guard let attachments = item.attachments else { continue }
            for provider in attachments {
                if provider.hasItemConformingToTypeIdentifier("public.url") {
                    provider.loadItem(forTypeIdentifier: "public.url") { [weak self] item, _ in
                        guard let url = item as? URL else { return }
                        self?.handleSharedURL(url)
                    }
                } else if provider.hasItemConformingToTypeIdentifier("public.plain-text") {
                    provider.loadItem(forTypeIdentifier: "public.plain-text") { [weak self] item, _ in
                        guard let text = item as? String else { return }
                        self?.handleSharedText(text)
                    }
                } else if provider.hasItemConformingToTypeIdentifier("public.image") {
                    provider.loadItem(forTypeIdentifier: "public.image") { [weak self] item, _ in
                        guard let imageURL = item as? URL else { return }
                        self?.handleSharedImage(imageURL)
                    }
                }
            }
        }
    }

    private func openMainApp(with path: String) {
        // Deep link back to main app via custom URL scheme
        let url = URL(string: "myapp://share?path=\(path)")!
        var responder: UIResponder? = self
        while let nextResponder = responder?.next {
            if let application = nextResponder as? UIApplication {
                application.open(url, options: [:], completionHandler: nil)
                break
            }
            responder = nextResponder
        }
    }

    private func close() {
        extensionContext?.completeRequest(returningItems: nil)
    }
}
```

> **Note:** `UIApplication.shared` is unavailable in extensions. The responder chain traversal above is the standard pattern for opening URLs from an extension context.

### Notification Content Extension

Custom notification UI using shared [[04-design-system-as-core-module|DesignSystem]] components. The Tuist target only needs the DesignSystem dependency to keep the binary minimal:

```swift
// In Project.swift
.target(
    name: "NotificationContent",
    destinations: [.iPhone],
    product: .appExtension,
    bundleId: "com.app.notification-content",
    infoPlist: .extendingDefault(with: [
        "NSExtension": [
            "NSExtensionPointIdentifier": "com.apple.usernotifications.content-extension",
            "NSExtensionPrincipalClass": "$(PRODUCT_MODULE_NAME).NotificationViewController",
            "NSExtensionAttributes": [
                "UNNotificationExtensionCategory": "CUSTOM_CATEGORY",
                "UNNotificationExtensionInitialContentSizeRatio": 0.75,
            ],
        ]
    ]),
    sources: ["Targets/NotificationContent/Sources/**"],
    dependencies: [
        .target(name: "DesignSystem"),
        // Only DesignSystem — no networking, no data layer
    ]
)
```

```swift
import UIKit
import UserNotificationsUI
import DesignSystem

class NotificationViewController: UIViewController, UNNotificationContentExtension {
    private let titleLabel = DSLabel(style: .headline)
    private let bodyLabel = DSLabel(style: .body)
    private let imageView = UIImageView()

    func didReceive(_ notification: UNNotification) {
        let content = notification.request.content
        titleLabel.text = content.title
        bodyLabel.text = content.body

        // Load image attachment if present
        if let attachment = content.attachments.first,
           attachment.url.startAccessingSecurityScopedResource() {
            defer { attachment.url.stopAccessingSecurityScopedResource() }
            imageView.image = UIImage(contentsOfFile: attachment.url.path)
        }
    }
}
```

### Binary Size Optimization

App Clips must stay under 15 MB (uncompressed thinned size). Audit and optimize aggressively:

```bash
# Validate App Clip size (must be under 15 MB)
xcrun appcliputil validate --clip MyAppClip.ipa

# Check raw bundle size
du -sh Build/Products/Release-iphoneos/MyAppClip.app

# List frameworks contributing to size
find Build/Products/Release-iphoneos/MyAppClip.app/Frameworks -name "*.framework" -exec du -sh {} \;
```

In the Tuist target or Xcode build settings, apply these optimizations:

```swift
// In Project.swift — build settings for clip/extension targets
.target(
    name: "MyAppClip",
    // ...
    settings: .settings(base: [
        "DEAD_CODE_STRIPPING": "YES",
        "STRIP_INSTALLED_PRODUCT": "YES",
        "ASSETCATALOG_COMPILER_OPTIMIZATION": "space",
        "GCC_OPTIMIZATION_LEVEL": "s",       // Optimize for size
        "SWIFT_OPTIMIZATION_LEVEL": "-Osize", // Swift size optimization
    ])
)
```

Use the Xcode Organizer's size report after archiving to see the thinned size per device variant. The App Store uses the thinned, encrypted size — not the raw build output — so always check the organizer report before submission.

## Edge Cases

- **App Clip 15 MB limit includes all frameworks** — the limit applies to the thinned, uncompressed `.app` bundle including every embedded `.framework`. Audit with `du -sh` on the `.app` bundle after thinning, not just the main binary.
- **App Group container returns nil if entitlement is missing** — always `guard let` the container URL. A missing or mismatched App Group identifier in the entitlements file silently returns `nil` instead of crashing.
- **Share Extension memory limit is 120 MB** — the system kills the extension process if it exceeds this, with no warning or low-memory notification. Large image processing must be done incrementally or deferred to the main app.
- **`openURL` from extensions requires responder chain traversal** — `UIApplication.shared` is unavailable in extension contexts. Walk the responder chain to find the `UIApplication` proxy, or the compiler will reject the call entirely.
- **Keychain sharing requires a separate entitlement** — `keychain-access-groups` must be configured in addition to App Groups. App Group alone does not grant keychain access across targets.
- **App Clip invocation URL must be registered in App Store Connect** — local testing requires an Associated Domains entitlement and a valid AASA (Apple App Site Association) file hosted on your domain. The clip experience cannot be fully tested without this server-side configuration.
- **Notification Content Extension has no network access by default** — design the extension to work entirely with data passed via the notification payload and attachments. Do not rely on network calls to render the UI.

## Why This Matters

- **App Clips provide zero-friction onboarding** — users experience core app functionality without installing, lowering the barrier to conversion
- **Extensions increase engagement from system surfaces** — Share sheet, widgets, and notifications keep users interacting with your app outside the main app context
- **[[01-tuist-modular-architecture|Modular architecture]] makes dependency filtering natural** — you can precisely control what goes into each target by selecting only the Core modules needed, rather than pulling in the entire dependency graph
- **Binary size violations cause App Store rejection** — an App Clip over 15 MB is rejected during review, and oversized extensions degrade system performance
- **[[02-protocol-driven-service-catalog|Protocol-driven dependencies]]** let you swap implementations between app and clip (e.g., a lightweight analytics stub in the clip vs. full analytics in the app)

## Anti-Patterns

- Don't include Feature modules in App Clips — they pull in transitive dependencies (other features, heavy data layers) that balloon binary size past the 15 MB limit
- Don't use Core Data in App Clips unless absolutely necessary — the Core Data framework adds significant binary size; prefer lightweight persistence like `UserDefaults` or `Codable` files in the App Group container
- Don't forget to test App Group data migration — the clip-to-app transition is a critical user journey that is easy to break when the data schema changes
- Don't hardcode App Group identifiers — define them as constants in a shared Core module (e.g., `AppGroupConfig.suiteName`) so all targets reference the same value
- Don't use `UIApplication.shared` in extensions — it is unavailable in the extension sandbox; use the responder chain or extension context APIs instead
- Don't skip binary size auditing in CI — add a build step that runs `xcrun appcliputil validate` and fails the pipeline if the clip exceeds the size limit
- Don't embed asset catalogs with unused images in clip/extension targets — create a separate, minimal asset catalog for each target instead of sharing the main app's full catalog
