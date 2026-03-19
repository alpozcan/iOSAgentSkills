---
title: "Reusable About Page & Settings Bundle"
description: "Drop-in About screen with app info, privacy policy, support/feedback, rate on App Store, and Pro subscription with StoreKit 2 IAP. Designed as a reusable pattern for any iOS app."
---

# Reusable About Page & Settings Bundle

## Context

Every iOS app needs an About/Settings screen. Instead of rebuilding it per project, this skill provides a reusable, modular About page that includes: app version info, privacy disclaimer, support/feedback link, App Store rating prompt, and a Pro subscription section with StoreKit 2 IAP. The design adapts to any app's design system tokens.

## Pattern

### Data Model

```swift
struct AboutConfig: Sendable {
    let appName: String
    let appIcon: String          // Asset catalog name
    let version: String          // Auto-derived from Bundle
    let build: String            // Auto-derived from Bundle
    let developerName: String
    let supportEmail: String
    let privacyPolicyURL: URL
    let termsURL: URL?
    let websiteURL: URL?
    let appStoreID: String       // For rating + App Store link
    let proProductID: String     // StoreKit 2 product identifier
    let proFeatures: [String]    // List of Pro features for paywall display

    static var current: AboutConfig {
        let info = Bundle.main.infoDictionary ?? [:]
        return AboutConfig(
            appName: info["CFBundleName"] as? String ?? "",
            appIcon: "AppIcon",
            version: info["CFBundleShortVersionString"] as? String ?? "1.0",
            build: info["CFBundleVersion"] as? String ?? "1",
            developerName: "Your Name",
            supportEmail: "support@yourapp.com",
            privacyPolicyURL: URL(string: "https://yourapp.com/privacy")!,
            termsURL: nil,
            websiteURL: nil,
            appStoreID: "123456789",
            proProductID: "com.yourapp.pro",
            proFeatures: []
        )
    }
}
```

### About View

```swift
import SwiftUI
import StoreKit

struct AboutView: View {
    let config: AboutConfig

    @Environment(\.requestReview) private var requestReview
    @State private var proProduct: Product?
    @State private var isPurchased = false
    @State private var isPurchasing = false

    var body: some View {
        List {
            appInfoSection
            proSection
            supportSection
            legalSection
        }
        .navigationTitle("About")
        .navigationBarTitleDisplayMode(.inline)
        .task { await loadProduct() }
        .task { await checkEntitlement() }
    }

    // MARK: - App Info

    @ViewBuilder
    private var appInfoSection: some View {
        Section {
            HStack(spacing: 16) {
                Image(config.appIcon)
                    .resizable()
                    .frame(width: 60, height: 60)
                    .clipShape(RoundedRectangle(cornerRadius: 14))

                VStack(alignment: .leading, spacing: 4) {
                    Text(config.appName)
                        .font(.headline)
                    Text("Version \(config.version) (\(config.build))")
                        .font(.caption)
                        .foregroundStyle(.secondary)
                }
            }
            .listRowBackground(Color.clear)
            .padding(.vertical, 4)
        }
    }

    // MARK: - Pro Subscription

    @ViewBuilder
    private var proSection: some View {
        Section {
            if isPurchased {
                Label("Pro Unlocked", systemImage: "checkmark.seal.fill")
                    .foregroundStyle(.green)
            } else if let product = proProduct {
                VStack(alignment: .leading, spacing: 12) {
                    Text("Upgrade to Pro")
                        .font(.headline)

                    ForEach(config.proFeatures, id: \.self) { feature in
                        Label(feature, systemImage: "checkmark")
                            .font(.subheadline)
                            .foregroundStyle(.secondary)
                    }

                    Button {
                        Task { await purchase(product) }
                    } label: {
                        HStack {
                            Text(product.displayName)
                            Spacer()
                            Text(product.displayPrice)
                                .fontWeight(.semibold)
                        }
                        .frame(maxWidth: .infinity)
                        .padding(.vertical, 4)
                    }
                    .disabled(isPurchasing)
                }
                .padding(.vertical, 4)
            }

            Button("Restore Purchases") {
                Task { await restorePurchases() }
            }
        } header: {
            Text("Pro")
        }
    }

    // MARK: - Support

    @ViewBuilder
    private var supportSection: some View {
        Section {
            // Rate on App Store
            Button {
                requestReview()
            } label: {
                Label("Rate on App Store", systemImage: "star")
            }

            // Support & Feedback
            Link(destination: URL(string: "mailto:\(config.supportEmail)")!) {
                Label("Support & Feedback", systemImage: "envelope")
            }

            // Website
            if let url = config.websiteURL {
                Link(destination: url) {
                    Label("Website", systemImage: "globe")
                }
            }
        } header: {
            Text("Support")
        }
    }

    // MARK: - Legal

    @ViewBuilder
    private var legalSection: some View {
        Section {
            Link(destination: config.privacyPolicyURL) {
                Label("Privacy Policy", systemImage: "hand.raised")
            }

            if let terms = config.termsURL {
                Link(destination: terms) {
                    Label("Terms of Use", systemImage: "doc.text")
                }
            }
        } header: {
            Text("Legal")
        } footer: {
            VStack(spacing: 4) {
                Text("Made with care by \(config.developerName)")
                Text("\(config.appName) v\(config.version)")
            }
            .font(.caption2)
            .foregroundStyle(.tertiary)
            .frame(maxWidth: .infinity)
            .padding(.top, 16)
        }
    }

    // MARK: - StoreKit 2

    private func loadProduct() async {
        do {
            let products = try await Product.products(for: [config.proProductID])
            proProduct = products.first
        } catch {
            // Silently fail — section just won't show
        }
    }

    private func checkEntitlement() async {
        for await result in Transaction.currentEntitlements {
            if case .verified(let tx) = result, tx.productID == config.proProductID {
                isPurchased = true
                return
            }
        }
    }

    private func purchase(_ product: Product) async {
        isPurchasing = true
        defer { isPurchasing = false }
        do {
            let result = try await product.purchase()
            if case .success(let verification) = result,
               case .verified = verification {
                isPurchased = true
            }
        } catch {
            // Handle purchase error
        }
    }

    private func restorePurchases() async {
        try? await AppStore.sync()
        await checkEntitlement()
    }
}
```

