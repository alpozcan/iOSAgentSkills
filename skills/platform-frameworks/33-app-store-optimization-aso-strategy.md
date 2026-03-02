---
title: "App Store Optimization (ASO) Strategy"
description: "Complete ASO pipeline for iOS apps: keyword strategy with the 30-30-100 rule, fastlane metadata automation, screenshot and preview video generation, review prompt optimization, in-app events, Custom Product Pages, WidgetKit retention, notifications, A/B testing, App Clips with Spotlight indexing, and retention analytics."
tags: [aso, app-store, keywords, screenshots, fastlane, localization, retention, analytics]
related_skills: [20-fastlane-app-store-connect-publishing, 10-privacy-first-analytics-architecture, 15-widgetkit-and-app-intents-integration, 11-notification-service-with-deep-linking, 23-app-clips-and-shared-extensions-modular-architecture, 16-localization-and-multi-language-patterns, 22-github-actions-ci-cd-for-ios, 07-storekit2-intelligence-based-trial]
category: platform-frameworks
---

# App Store Optimization (ASO) Strategy

## Context

Your iOS app is feature-complete and ready for App Store submission. Code quality, tests, and CI/CD are in place — now the challenge shifts from *building* to *being found*. App Store Optimization is the systematic process of maximizing organic visibility, conversion rate, and retention through metadata, creative assets, and product page configuration. This skill distills a 12-phase ASO strategy into reusable patterns that apply to any iOS app.

## Pattern

### 1. Keyword Strategy — The 30-30-100 Rule

Every App Store locale gives you three keyword surfaces:

| Field | Max Length | Indexed? |
|-------|-----------|----------|
| App Name | 30 chars | Yes |
| Subtitle | 30 chars | Yes |
| Keywords field | 100 chars (comma-separated) | Yes |

**Cross-locale multiplier:** Keywords set in one locale are indexed for that locale only. Supporting 40 locales means 40 independent keyword slots — use each locale's keywords field for locale-specific search terms, not just translations of your primary keywords.

```
fastlane/metadata/en-US/keywords.txt   → "tolkien,middle earth,lotr,fantasy map"
fastlane/metadata/de-DE/keywords.txt   → "mittelerde,herr der ringe,fantasy karte"
fastlane/metadata/ja/keywords.txt      → "指輪物語,中つ国,ファンタジー地図"
```

