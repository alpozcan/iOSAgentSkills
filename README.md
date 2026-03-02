# iOS Agent Skills

> **Production-ready architectural patterns and implementation skills for building modern iOS apps with Swift, SwiftUI, and Tuist — designed as context for AI coding agents.**

Created and maintained by [**Alp Özcan**](https://github.com/alpozcan).

---

## Why This Exists

AI coding agents are powerful, but they lack the opinionated, battle-tested architectural knowledge that comes from shipping real iOS apps. They'll happily generate a `ViewController` with 2,000 lines, wire up Singletons everywhere, and skip actor isolation entirely.

**iOS Agent Skills** is a collection of 16 self-contained skill documents that teach AI agents (and developers) how to build modular, testable, privacy-first iOS applications using modern Swift patterns. Each skill encodes a specific architectural decision — the kind of thing a senior iOS engineer carries in their head but rarely writes down.

These skills were extracted from building a production iOS app with 10+ Tuist modules, on-device LLM integration (Apple Foundation Models), StoreKit 2 subscriptions, EventKit–CoreData sync, and a fully custom SwiftUI design system. They represent real patterns that survived contact with real users.

## What's Inside

**19 skills** organized across **5 categories**, covering everything from build system configuration to on-device AI safety guardrails.

### 🏗️ [Architecture](skills/architecture/) — 6 skills

The structural backbone of a modular iOS app. These skills define how code is organized, how dependencies flow, how services communicate safely across threads, how errors propagate with full type safety, and how all workflows are orchestrated through a single entry point.

| # | Skill | What You'll Learn |
|---|-------|-------------------|
| 01 | [Tuist-Based Modular Architecture](skills/architecture/01-tuist-modular-architecture.md) | How to define a `Project.swift` with Core/Feature module layers, enforce strict dependency boundaries at compile time, eliminate Xcode project merge conflicts, and structure a directory convention that scales to 10+ modules. Includes helper functions for `coreFramework()` and `featureFramework()` with automatic test target generation. |
| 02 | [Protocol-Driven Service Catalog](skills/architecture/02-protocol-driven-service-catalog.md) | A complete service catalog in ~80 lines of Swift — no third-party frameworks. Three-layer architecture: Protocol (module-owned), Blueprint (factory), and Registration (composition root). Thread-safe resolution via `NSLock`, `KeyPath`-based type-safe lookups, shared vs. transient registration, and clean `reset()` for test isolation. |
| 03 | [UI Composer Pattern for Feature Modules](skills/architecture/03-ui-composer-pattern-for-feature-modules.md) | How Feature modules expose a single `UIComposer` that accepts pre-resolved dependencies via constructor injection and produces concrete SwiftUI Views (no `AnyView`). Covers `compose*View()` variations for callbacks, runtime data, and the ViewModel-as-dependency-hub pattern. Eliminates service locator calls from Views entirely. |
| 06 | [Actor-Based Concurrency Patterns](skills/architecture/06-actor-based-concurrency-patterns.md) | When to use `actor` vs `@MainActor` vs `@unchecked Sendable` with `NSLock`. Covers actor-isolated data stores (CoreData), `nonisolated` escape hatches for `AsyncThrowingStream`, actor-to-actor communication chains, `Sendable` value-type domain models, and app lifecycle integration. Prevents data races at compile time. |
| 14 | [Typed Error System with Recovery Actions](skills/architecture/14-error-handling-and-typed-error-system.md) | A `WythnosError` enum where every case maps to a title, message, SF Symbol icon, and `RecoveryAction` — rendered through a reusable `NyxErrorCard` design system component. Includes AI safety classification (`SafetyClassification`), localized refusal responses in English and Turkish, and emergency crisis resource handling. |
| 18 | [Makefile for iOS Project Workflows](skills/architecture/18-makefile-for-ios-project-workflows.md) | A self-documenting Makefile as the single entry point for every project workflow: Tuist lifecycle (`make setup`), multiple launch modes (`make run`, `make free`, `make fresh`, `make dev-mode`), unit and snapshot testing (`make test`, `make snapshots`, per-module `make snapshots-design`), snapshot recording (`make snapshots-record`), simulator management, and log streaming. Covers `CODE_SIGNING_ALLOWED=NO` for agents/CI, `_ensure-workspace` auto-generation, `define` functions for app discovery/launch, and `SIMULATOR=` override for targeting any device. |

### 🎨 [UI](skills/ui/) — 4 skills

Design system infrastructure, custom navigation, and multi-language support for SwiftUI apps that feel premium.

| # | Skill | What You'll Learn |
|---|-------|-------------------|
| 04 | [Design System as Core Module](skills/ui/04-design-system-as-core-module.md) | A four-layer design system: tokens (colors, typography, spacing, corner radii as static enums), reusable components (`NyxCard`, `NyxErrorCard`, `NyxFeedbackToggle`, `AutoSizingTextEditor`, `FlowLayout`), animation components (typed text, count-up numbers, slide-up/breathing-glow modifiers), and a centralized haptic feedback system with pre-initialized generators. Dark-mode-only, 4pt grid, OLED-optimized true black. |
| 13 | [Custom Tab Bar & Navigation](skills/ui/13-swiftui-custom-tab-bar-and-navigation.md) | Replacing `TabView` with a fully custom tab bar using `safeAreaInset(edge: .bottom)` (the correct API — not `overlay` or `ZStack`). Covers re-tap detection for scroll-to-top via `.id()` change, spring-physics `ButtonStyle`, breathing glow animations for the active tab glyph, keyboard avoidance with `.ignoresSafeArea(.keyboard)`, splash screen → content transitions, and onboarding → main app flow. |
| 16 | [Localization & Multi-Language Patterns](skills/ui/16-localization-and-multi-language-patterns.md) | Supporting 43 languages in a modular Tuist app where each framework owns its string catalogs via `bundle: .module`. Covers bilingual AI intent classification (English + Turkish keywords), locale-aware gene pools for AI personality, localized safety refusal responses for crisis situations, and test coverage for multi-language keyword detection. |
| 17 | [safeAreaInset Stacking & Bottom-Pinned Views](skills/ui/17-safe-area-inset-stacking-and-bottom-pinned-views.md) | Why nested `safeAreaInset(edge: .bottom)` hides inner views behind the outer one. Documents five approaches that **don't work** (VStack below ScrollView, inner `safeAreaInset`, overlay, layoutPriority, toolbar bottomBar) and the root cause: the outer `safeAreaInset` always wins in Z-order. The fix: place all bottom-pinned chrome (tab bar + per-tab input bars) inside a **single** `safeAreaInset` block at the outermost level, with conditional content per tab. |

### 🧪 [Testing & Debugging](skills/testing-and-debugging/) — 4 skills

Quality assurance patterns that keep modular apps reliable across CI, simulators, and Xcode Previews.

| # | Skill | What You'll Learn |
|---|-------|-------------------|
| 05 | [Swift Testing & TDD Patterns](skills/testing-and-debugging/05-swift-testing-and-tdd-patterns.md) | Modern Swift Testing framework (`@Suite`, `@Test`, `#expect`) over XCTest for all unit tests. Test isolation via `UserDefaults(suiteName: #function)`, five categories of tests (service logic, catalog, AI engine, sanitizer/filter, actor-based async), stub override via `Catalog.main.supply()`, UI test base class with launch arguments, and regression tests driven by real bug screenshots. |
| 09 | [Debug Modes & Mock Services](skills/testing-and-debugging/09-debug-modes-and-mock-service-strategy.md) | A three-tier mock system: Tier 1 (UI testing — minimal, fixed data), Tier 2 (rich debug — 30 days of realistic calendar patterns with Pro access), and Tier 3 (developer mode — secret gesture activation with SHA256-hashed codes for QA testers without Xcode). Shows how to override `Catalog` registrations via launch arguments (`--uitesting`, `--pro-debug`) and re-supply dependent services for graph consistency. |
| 17 | [Snapshot Testing with swift-snapshot-testing](skills/testing-and-debugging/17-snapshot-testing-with-swift-snapshot-testing.md) | Per-module visual regression testing using Point-Free's swift-snapshot-testing in a Tuist modular app. Covers strategic language selection (5 of 43 locales: en, ar/RTL, de/long words, ja/CJK, tr), device matrix (SE/Pro/Pro Max), `SnapshotTestSupport` static framework with `ENABLE_TESTING_SEARCH_PATHS`, component-level and full-screen assertions, deterministic mock factories, recording vs verification modes, CI pipeline configuration, and precision tolerance for cross-platform rendering. |
| 18 | [UI Testing for Regression & Smoke Testing](skills/testing-and-debugging/18-ui-testing-regression-and-smoke.md) | Comprehensive XCUITest suite for critical user journeys. Covers test base class with launch arguments (`--uitesting`, `--skip-onboarding`), accessibility identifier conventions, tab navigation tests, chat flow (send message, feedback buttons), chat history (open, select session, new chat), settings (name field, privacy links, web views, feedback button), keyboard behavior (show/dismiss, tab switch), CI integration, Makefile targets, and handling flaky tests with `waitForExistence` and predicates. |
| 25 | [Performance Monitoring & Profiling Patterns](skills/testing-and-debugging/25-performance-monitoring-and-profiling-patterns.md) | MetricKit integration for crash/hang metrics and diagnostic payloads, `OSSignposter` for custom interval measurement (LLM inference, calendar sync), launch time profiling in three phases (pre-main, catalog registration, first frame), memory pressure tests with `mach_task_basic_info` for 10K-event sync validation, privacy-aware metric reporting (typed `PerformanceMetric` struct, no PII), and Tuist generation time profiling at scale. |
| 26 | [Structured Logging & Log Levels](skills/testing-and-debugging/26-structured-logging-and-log-levels.md) | Per-module `os.Logger` configuration with subsystem/category organization mirroring Tuist module boundaries, strict log level discipline (debug/info/notice/error/fault decision tree), privacy annotations for PII protection (`privacy: .private`, `.private(mask: .hash)`), log forging prevention for user-controlled strings, `OSLogStore` for in-app log retrieval in debug builds, Console.app and `log` CLI filtering commands, Makefile target integration, and Sendable-safe logging in actors. |

### 📱 [Platform Frameworks](skills/platform-frameworks/) — 6 skills

Patterns for integrating Apple system frameworks (StoreKit, EventKit, WidgetKit, UNUserNotificationCenter) into modular apps, plus App Store publishing and direct distribution pipelines.

| # | Skill | What You'll Learn |
|---|-------|-------------------|
| 07 | [StoreKit 2 Subscription System](skills/platform-frameworks/07-storekit2-intelligence-based-trial.md) | An intelligence-based trial that ends when the AI demonstrates value (50 interactions, 3 patterns discovered, or 21 days) — not a fixed free trial. Covers the `TrialManager` actor state machine (trial → grace → expired → subscribed), `DailyQueryTracker` with generous first-week limits (150 queries), StoreKit 2 async purchase/entitlement API, product ID conventions, feature gating, and trial funnel analytics. |
| 10 | [Privacy-First Analytics Architecture](skills/platform-frameworks/10-privacy-first-analytics-architecture.md) | A pluggable multi-backend analytics actor with typed event factory methods (no free-form strings). Covers the privacy filter (what is NEVER sent: calendar titles, locations, notes, user names), `nonisolated` fire-and-forget tracking from any context, buffer + flush for network efficiency, OSLog backend as zero-dependency fallback, and graceful TelemetryDeck configuration via Info.plist. |
| 11 | [Notification Service with Deep Linking](skills/platform-frameworks/11-notification-service-with-deep-linking.md) | An actor-based notification service that schedules LLM-generated local notifications (weekly summaries, meeting reminders, insight milestones). Deep linking to native video call apps (Zoom, Meet, Teams, Webex — 11 platforms) by extracting URLs from event locations and converting to native URL schemes. Covers engagement-gated permission requests, smart meeting importance scoring, and `NotificationPreferences` with `Codable` persistence. |
| 12 | [EventKit–CoreData Sync Architecture](skills/platform-frameworks/12-eventkit-coredata-sync-architecture.md) | A three-layer sync pipeline: `CalendarService` (EventKit abstraction) → `CalendarSyncManager` (orchestrator actor) → `CalendarStore` (CoreData actor). Covers programmatic CoreData model creation (no `.xcdatamodeld` — better for Tuist framework targets), upsert by ID, `NSMergeByPropertyObjectTrumpMergePolicy`, full sync (90 days back + 7 forward), incremental sync from last sync date, `Sendable` value-type domain models, and `inMemory: true` for fast tests. |
| 15 | [WidgetKit & App Intents Integration](skills/platform-frameworks/15-widgetkit-and-app-intents-integration.md) | Lightweight Home Screen widgets as a separate Tuist target with no framework dependencies. Covers `TimelineProvider` with placeholder/snapshot/timeline, multi-size widget views (`systemSmall` + `systemMedium`), `containerBackground(.black, for: .widget)` (iOS 17+), `AppIntent` for Siri Shortcuts, data sharing via App Groups, and 15-minute refresh cadence for battery efficiency. |
| 20 | [App Store Connect Publishing Pipeline](skills/platform-frameworks/20-fastlane-app-store-connect-publishing.md) | End-to-end publishing pipeline: certificate creation (Development, Distribution, Developer ID), App Store Connect API key auth, fastlane lanes (create_app, build, upload, metadata, release) for iOS and macOS, xcodebuild archive workflows, ExportOptions plist, Developer ID direct distribution with notarization, GitHub Releases, entitlements management, Makefile integration for App Store and direct distribution targets, and a first-time setup script. |

### 🤖 [AI & Intelligence](skills/ai-and-intelligence/) — 1 skill

On-device AI architecture for privacy-first, zero-latency inference.

| # | Skill | What You'll Learn |
|---|-------|-------------------|
| 08 | [On-Device LLM with Apple Foundation Models](skills/ai-and-intelligence/08-on-device-llm-with-apple-foundation-models.md) | 100% on-device inference using Apple's Foundation Models framework (iOS 26+) — no model download, no API key, no data leaves the device. Covers protocol-first `LLMEngineProtocol`, `LanguageModelSession` for single-shot and streaming responses, a dynamic prompt gene pool system (10 gene types: persona, format, domain instruction, context template, evolution directive, insight pattern, emotional tone, language mixing, error recovery, safety guardrail), fitness-weighted gene selection, feedback-driven evolution (positive → +0.05, negative → −0.08, low-fitness triggers mutation), safety classification decorator with localized refusal responses, and response sanitization to strip LLM artifacts before display. |

---

## How to Use These Skills

### 🤖 As AI Agent Context

Point your AI coding agent at individual skill files or entire categories to give it production-grade iOS knowledge:

```bash
# Load a specific skill as context
@skills/architecture/01-tuist-modular-architecture.md

# Load all architecture skills for a broad foundation
@skills/architecture/

# Load everything for a comprehensive iOS knowledge base
@skills/
```

Each skill is **self-contained** — it provides enough context for an AI agent to implement the pattern correctly without reading other skills. Cross-references between skills are informational, not required.

### 📖 As a Developer Reference

Browse the categories above and jump to whichever skill matches your current implementation challenge. The skills are numbered for reference but can be read in any order.

### 🧩 As a Starting Point for Your Own Skills

Fork this repo and add skills specific to your project's domain. The [CONTRIBUTING.md](CONTRIBUTING.md) guide explains the file naming convention, template structure, and quality checklist.

---

## Skill Structure

Every skill follows the same three-part structure for consistency:

```
# Title

## Context
When and why you need this pattern. What problem does it solve?

## Pattern
The concrete implementation with Swift code.
Broken into layers/steps with ### subheadings.

## Why This Matters / Anti-Patterns
Guardrails — what to do and what to avoid.
```

This structure is designed for AI agent consumption: the **Context** helps the agent decide *whether* to apply the skill, the **Pattern** provides *how*, and the **Anti-Patterns** prevent common mistakes.

---

## Tech Stack

These skills assume the following technology stack:

| Layer | Technology |
|-------|-----------|
| Language | **Swift 6+** with strict concurrency checking |
| UI Framework | **SwiftUI** (iOS 17+ minimum, some skills target iOS 26+) |
| Build System | **Tuist** for project generation and module management |
| Testing | **Swift Testing** framework (`@Suite`, `@Test`, `#expect`) |
| Persistence | **CoreData** with programmatic model creation |
| AI / ML | **Apple Foundation Models** (on-device LLM, iOS 26+) |
| Monetization | **StoreKit 2** (async API, `Transaction.currentEntitlements`) |
| Analytics | **TelemetryDeck** (privacy-first) with pluggable backend protocol |
| System Frameworks | EventKit, WidgetKit, UNUserNotificationCenter, AppIntents |

---

## Project Structure

```
iOSAgentSkills/
├── README.md                              ← You are here
├── CONTRIBUTING.md                        ← How to add new skills
├── LICENSE                                ← MIT License
│
└── skills/
    ├── architecture/                      ← 5 skills
    │   ├── README.md
    │   ├── 01-tuist-modular-architecture.md
    │   ├── 02-protocol-driven-service-catalog.md
    │   ├── 03-ui-composer-pattern-for-feature-modules.md
    │   ├── 06-actor-based-concurrency-patterns.md
    │   └── 14-error-handling-and-typed-error-system.md
    │
    ├── ui/                                ← 4 skills
    │   ├── README.md
    │   ├── 04-design-system-as-core-module.md
    │   ├── 13-swiftui-custom-tab-bar-and-navigation.md
    │   ├── 16-localization-and-multi-language-patterns.md
    │   └── 17-safe-area-inset-stacking-and-bottom-pinned-views.md
    │
    ├── testing-and-debugging/             ← 4 skills
    │   ├── README.md
    │   ├── 05-swift-testing-and-tdd-patterns.md
    │   ├── 09-debug-modes-and-mock-service-strategy.md
    │   ├── 17-snapshot-testing-with-swift-snapshot-testing.md
    │   └── 18-ui-testing-regression-and-smoke.md
    │
    ├── platform-frameworks/               ← 6 skills
    │   ├── README.md
    │   ├── 07-storekit2-intelligence-based-trial.md
    │   ├── 10-privacy-first-analytics-architecture.md
    │   ├── 11-notification-service-with-deep-linking.md
    │   ├── 12-eventkit-coredata-sync-architecture.md
    │   ├── 15-widgetkit-and-app-intents-integration.md
    │   └── 20-fastlane-app-store-connect-publishing.md
    │
    └── ai-and-intelligence/               ← 1 skill
        ├── README.md
        └── 08-on-device-llm-with-apple-foundation-models.md
```

---

## Contributing

Contributions are welcome! Whether it's a new skill, a correction to an existing one, or an improvement to the project structure.

See [**CONTRIBUTING.md**](CONTRIBUTING.md) for the full guide — including the skill template, naming convention, category descriptions, and quality checklist.

**Quick start:**

1. Fork the repo
2. Create your skill: `skills/<category>/XX-descriptive-name.md`
3. Follow the Context → Pattern → Anti-Patterns structure
4. Add it to the relevant table in this README
5. Open a pull request

---

## License

This project is licensed under the **MIT License** — see the [LICENSE](LICENSE) file for details.

---

<p align="center">
  Built with care by <a href="https://github.com/alpozcan"><strong>@alpozcan</strong></a>
</p>
