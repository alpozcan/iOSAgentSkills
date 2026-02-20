# iOS Agent Skills — Skill Graph

This is the entry point for the iOS Agent Skills graph. Every node is a self-contained skill file with YAML frontmatter you can scan without reading the full content. Wikilinks between skills carry meaning — follow the ones relevant to your current task, skip the rest.

## How to Navigate

1. **Scan this index** — read the descriptions below to find the right cluster
2. **Open a MOC** — each category has a Map of Content that shows how skills within it connect
3. **Follow wikilinks** — links inside skill prose tell you *when* and *why* another skill matters
4. **Read sections, not files** — most skills have `### subheadings` so you can jump to the relevant part

## Architecture

The structural backbone. These skills define how code is organized, how dependencies flow between modules, and how concurrency is managed across the app.

- [[01-tuist-modular-architecture]] — the module hierarchy that everything else builds on. Core modules own protocols, Feature modules expose factories, the App target is the composition root
- [[02-protocol-driven-service-catalog]] — a ~80-line catalog with no third-party dependencies. Thread-safe resolution via `NSLock`, `KeyPath`-based type lookups, and clean `reset()` for test isolation
- [[03-ui-composer-pattern-for-feature-modules]] — how Feature modules expose Views without leaking their internals. Factories accept protocols, construct ViewModels, return concrete types
- [[06-actor-based-concurrency-patterns]] — when to use `actor` vs `@MainActor` vs `@unchecked Sendable`. The concurrency rules that every service and ViewModel must follow
- [[14-error-handling-and-typed-error-system]] — typed errors with recovery actions, safety classification for AI responses, and localized refusal handling
- [[18-makefile-for-ios-project-workflows]] — the single entry point for every project workflow: build, test, run, snapshot, and App Store submission

**MOC:** [[architecture-moc]]

## UI

Design system infrastructure and navigation patterns for SwiftUI apps.

- [[04-design-system-as-core-module]] — tokens, components, animations, and haptics as a Core module. Dark-mode-only, 4pt grid, OLED-optimized
- [[13-swiftui-custom-tab-bar-and-navigation]] — replacing `TabView` with `safeAreaInset(edge: .bottom)`, re-tap scroll-to-top, splash screen transitions
- [[16-localization-and-multi-language-patterns]] — 43 languages with per-module string catalogs, bilingual AI intent classification, locale-aware safety responses
- [[17-safe-area-inset-stacking-and-bottom-pinned-views]] — why nested `safeAreaInset` fails and how to fix it. The five approaches that don't work and the one that does

**MOC:** [[ui-moc]]

## Testing & Debugging

Quality assurance patterns that keep modular apps reliable.

- [[05-swift-testing-and-tdd-patterns]] — Swift Testing framework over XCTest. Test isolation, five test categories, mock injection, regression tests from real bugs
- [[09-debug-modes-and-mock-service-strategy]] — three-tier mock system (UI testing, rich debug, developer mode) with launch argument flags
- [[17-snapshot-testing-with-swift-snapshot-testing]] — per-module visual regression with a strategic 5-language matrix and 3 device sizes
- [[18-ui-testing-regression-and-smoke]] — XCUITest for critical user journeys: navigation, chat flow, settings, keyboard behavior

**MOC:** [[testing-moc]]

## Platform Frameworks

Patterns for integrating Apple system frameworks into modular apps.

- [[07-storekit2-intelligence-based-trial]] — trial that ends when AI demonstrates value, not on a fixed timer. State machine, daily query limits, StoreKit 2 async API
- [[10-privacy-first-analytics-architecture]] — typed events, privacy filter (never send PII), multi-backend with OSLog fallback
- [[11-notification-service-with-deep-linking]] — AI-generated local notifications with deep links to native video apps across 11 platforms
- [[12-eventkit-coredata-sync-architecture]] — three-layer sync from EventKit to CoreData with programmatic model creation and actor isolation
- [[15-widgetkit-and-app-intents-integration]] — lightweight widgets with App Group data sharing and Siri Shortcuts
- [[20-fastlane-app-store-connect-publishing]] — end-to-end publishing pipeline: certificates, code signing, notarization, fastlane lanes, dual distribution (App Store + GitHub Releases), localized metadata sync, release automation

**MOC:** [[platform-frameworks-moc]]

## AI & Intelligence

On-device AI architecture for privacy-first inference.

- [[08-on-device-llm-with-apple-foundation-models]] — 100% on-device LLM with dynamic prompt gene pool, fitness-weighted selection, safety classification, and response sanitization

**MOC:** [[ai-moc]]

## Cross-Domain Connections

Some of the most important patterns span multiple categories:

- **The data flow triangle:** [[12-eventkit-coredata-sync-architecture]] feeds calendar context into [[08-on-device-llm-with-apple-foundation-models]] for analysis, [[11-notification-service-with-deep-linking]] for meeting reminders, and [[15-widgetkit-and-app-intents-integration]] for Home Screen display
- **The monetization loop:** [[07-storekit2-intelligence-based-trial]] controls access to LLM queries, [[10-privacy-first-analytics-architecture]] tracks the trial funnel, and [[20-fastlane-app-store-connect-publishing]] delivers the app to users
- **The testing pyramid:** [[05-swift-testing-and-tdd-patterns]] provides the foundation, [[09-debug-modes-and-mock-service-strategy]] supplies deterministic data, [[17-snapshot-testing-with-swift-snapshot-testing]] catches visual regressions, and [[18-ui-testing-regression-and-smoke]] validates critical user journeys
- **The build-to-ship pipeline:** [[01-tuist-modular-architecture]] generates the project, [[18-makefile-for-ios-project-workflows]] orchestrates workflows, and [[20-fastlane-app-store-connect-publishing]] handles signing, notarization, and App Store delivery
- **Concurrency everywhere:** [[06-actor-based-concurrency-patterns]] defines the rules that [[02-protocol-driven-service-catalog]], [[12-eventkit-coredata-sync-architecture]], [[07-storekit2-intelligence-based-trial]], and [[10-privacy-first-analytics-architecture]] all follow

## Explorations Needed

- Missing: accessibility patterns (VoiceOver, Dynamic Type scaling, reduced motion). The design system enforces 44pt touch targets but doesn't document VoiceOver traversal order
- Missing: CI/CD pipeline skill (GitHub Actions workflows). The Makefile and fastlane are ready but no workflow YAML exists
- Missing: App Clips or ShareExtension patterns for modular apps
- Scaling question: at what module count does Tuist generation time become a bottleneck?
