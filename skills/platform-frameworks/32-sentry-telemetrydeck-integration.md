---
title: "Sentry & TelemetryDeck Integration"
description: "Privacy-first crash reporting (Sentry) and anonymous usage analytics (TelemetryDeck) for macOS/iOS apps — programmatic Sentry project creation, SDK setup, typed events, App Store privacy labels, and App Store Connect data collection questionnaire."
---

# Sentry & TelemetryDeck Integration

## Context

iOS and macOS apps need crash reporting and usage analytics to improve quality and understand feature adoption. Sentry provides crash reports with stack traces; TelemetryDeck provides anonymous usage analytics with no PII. Both are privacy-first and designed for App Store compliance. The key constraint: **no screen content, code, user data, or personal identifiers may ever leave the device.**

## Pattern

### Programmatic Sentry Project Creation

Sentry has a full REST API. Create a project for a new app without touching the dashboard:

```bash
# 1. Create an auth token at https://sentry.io/settings/auth-tokens/
#    Required scope: project:write (or org:write if member creation is restricted)

# 2. Create the project
curl -X POST "https://sentry.io/api/0/teams/{org-slug}/{team-slug}/projects/" \
  -H "Authorization: Bearer $SENTRY_AUTH_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "YourApp",
    "slug": "yourapp",
    "platform": "apple-macos",
    "default_rules": true
  }'

# Response includes the project DSN:
# { "id": "...", "slug": "yourapp", "dsn": { "public": "https://...@sentry.io/..." } }

# 3. Retrieve the client DSN (if you missed it from the create response)
curl "https://sentry.io/api/0/projects/{org-slug}/yourapp/keys/" \
  -H "Authorization: Bearer $SENTRY_AUTH_TOKEN"
# Returns array with dsn.public for each key
```

**Platform values:** `apple-macos` for macOS, `apple-ios` for iOS, `swift` for generic Swift.

**Scripted setup (Makefile):**

```makefile
sentry-project:
	@curl -s -X POST "https://sentry.io/api/0/teams/$(SENTRY_ORG)/$(SENTRY_TEAM)/projects/" \
		-H "Authorization: Bearer $(SENTRY_AUTH_TOKEN)" \
		-H "Content-Type: application/json" \
		-d '{"name": "$(APP_NAME)", "platform": "apple-macos"}' \
		| python3 -c "import sys,json; d=json.load(sys.stdin); print(f'SENTRY_DSN={d[\"dsn\"][\"public\"]}')"
```

### TelemetryDeck App Creation

TelemetryDeck has **no management API** — app creation is dashboard-only:

