# Platform Frameworks — Map of Content

Integration patterns for Apple system frameworks in modular iOS and macOS apps.

## The Data Flow Triangle

Three skills form a data pipeline that feeds the rest of the app:

[[12-eventkit-coredata-sync-architecture]] reads calendar events from EventKit and stores them in CoreData via a three-layer actor pipeline. This data flows in three directions: into [[08-on-device-llm-with-apple-foundation-models]] as context for AI analysis, into [[11-notification-service-with-deep-linking]] for smart meeting reminders with video call deep links, and into [[15-widgetkit-and-app-intents-integration]] for Home Screen display via App Groups.

## Monetization & Analytics

[[07-storekit2-intelligence-based-trial]] implements a trial that ends when the AI demonstrates value — not on a fixed timer. The `TrialManager` actor tracks interactions and pattern discoveries, gating LLM query access from [[08-on-device-llm-with-apple-foundation-models]] behind a daily limit that tightens after the trial period.

[[10-privacy-first-analytics-architecture]] tracks typed events with a strict privacy filter — no calendar titles, locations, or personal content ever leaves the device. It consumes subscription events from [[07-storekit2-intelligence-based-trial]] and LLM inference metrics from [[08-on-device-llm-with-apple-foundation-models]], routing them through a pluggable multi-backend system with OSLog as the zero-dependency fallback.

## Distribution & Publishing

[[20-fastlane-app-store-connect-publishing]] covers the complete publishing pipeline: certificate management (Development, Distribution, Developer ID), App Store Connect API key authentication, fastlane lanes for build/upload/metadata/release, xcodebuild archive workflows, notarization, dual distribution to App Store and GitHub Releases, and one-command release workflows via Makefile targets from [[18-makefile-for-ios-project-workflows]].

## How They Connect

```
12 EventKit-CoreData Sync
 ├── 08 On-Device LLM (calendar context for AI)
 ├── 11 Notifications (meeting reminders + deep links)
 └── 15 WidgetKit (Home Screen display via App Groups)

07 StoreKit 2 Trial ←→ 10 Analytics (trial funnel tracking)
       ↓
08 On-Device LLM (query limits gate access)

20 Fastlane Publishing (certificates, App Store, GitHub Releases)
```