**Keyword research pattern:**
1. List 50+ candidate terms from competitor listings, autocomplete, and genre terms
2. Deduplicate against name + subtitle (already indexed — don't waste keyword field chars)
3. Prioritize medium-competition terms where you can realistically rank top 10
4. Fill all 100 chars per locale — unused characters are wasted ranking potential
5. Use commas as separators, no spaces after commas (spaces waste characters)
6. Never repeat words across name + subtitle + keywords within the same locale

### 2. Fastlane Metadata Pipeline

Use `deliver` from [[20-fastlane-app-store-connect-publishing]] to push localized metadata for all supported locales:

```ruby
lane :metadata do
  deliver(
    api_key: api_key,
    app_identifier: "com.company.app",
    skip_binary_upload: true,
    skip_app_version_update: true,
    force: true,
    submit_for_review: false,
    automatic_release: false
  )
end
```

Directory structure for N locales:

```
fastlane/metadata/
├── en-US/
│   ├── name.txt            # 30 chars
│   ├── subtitle.txt        # 30 chars
│   ├── description.txt     # 4000 chars
│   ├── keywords.txt        # 100 chars, comma-separated
│   ├── promotional_text.txt # 170 chars, updatable without review
│   └── release_notes.txt   # 4000 chars
├── de-DE/
├── ja/
└── ... (up to 40 locales)
```

**Promotional text** is the only field you can update without a new binary submission — use it for seasonal campaigns, event tie-ins, and A/B messaging tests.

### 3. Screenshot Automation

Use fastlane `snapshot` with a `Snapfile` and `Framefile` to generate device-framed screenshots across locales and device sizes:

```ruby
# Snapfile
devices([
  "iPhone 16 Pro Max",    # 6.9" (required)
  "iPhone 16 Pro",        # 6.3"
  "iPhone SE (3rd generation)", # 4.7" (if supporting)
  "iPad Pro 13-inch (M4)" # iPad (if universal)
])

languages(["en-US", "de-DE", "ja", "fr-FR", "es-ES"])

output_directory("./fastlane/screenshots")
scheme("MyAppUITests")
```

**Framefile.json** adds device bezels and marketing text:

```json
{
  "default": {
    "keyword": { "font": "./fonts/SF-Pro-Display-Bold.otf", "color": "#FFFFFF" },
    "title": { "font": "./fonts/SF-Pro-Display-Regular.otf", "color": "#CCCCCC" },
    "background": "#000000",
    "padding": 50,
    "show_complete_frame": true
  },
  "data": [
    { "filter": "01_", "keyword": "Explore Middle-earth" },
    { "filter": "02_", "keyword": "Interactive Timeline" },
    { "filter": "03_", "keyword": "Detailed Locations" }
  ]
}
```

Required screenshot sizes:

| Device Class | Resolution | Required? |
|-------------|-----------|-----------|
| iPhone 6.9" | 1320 × 2868 | Yes (one of 6.9/6.7/6.5) |
| iPhone 6.7" | 1290 × 2796 | Yes (one of 6.9/6.7/6.5) |
| iPhone 6.5" | 1242 × 2688 | Yes (one of 6.9/6.7/6.5) |
| iPhone 5.5" | 1242 × 2208 | If supporting older devices |
| iPad Pro 13" | 2048 × 2732 | If universal app |

### 4. App Preview Video

Record UI test runs as app preview videos using `simctl`:

```bash
# Start recording
xcrun simctl io booted recordVideo --codec=h264 preview.mp4 &
RECORD_PID=$!

# Run the UI test that demonstrates the feature
xcodebuild test \
  -project MyApp.xcodeproj \
  -scheme MyAppUITests \
  -testPlan PreviewVideoTests \
  -destination 'platform=iOS Simulator,name=iPhone 16 Pro Max'

# Stop recording
kill -INT $RECORD_PID
wait $RECORD_PID
```

**Preview video constraints:**
- Duration: 15–30 seconds
- Resolution: must match device screenshot resolution
- Format: H.264, 30fps
- No device bezels allowed in the video itself (App Store adds them)

Create a dedicated `PreviewVideoTests` UI test class that walks through the app's key features at a pace viewers can follow.

### 5. Review Prompt Optimization

Use `SKStoreReviewController` with session-aware gating from [[07-storekit2-intelligence-based-trial]]:

```swift
import StoreKit

actor ReviewService {
    private let sessionThreshold = 5
    private let featureDiscoveryThreshold = 3

    func requestReviewIfAppropriate() {
        let sessions = UserDefaults.standard.integer(forKey: "sessionCount")
        let features = UserDefaults.standard.integer(forKey: "featuresDiscovered")
        let lastPrompt = UserDefaults.standard.double(forKey: "lastReviewPrompt")
        let daysSinceLastPrompt = (Date().timeIntervalSince1970 - lastPrompt) / 86400

        guard sessions >= sessionThreshold,
              features >= featureDiscoveryThreshold,
              daysSinceLastPrompt > 120 else { return }

        if let scene = UIApplication.shared.connectedScenes
            .compactMap({ $0 as? UIWindowScene }).first {
            SKStoreReviewController.requestReview(in: scene)
            UserDefaults.standard.set(Date().timeIntervalSince1970, forKey: "lastReviewPrompt")
        }
    }
}
```

**Gating rules:**
- Minimum 5 sessions before first prompt
- User must have discovered 3+ features (not just opened the app)
- 120-day cooldown between prompts
- Apple limits to 3 prompts per 365-day period — the system enforces this, but your own cooldown should be longer
- Never prompt after a crash, error, or negative experience

### 6. In-App Events & Seasonal Campaigns

Configure in-app events in App Store Connect to appear on your product page and in editorial features:

```
fastlane/metadata/en-US/promotional_text.txt
→ "🗺️ New: Battle of Helm's Deep — interactive timeline event"
```

**Event deep links** route users to specific content:

```swift
enum DeepLink {
    case event(id: String)
    case location(id: String)
    case timeline(era: String)

    var url: URL {
        switch self {
        case .event(let id): URL(string: "myapp://event/\(id)")!
        case .location(let id): URL(string: "myapp://location/\(id)")!
        case .timeline(let era): URL(string: "myapp://timeline/\(era)")!
        }
    }
}
```

**Seasonal rotation pattern:**
- Rotate `promotional_text.txt` monthly (no binary submission needed)
- Create App Store Connect in-app events for holidays, movie releases, or content updates
- Use event badges (Challenge, Competition, Live Event, Major Update, New Season, Premiere, Special Event) to match the occasion

### 7. Custom Product Pages (CPPs)

Create up to 35 Custom Product Pages, each with unique screenshots, preview videos, and promotional text. Use CPPs to target different audience segments from different ad campaigns:

```
CPP 1: "Fantasy Map Lovers"    → Map-focused screenshots, geographic keywords
CPP 2: "Tolkien Scholars"      → Timeline + lore screenshots, academic keywords
CPP 3: "Movie Fans"            → Film location screenshots, movie-related keywords
```

**Screenshot sets per CPP** must match the same device size requirements as the default page. Measure conversion rate per CPP in App Store Connect Analytics to identify which messaging resonates.

Link CPPs to ad campaigns by appending the CPP ID to your App Store URL:
```
https://apps.apple.com/app/id123456?ppid=custom-page-id
```

### 8. WidgetKit for Retention

Use [[15-widgetkit-and-app-intents-integration]] to maintain daily presence on the Home Screen and Lock Screen:

**Home Screen widget** — rotate daily content (location of the day, event anniversary, quote):

```swift
struct DailyContentProvider: TimelineProvider {
    func getTimeline(in context: Context, completion: @escaping (Timeline<DailyEntry>) -> Void) {
        let today = DailyContent.forDate(Date())
        let entry = DailyEntry(date: Date(), content: today)
        let nextUpdate = Calendar.current.startOfDay(for: Date()).addingTimeInterval(86400)
        completion(Timeline(entries: [entry], policy: .after(nextUpdate)))
    }
}
```

**Lock Screen widget** — compact accessory showing a daily stat or countdown:

```swift
struct LockScreenWidget: Widget {
    var body: some WidgetConfiguration {
        StaticConfiguration(kind: "lockscreen", provider: LockScreenProvider()) { entry in
            LockScreenView(entry: entry)
        }
        .supportedFamilies([.accessoryCircular, .accessoryRectangular, .accessoryInline])
    }
}
```

Widgets drive retention by giving users a reason to interact with your app daily without requiring a push notification.

### 9. Local Notifications for Re-engagement

Use [[11-notification-service-with-deep-linking]] to bring lapsed users back:

```swift
func scheduleReEngagementNotification() {
    let content = UNMutableNotificationContent()
    content.title = String(localized: "notification.reengagement.title")
    content.body = String(localized: "notification.reengagement.body")
    content.sound = .default
    content.userInfo = ["deepLink": "myapp://timeline/third-age"]

    // Fire 7 days after last session
    let trigger = UNTimeIntervalNotificationTrigger(timeInterval: 7 * 86400, repeats: false)
    let request = UNNotificationRequest(identifier: "reengagement", content: content, trigger: trigger)
    UNUserNotificationCenter.current().add(request)
}
```

**Notification cadence:**
- D3: "Did you know?" — highlight an undiscovered feature
- D7: Content update or new content tease
- D14: Last-chance re-engagement with high-value content
- Never more than 1 notification per week for inactive users
- Cancel all pending notifications when the user opens the app

### 10. Product Page A/B Testing

App Store Connect supports native A/B testing of product page elements:

**Testable elements:**
- App icon (up to 3 variants)
- Screenshots (order and content)
- Preview videos

**Test configuration:**
- Traffic split: 50/50 or custom percentages
- Minimum test duration: 7 days (Apple recommendation)
- Statistical significance threshold: 90% confidence

**Attribution with AdServices:**

```swift
import AdServices

func attributeInstall() async {
    do {
        let token = try AAAttribution.attributionToken()
        // Send token to your attribution endpoint
        // Correlate with CPP variant to measure per-campaign conversion
    } catch {
        // Not all installs have attribution — organic installs won't
    }
}
```

Track which Custom Product Page or A/B test variant drove the install to measure creative effectiveness.

### 11. App Clip + Core Spotlight + NSUserActivity

Use [[23-app-clips-and-shared-extensions-modular-architecture]] for instant try-before-install experiences:

**App Clip card** — appears in Safari, Messages, Maps, and NFC:

```swift
// App Clip entry point
@main
struct MyAppClip: App {
    var body: some Scene {
        WindowGroup {
            AppClipView()
                .onContinueUserActivity(NSUserActivityTypeBrowsingWeb) { activity in
                    guard let url = activity.webpageURL else { return }
                    // Route to specific content based on URL
                }
        }
    }
}
```

**Core Spotlight indexing** — make app content searchable from iOS Spotlight:

```swift
import CoreSpotlight

func indexLocation(_ location: Location) {
    let attributes = CSSearchableItemAttributeSet(contentType: .content)
    attributes.title = location.name
    attributes.contentDescription = location.summary
    attributes.thumbnailData = location.thumbnailData

    let item = CSSearchableItem(
        uniqueIdentifier: "location-\(location.id)",
        domainIdentifier: "com.company.app.locations",
        attributeSet: attributes
    )
    CSSearchableIndex.default().indexSearchableItems([item])
}
```

**NSUserActivity** — enable Handoff and Siri suggestions:

```swift
func makeUserActivity(for location: Location) -> NSUserActivity {
    let activity = NSUserActivity(activityType: "com.company.app.viewLocation")
    activity.title = location.name
    activity.isEligibleForSearch = true
    activity.isEligibleForHandoff = true
    activity.isEligibleForPrediction = true
    activity.userInfo = ["locationId": location.id]
    return activity
}
```

### 12. Retention Analytics

Track the metrics that matter for ASO with [[10-privacy-first-analytics-architecture]]:

```swift
enum RetentionEvent: String {
    case appLaunch = "app_launch"
    case sessionStart = "session_start"
    case sessionEnd = "session_end"
    case featureDiscovered = "feature_discovered"
    case bookmarkCreated = "bookmark_created"
    case shareAction = "share_action"
    case widgetTapped = "widget_tapped"
    case deepLinkOpened = "deep_link_opened"
    case notificationTapped = "notification_tapped"
}
```

**Key retention metrics:**

| Metric | Calculation | Healthy Benchmark |
|--------|------------|-------------------|
| D1 Retention | Users active day after install / installs | > 25% |
| D7 Retention | Users active 7 days after install / installs | > 12% |
| D30 Retention | Users active 30 days after install / installs | > 8% |
| Session length | Average time between sessionStart and sessionEnd | > 3 min |
| Feature discovery rate | Users who found 3+ features / total users | > 40% |

**Session tracking pattern:**

```swift
actor SessionTracker {
    private var sessionStart: Date?

    func startSession() {
        sessionStart = Date()
        let count = UserDefaults.standard.integer(forKey: "sessionCount") + 1
        UserDefaults.standard.set(count, forKey: "sessionCount")
        Analytics.track(.sessionStart, properties: ["session_number": count])
    }

    func endSession() {
        guard let start = sessionStart else { return }
        let duration = Date().timeIntervalSince(start)
        Analytics.track(.sessionEnd, properties: ["duration_seconds": Int(duration)])
        sessionStart = nil
    }
}
```

## Edge Cases

- **Mac Catalyst:** If your iOS app runs on Mac via Catalyst, Mac App Store screenshots are separate from iOS — you need Mac-sized screenshots (1280×800 minimum). Keywords are shared across iOS and Mac for the same app record, but the product page is separate.
- **iPad-only screenshots:** If your app is universal, you must provide iPad Pro 13" screenshots (2048×2732) in addition to iPhone. Missing iPad screenshots block submission for universal apps.
- **CJK locale character limits:** Chinese, Japanese, and Korean characters count as 1 character each in the keywords field, but CJK keywords often pack more meaning per character. Optimize by using shorter compound words rather than phrases.
- **RTL locales (ar-SA, he):** Screenshot marketing text must read right-to-left. Verify `Framefile` renders RTL text correctly, or create separate RTL frames.
- **Promotional text timing:** Changes to `promotional_text.txt` propagate within 24 hours — plan seasonal campaigns at least 2 days before the event.
- **App Clip size limit:** App Clips must be under 15MB after thinning. Use only essential assets and shared Core frameworks from [[23-app-clips-and-shared-extensions-modular-architecture]].
- **Review prompt suppression:** `SKStoreReviewController.requestReview(in:)` is silently ignored in TestFlight, debug builds, and when the user has disabled review prompts in Settings.

## Why This Matters

Discoverability drives organic installs. App Store search accounts for ~65% of all app discoveries, making ASO the highest-ROI growth lever available. Every improvement to keyword ranking, screenshot conversion, or retention rate compounds over time — a 10% improvement in conversion rate delivers 10% more installs every day, permanently. Unlike paid acquisition, ASO improvements don't require ongoing spend.

The 12 areas in this skill form a flywheel: better keywords → more impressions → better screenshots → higher conversion → more installs → more reviews → higher ranking → more impressions. Retention (widgets, notifications, engagement features) feeds back into ranking because Apple weights engagement metrics in search results.

## Anti-Patterns

- **Keyword stuffing** — Repeating the same word across name, subtitle, and keywords wastes characters. Apple indexes each word once per locale; duplicates add zero ranking benefit.
- **Translating rather than localizing keywords** — A literal translation of "fantasy map" may not be what German or Japanese users actually search for. Research local search terms per market using App Store Connect search analytics or third-party tools.
- **Prompting reviews too early** — Asking for a review on first launch or before the user has experienced value generates low-star ratings. Wait for demonstrated engagement (5+ sessions, 3+ features discovered).
- **Ignoring promotional text** — It's the only metadata field updatable without review. Not using it for seasonal messaging wastes a free optimization lever.
- **Static screenshots** — Using the same screenshots for years signals an abandoned app. Refresh screenshots with each major release and rotate seasonal marketing text.
- **No deep links in notifications** — Notifications that just open the app to the home screen have lower tap-through rates than those linking to specific, relevant content.
- **Skipping analytics** — Without D1/D7/D30 retention data and feature discovery tracking, you're optimizing blind. Every ASO change should be measurable.
- **One-size-fits-all product page** — Not using Custom Product Pages for different audience segments means your default page must appeal to everyone, diluting its effectiveness for any specific group.

## Related Skills

- [[20-fastlane-app-store-connect-publishing]] — the publishing pipeline that delivers metadata, screenshots, and builds to App Store Connect
- [[10-privacy-first-analytics-architecture]] — privacy-first event tracking for retention metrics and feature discovery
- [[15-widgetkit-and-app-intents-integration]] — WidgetKit patterns for Home Screen and Lock Screen presence
- [[11-notification-service-with-deep-linking]] — notification scheduling with deep link routing for re-engagement
- [[23-app-clips-and-shared-extensions-modular-architecture]] — App Clip architecture with size budget management
- [[16-localization-and-multi-language-patterns]] — in-app localization that aligns with ASO locale strategy
- [[22-github-actions-ci-cd-for-ios]] — CI/CD pipeline for automating screenshot generation and metadata deployment
- [[07-storekit2-intelligence-based-trial]] — trial and monetization patterns that feed review prompt gating
