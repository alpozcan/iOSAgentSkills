---
title: "StoreKit 2 Subscription System with Intelligence-Based Trial"
description: "Intelligence-based trial ending when AI demonstrates value (50 interactions, 3 patterns, or 21 days). TrialManager actor state machine, DailyQueryTracker, StoreKit 2 async API, and trial funnel analytics."
---

# StoreKit 2 Subscription System with Intelligence-Based Trial

## Context
You need a monetization system that doesn't use a fixed-day free trial (which users game by reinstalling). Instead, the trial ends when the AI demonstrates value — based on interaction count, pattern discovery, or elapsed time. The system must use StoreKit 2's modern async API and feel frictionless.

## Pattern

### Trial State Machine

```
Install → Trial (silent Pro, user doesn't know)
   ↓
ANY of these triggers:
  - 50 AI interactions
  - 3 patterns discovered
  - 21 calendar days
   ↓
Grace Period (48 hours, gentle inline hints)
   ↓
Free Tier (rate-limited, inline upgrade cards)

At ANY point: User subscribes → Pro (permanent until expiration)
```

### TrialManager ([[06-actor-based-concurrency-patterns|Actor-Based]] State Machine)

```swift
public actor TrialManager {
    public enum TrialState: String, Codable, Sendable {
        case trial      // Full Pro, silently
        case grace      // 48h grace after maturity
        case expired    // Free tier
        case subscribed // Paying user
    }
    
    public static let interactionThreshold = 50
    public static let patternThreshold = 3
    public static let maxTrialDays = 21
    public static let gracePeriodHours = 48
    
    private let defaults: UserDefaults
    
    public var currentState: TrialState {
        guard let raw = defaults.string(forKey: Keys.trialState),
              let state = TrialState(rawValue: raw) else { return .trial }
        
        // Auto-expire grace period
        if state == .grace, let graceStart = defaults.object(forKey: Keys.graceStartDate) as? Date {
            let graceEnd = graceStart.addingTimeInterval(Double(Self.gracePeriodHours) * 3600)
            if Date() > graceEnd {
                defaults.set(TrialState.expired.rawValue, forKey: Keys.trialState)
                return .expired
            }
        }
        return state
    }
    
    @discardableResult
    public func evaluate(
        totalInteractions: Int,
        discoveredPatterns: Int,
        isStoreSubscribed: Bool
    ) -> TrialState {
        if isStoreSubscribed {
            defaults.set(TrialState.subscribed.rawValue, forKey: Keys.trialState)
            return .subscribed
        }
        
        let state = currentState
        guard state == .trial else { return state }
        
        if checkMaturity(totalInteractions: totalInteractions, discoveredPatterns: discoveredPatterns) {
            defaults.set(TrialState.grace.rawValue, forKey: Keys.trialState)
            defaults.set(Date(), forKey: Keys.graceStartDate)
            return .grace
        }
        
        return state
    }
}
```

### DailyQueryTracker with Generous First-Week Trial

```swift
public final class DailyQueryTracker: @unchecked Sendable {
    public static let dailyFreeLimit = 5
    public static let generousTrialLimit = 150
    public static let generousTrialDays = 7
    
    public var isInGenerousTrial: Bool {
        guard !defaults.bool(forKey: Keys.generousTrialCompleted) else { return false }
        guard let firstLaunch = defaults.object(forKey: Keys.firstLaunchDate) as? Date else { return false }
        let daysPassed = Calendar.current.dateComponents([.day], from: firstLaunch, to: Date()).day ?? 0
        return daysPassed < Self.generousTrialDays
    }
    
    @discardableResult
    public func recordQuery() -> Bool {
        if isInGenerousTrial {
            let used = defaults.integer(forKey: Keys.generousTrialQueriesUsed)
            guard used < Self.generousTrialLimit else {
                defaults.set(true, forKey: Keys.generousTrialCompleted)
                return false
            }
            defaults.set(used + 1, forKey: Keys.generousTrialQueriesUsed)
            return true
        } else {
            resetIfNewDay()
            let current = defaults.integer(forKey: Keys.queryCount)
            guard current < Self.dailyFreeLimit else { return false }
            defaults.set(current + 1, forKey: Keys.queryCount)
            return true
        }
    }
}
```