1. Log in at [dashboard.telemetrydeck.com](https://dashboard.telemetrydeck.com)
2. Click the dropdown next to the TelemetryDeck logo → **Create New App**
3. Name your app
4. Go to **Settings → Set Up App** → copy the **App ID** (UUID format)

Store the App ID in your `.env` file alongside the Sentry DSN.

### SPM Dependencies

```swift
// Package.swift
dependencies: [
    .package(url: "https://github.com/getsentry/sentry-cocoa", from: "8.0.0"),
    .package(url: "https://github.com/TelemetryDeck/SwiftSDK", from: "2.0.0"),
],
targets: [
    .target(
        name: "YourCore",
        dependencies: [
            .product(name: "Sentry", package: "sentry-cocoa"),
            .product(name: "TelemetryDeck", package: "SwiftSDK"),
        ]
    ),
]
```

### Secrets Management

Generate secrets at build time from a `.env` file — never commit API keys to source control:

**.env** (gitignored):
```
SENTRY_DSN=https://abc123@o12345.ingest.sentry.io/67890
TELEMETRYDECK_APP_ID=B47A1234-5678-9ABC-DEF0-123456789ABC
```

**Pre-build script** (in `project.yml` or Xcode build phases):
```bash
ENV_FILE="${SRCROOT}/../.env"
OUTPUT="${SRCROOT}/Kesit/Secrets.swift"

if [ -f "$ENV_FILE" ]; then
    SENTRY_DSN=$(grep SENTRY_DSN "$ENV_FILE" | cut -d'=' -f2)
    TD_APP_ID=$(grep TELEMETRYDECK_APP_ID "$ENV_FILE" | cut -d'=' -f2)
else
    SENTRY_DSN=""; TD_APP_ID=""
fi

cat > "$OUTPUT" << EOF
enum Secrets {
    static let sentryDSN = "$SENTRY_DSN"
    static let telemetryDeckAppID = "$TD_APP_ID"
}
EOF
```

Add to `.gitignore`:
```
.env
Kesit/Secrets.swift
```

### Analytics Facade

A single entry point wrapping both services:

```swift
import Sentry
import TelemetryDeck

@MainActor
public enum AppAnalytics {

    private static var isConfigured = false

    // MARK: - Configuration

    public static func configure(sentryDSN: String, telemetryDeckAppID: String) {
        guard !isConfigured else { return }
        isConfigured = true

        // Sentry — crash reporting
        SentrySDK.start { options in
            options.dsn = sentryDSN
            options.tracesSampleRate = 0.2
            options.enableAutoSessionTracking = true
            options.enableCaptureFailedRequests = false
            options.enableNetworkBreadcrumbs = false  // Privacy: no network tracking
            options.debug = false
            #if DEBUG
            options.enabled = false
            #endif
        }

        // TelemetryDeck — anonymous usage analytics
        let config = TelemetryDeck.Config(appID: telemetryDeckAppID)
        TelemetryDeck.initialize(config: config)
    }

    // MARK: - Events

    public static func track(_ event: AnalyticsEvent, parameters: [String: String] = [:]) {
        TelemetryDeck.signal(event.rawValue, parameters: parameters)
    }

    public static func captureError(_ error: Error, context: [String: Any] = [:]) {
        #if !DEBUG
        SentrySDK.capture(error: error) { scope in
            for (key, value) in context {
                scope.setExtra(value: value, key: key)
            }
        }
        #endif
    }
}

// MARK: - Typed Events

public enum AnalyticsEvent: String, Sendable {
    case appLaunched = "app.launched"
    case captureStarted = "capture.started"
    case pipelineCompleted = "pipeline.completed"
    case skillApplied = "skill.applied"
    case paywallOpened = "paywall.opened"
    case purchaseCompleted = "purchase.completed"
    case purchaseFailed = "purchase.failed"
}
```

**Key design decisions:**
- `@MainActor` enum — no instances, all static methods, matches ViewModel isolation
- Sentry disabled in `#if DEBUG` — no noise during development
- `captureError` gated behind `#if !DEBUG` — prevents test crashes polluting Sentry
- Network breadcrumbs disabled — privacy: don't log HTTP requests
- Typed `AnalyticsEvent` enum — prevents typos, discoverable via autocomplete

### App Store Privacy Labels

When submitting to the App Store, declare the following in App Store Connect:

**"Do you or your third-party partners collect data from this app?"** → **Yes**

**Data types to declare:**

| Data Type | Collected | Linked to Identity | Used for Tracking |
|-----------|-----------|-------------------|-------------------|
| Crash Data | Yes | No | No |
| Performance Data | Yes | No | No |

**Crash Data purpose:** App Functionality (minimize crashes, improve stability)
**Performance Data purpose:** Analytics (understand feature usage)

**Do NOT check:** Identifiers, Contact Info, User Content, Location, Browsing History, or any other category.

### App Store Connect Data Collection Questionnaire

When App Store Connect asks:

1. **"Do you or your third-party partners collect data from this app?"** → Yes
2. **Crash Data → How is it used?** → App Functionality
3. **Crash Data → Linked to user's identity?** → No
4. **Crash Data → Used for tracking?** → No
5. **Performance Data → How is it used?** → Analytics
6. **Performance Data → Linked to user's identity?** → No
7. **Performance Data → Used for tracking?** → No

## Edge Cases

- **Sentry DSN vs auth token:** The DSN (Data Source Name) is a public client-side key used by the SDK to send events. The auth token is a private server-side key for the REST API. Never ship the auth token in the app.
- **TelemetryDeck App ID is not secret:** It's designed to be embedded in the app binary. It can only be used to send signals, not read data.
- **Sentry in debug builds:** Always disable via `options.enabled = false` in `#if DEBUG` to avoid polluting crash reports with development issues.
- **Rate limiting:** Sentry has event quotas per plan. Set `tracesSampleRate` to 0.1-0.2 to stay within limits.
- **Privacy policy requirement:** Both Sentry and TelemetryDeck must be disclosed in your privacy policy. See [[24-github-org-profile-and-website]] for the privacy page template.
- **Opt-out:** Provide a toggle in Settings. For Sentry: `SentrySDK.close()`. For TelemetryDeck: don't call `TelemetryDeck.signal()`.

## Why This Matters

- **Sentry catches crashes you'd never hear about** — most users don't report bugs, they just leave
- **TelemetryDeck shows which features matter** — essential for prioritizing development
- **Both are privacy-first** — no PII, no IP retention (TelemetryDeck), no screen content
- **Programmatic Sentry setup** means new projects can be created in CI/CD without manual dashboard clicks
- **Typed events** prevent analytics drift — every event is discoverable and refactorable

## Anti-Patterns

- Don't use `print()` or `NSLog()` for production diagnostics — use Sentry for crashes and structured logging for debug info
- Don't send user content (code, text, screenshots) to analytics — only send event names and anonymous metadata
- Don't hardcode DSN/App ID in source — generate from `.env` at build time
- Don't enable Sentry in debug builds — it pollutes crash data and slows iteration
- Don't skip the App Store privacy label — Apple will reject the submission
- Don't use Sentry for analytics or TelemetryDeck for crash reporting — each tool has its purpose