### Usage

```swift
// In any app's navigation
NavigationLink("About") {
    AboutView(config: .current)
}
```

### Customization Points

The `AboutConfig` struct is the single place to configure per-app. Override `.current` with your app's values:

```swift
extension AboutConfig {
    static var current: AboutConfig {
        AboutConfig(
            appName: "Papercut",
            appIcon: "AppIcon",
            version: Bundle.main.infoDictionary?["CFBundleShortVersionString"] as? String ?? "1.0",
            build: Bundle.main.infoDictionary?["CFBundleVersion"] as? String ?? "1",
            developerName: "Alp Ozcan",
            supportEmail: "support@papercut.app",
            privacyPolicyURL: URL(string: "https://papercut.app/privacy")!,
            termsURL: URL(string: "https://papercut.app/terms"),
            websiteURL: URL(string: "https://papercut.app"),
            appStoreID: "123456789",
            proProductID: "com.papercut.pro.yearly",
            proFeatures: [
                "Multiple widget styles",
                "Custom themes (cream, dark, sepia)",
                "Custom share card styles",
                "Alternative app icons",
                "Export quotes as CSV/JSON"
            ]
        )
    }
}
```

## Rules

- **Rate prompt:** Use `@Environment(\.requestReview)` — never call `SKStoreReviewController` directly. Apple throttles it automatically.
- **Privacy policy:** Required for any app with network access or data collection. Link to a hosted page, not inline text.
- **StoreKit 2 only:** Do not use original StoreKit (`SKPaymentQueue`). StoreKit 2 async API is the modern standard.
- **Restore button:** Always include "Restore Purchases" — App Store Review guideline 3.1.1.
- **Footer:** Include developer attribution and version — builds trust and helps support debug.
- **No account:** This pattern is for apps without user accounts. If your app has auth, extend with account management section.

## Anti-Patterns

- ❌ DON'T hardcode version strings — always derive from `Bundle.main.infoDictionary`
- ❌ DON'T open App Store externally for rating — use the in-app review API
- ❌ DON'T show subscription price without loading from StoreKit — prices vary by region
- ❌ DON'T forget to handle the case where StoreKit products fail to load — section should gracefully disappear