### StoreKit 2 Service

```swift
public actor StoreKitSubscriptionService: SubscriptionServiceProtocol {
    private var purchasedProductIDs: Set<String> = []
    
    public func currentStatus() async -> SubscriptionStatus {
        for await result in Transaction.currentEntitlements {
            guard case .verified(let transaction) = result else { continue }
            
            if SubscriptionProductID.allProducts.contains(transaction.productID) {
                // Lifetime purchase — no expiration
                if transaction.productID == SubscriptionProductID.proLifetime {
                    return SubscriptionStatus(tier: .pro, isActive: true)
                }
                // Subscription — check expiry
                if let exp = transaction.expirationDate, exp > Date() {
                    return SubscriptionStatus(tier: .pro, expirationDate: exp, isActive: true)
                }
            }
        }
        return .free
    }
    
    public func purchase(_ product: Product) async throws -> Bool {
        let result = try await product.purchase()
        switch result {
        case .success(let verification):
            guard case .verified(let transaction) = verification else { return false }
            await transaction.finish()
            purchasedProductIDs.insert(transaction.productID)
            return true
        case .userCancelled, .pending: return false
        @unknown default: return false
        }
    }
    
    private func listenForTransactions() async {
        for await result in Transaction.updates {
            guard case .verified(let transaction) = result else { continue }
            purchasedProductIDs.insert(transaction.productID)
            await transaction.finish()
        }
    }
}
```

### Product ID Convention

```swift
public enum SubscriptionProductID {
    public static let proMonthly = "app.wythnos.ios.pro.monthly"    // $1.99
    public static let proAnnual = "app.wythnos.ios.pro.annual"      // $14.99
    public static let proLifetime = "app.wythnos.ios.pro.lifetime"  // $39.99
    
    public static let allSubscriptions: Set<String> = [proMonthly, proAnnual]
    public static let allProducts: Set<String> = [proMonthly, proAnnual, proLifetime]
}
```

### Feature Gating

```swift
public enum ProFeature: String, Sendable {
    case unlimitedChat
    case deepAnalysis
    case allPatterns
    case exportInsights
}

// In ViewModel:
if !isPremium && dailyQueryTracker.isLimitReached {
    analytics.track(.freeLimitHit(type: "daily_query"))
    messages.append(ChatMessage(content: "upgrade_hint", isUser: false))
    return  // Don't send to LLM (see [[08-on-device-llm-with-apple-foundation-models]])
}
```

### [[10-privacy-first-analytics-architecture|Analytics]] Integration for Trial Funnel

```swift
public extension AnalyticsEvent {
    static func trialMaturityReached(days: Int, interactions: Int, patterns: Int) -> AnalyticsEvent {
        AnalyticsEvent(name: "trial_maturity_reached", parameters: [
            "days_to_maturity": "\(days)",
            "total_interactions": "\(interactions)",
            "pattern_count": "\(patterns)"
        ])
    }
    
    static func paywallViewed(source: String) -> AnalyticsEvent {
        AnalyticsEvent(name: "paywall_viewed", parameters: ["source": source])
    }
    
    static func subscriptionStarted(productID: String, trialDay: Int, source: String) -> AnalyticsEvent {
        AnalyticsEvent(name: "subscription_started", parameters: [
            "product_id": productID, "trial_day": "\(trialDay)", "source": source
        ])
    }
}
```

## Why This Matters

- **Intelligence-based trial feels earned** — the user gets Pro access because the AI is "learning" them, creating emotional investment
- **No modal paywalls** — only inline upgrade cards, respecting the user's flow
- **StoreKit 2 async API** — no more delegates, completion handlers, or receipt validation
- **Transaction.currentEntitlements** replaces receipt validation entirely
- **Generous first-week trial (150 queries)** hooks users before they hit limits

## Anti-Patterns

- Don't show a paywall on first launch — the silent trial should be invisible
- Don't use `SKPaymentQueue` (StoreKit 1) — StoreKit 2 is simpler and more reliable
- Don't store subscription state in UserDefaults as source of truth — always verify with `Transaction.currentEntitlements`
- Don't block UI on subscription checks — check asynchronously and default to free
