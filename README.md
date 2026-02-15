# iOS Agent Skills

> **Production-ready architectural patterns and implementation skills for building modern iOS apps with Swift, SwiftUI, and Tuist — designed as context for AI coding agents.**

Created and maintained by [**Alp Özcan**](https://github.com/alpozcan).

---

## Why This Exists

AI coding agents are powerful, but they lack the opinionated, battle-tested architectural knowledge that comes from shipping real iOS apps. They'll happily generate a `ViewController` with 2,000 lines, wire up Singletons everywhere, and skip actor isolation entirely.

**iOS Agent Skills** is a collection of 16 self-contained skill documents that teach AI agents (and developers) how to build modular, testable, privacy-first iOS applications using modern Swift patterns. Each skill encodes a specific architectural decision — the kind of thing a senior iOS engineer carries in their head but rarely writes down.

These skills were extracted from building a production iOS app with 10+ Tuist modules, on-device LLM integration (Apple Foundation Models), StoreKit 2 subscriptions, EventKit–CoreData sync, and a fully custom SwiftUI design system. They represent real patterns that survived contact with real users.

## What's Inside

**16 skills** organized across **5 categories**, covering everything from build system configuration to on-device AI safety guardrails.

### 🏗️ [Architecture](skills/architecture/) — 5 skills

The structural backbone of a modular iOS app. These skills define how code is organized, how dependencies flow, how services communicate safely across threads, and how errors propagate with full type safety.

| # | Skill | What You'll Learn |
|---|-------|-------------------|
| 01 | [Tuist-Based Modular Architecture](skills/architecture/01-tuist-modular-architecture.md) | How to define a `Project.swift` with Core/Feature module layers, enforce strict dependency boundaries at compile time, eliminate Xcode project merge conflicts, and structure a directory convention that scales to 10+ modules. Includes helper functions for `coreFramework()` and `featureFramework()` with automatic test target generation. |
| 02 | [Protocol-Driven Dependency Injection](skills/architecture/02-protocol-driven-dependency-injection.md) | A complete DI system in ~80 lines of Swift — no third-party frameworks. Three-layer architecture: Protocol (module-owned), DependencySpec (factory), and Registration (composition root). Thread-safe resolution via `NSLock`, `KeyPath`-based type-safe lookups, singleton vs. transient registration, and clean `reset()` for test isolation. |
| 03 | [UI Factory Pattern for Feature Modules](skills/architecture/03-ui-factory-pattern-for-feature-modules.md) | How Feature modules expose a single `UIFactory` that accepts pre-resolved dependencies via constructor injection and produces concrete SwiftUI Views (no `AnyView`). Covers factory variations for callbacks, runtime data, and the ViewModel-as-dependency-hub pattern. Eliminates service locator calls from Views entirely. |
| 06 | [Actor-Based Concurrency Patterns](skills/architecture/06-actor-based-concurrency-patterns.md) | When to use `actor` vs `@MainActor` vs `@unchecked Sendable` with `NSLock`. Covers actor-isolated data stores (CoreData), `nonisolated` escape hatches for `AsyncThrowingStream`, actor-to-actor communication chains, `Sendable` value-type domain models, and app lifecycle integration. Prevents data races at compile time. |
| 14 | [Typed Error System with Recovery Actions](skills/architecture/14-error-handling-and-typed-error-system.md) | A `WythnosError` enum where every case maps to a title, message, SF Symbol icon, and `RecoveryAction` — rendered through a reusable `NyxErrorCard` design system component. Includes AI safety classification (`SafetyClassification`), localized refusal responses in English and Turkish, and emergency crisis resource handling. |

### 🎨 [UI](skills/ui/) — 3 skills

Design system infrastructure, custom navigation, and multi-language support for SwiftUI apps that feel premium.

| # | Skill | What You'll Learn |
|---|-------|-------------------|
| 04 | [Design System as Core Module](skills/ui/04-design-system-as-core-module.md) | A four-layer design system: tokens (colors, typography, spacing, corner radii as static enums), reusable components (`NyxCard`, `NyxErrorCard`, `NyxFeedbackToggle`, `AutoSizingTextEditor`, `FlowLayout`), animation components (typed text, count-up numbers, slide-up/breathing-glow modifiers), and a centralized haptic feedback system with pre-initialized generators. Dark-mode-only, 4pt grid, OLED-optimized true black. |
| 13 | [Custom Tab Bar & Navigation](skills/ui/13-swiftui-custom-tab-bar-and-navigation.md) | Replacing `TabView` with a fully custom tab bar using `safeAreaInset(edge: .bottom)` (the correct API — not `overlay` or `ZStack`). Covers re-tap detection for scroll-to-top via `.id()` change, spring-physics `ButtonStyle`, breathing glow animations for the active tab glyph, keyboard avoidance with `.ignoresSafeArea(.keyboard)`, splash screen → content transitions, and onboarding → main app flow. |
| 16 | [Localization & Multi-Language Patterns](skills/ui/16-localization-and-multi-language-patterns.md) | Supporting 43 languages in a modular Tuist app where each framework owns its string catalogs via `bundle: .module`. Covers bilingual AI intent classification (English + Turkish keywords), locale-aware gene pools for AI personality, localized safety refusal responses for crisis situations, and test coverage for multi-language keyword detection. |

### 🧪 [Testing & Debugging](skills/testing-and-debugging/) — 2 skills

Quality assurance patterns that keep modular apps reliable across CI, simulators, and Xcode Previews.

| # | Skill | What You'll Learn |
|---|-------|-------------------|
| 05 | [Swift Testing & TDD Patterns](skills/testing-and-debugging/05-swift-testing-and-tdd-patterns.md) | Modern Swift Testing framework (`@Suite`, `@Test`, `#expect`) over XCTest for all unit tests. Test isolation via `UserDefaults(suiteName: #function)`, five categories of tests (service logic, DI container, AI engine, sanitizer/filter, actor-based async), mock override via `ServiceProvider.shared.register()`, UI test base class with launch arguments, and regression tests driven by real bug screenshots. |
| 09 | [Debug Modes & Mock Services](skills/testing-and-debugging/09-debug-modes-and-mock-service-strategy.md) | A three-tier mock system: Tier 1 (UI testing — minimal, fixed data), Tier 2 (rich debug — 30 days of realistic calendar patterns with Pro access), and Tier 3 (developer mode — secret gesture activation with SHA256-hashed codes for QA testers without Xcode). Shows how to override `ServiceProvider` registrations via launch arguments (`--uitesting`, `--pro-debug`) and re-register dependent services for graph consistency. |

### 📱 [Platform Frameworks](skills/platform-frameworks/) — 5 skills

Patterns for integrating Apple system frameworks (StoreKit, EventKit, WidgetKit, UNUserNotificationCenter) into modular apps.

| # | Skill | What You'll Learn |
|---|-------|-------------------|
| 07 | [StoreKit 2 Subscription System](skills/platform-frameworks/07-storekit2-intelligence-based-trial.md) | An intelligence-based trial that ends when the AI demonstrates value (50 interactions, 3 patterns discovered, or 21 days) — not a fixed free trial. Covers the `TrialManager` actor state machine (trial → grace → expired → subscribed), `DailyQueryTracker` with generous first-week limits (150 queries), StoreKit 2 async purchase/entitlement API, product ID conventions, feature gating, and trial funnel analytics. |
| 10 | [Privacy-First Analytics Architecture](skills/platform-frameworks/10-privacy-first-analytics-architecture.md) | A pluggable multi-backend analytics actor with typed event factory methods (no free-form strings). Covers the privacy filter (what is NEVER sent: calendar titles, locations, notes, user names), `nonisolated` fire-and-forget tracking from any context, buffer + flush for network efficiency, OSLog backend as zero-dependency fallback, and graceful TelemetryDeck configuration via Info.plist. |
| 11 | [Notification Service with Deep Linking](skills/platform-frameworks/11-notification-service-with-deep-linking.md) | An actor-based notification service that schedules LLM-generated local notifications (weekly summaries, meeting reminders, insight milestones). Deep linking to native video call apps (Zoom, Meet, Teams, Webex — 11 platforms) by extracting URLs from event locations and converting to native URL schemes. Covers engagement-gated permission requests, smart meeting importance scoring, and `NotificationPreferences` with `Codable` persistence. |
| 12 | [EventKit–CoreData Sync Architecture](skills/platform-frameworks/12-eventkit-coredata-sync-architecture.md) | A three-layer sync pipeline: `CalendarService` (EventKit abstraction) → `CalendarSyncManager` (orchestrator actor) → `CalendarStore` (CoreData actor). Covers programmatic CoreData model creation (no `.xcdatamodeld` — better for Tuist framework targets), upsert by ID, `NSMergeByPropertyObjectTrumpMergePolicy`, full sync (90 days back + 7 forward), incremental sync from last sync date, `Sendable` value-type domain models, and `inMemory: true` for fast tests. |
| 15 | [WidgetKit & App Intents Integration](skills/platform-frameworks/15-widgetkit-and-app-intents-integration.md) | Lightweight Home Screen widgets as a separate Tuist target with no framework dependencies. Covers `TimelineProvider` with placeholder/snapshot/timeline, multi-size widget views (`systemSmall` + `systemMedium`), `containerBackground(.black, for: .widget)` (iOS 17+), `AppIntent` for Siri Shortcuts, data sharing via App Groups, and 15-minute refresh cadence for battery efficiency. |

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
    │   ├── 02-protocol-driven-dependency-injection.md
    │   ├── 03-ui-factory-pattern-for-feature-modules.md
    │   ├── 06-actor-based-concurrency-patterns.md
    │   └── 14-error-handling-and-typed-error-system.md
    │
    ├── ui/                                ← 3 skills
    │   ├── README.md
    │   ├── 04-design-system-as-core-module.md
    │   ├── 13-swiftui-custom-tab-bar-and-navigation.md
    │   └── 16-localization-and-multi-language-patterns.md
    │
    ├── testing-and-debugging/             ← 2 skills
    │   ├── README.md
    │   ├── 05-swift-testing-and-tdd-patterns.md
    │   └── 09-debug-modes-and-mock-service-strategy.md
    │
    ├── platform-frameworks/               ← 5 skills
    │   ├── README.md
    │   ├── 07-storekit2-intelligence-based-trial.md
    │   ├── 10-privacy-first-analytics-architecture.md
    │   ├── 11-notification-service-with-deep-linking.md
    │   ├── 12-eventkit-coredata-sync-architecture.md
    │   └── 15-widgetkit-and-app-intents-integration.md
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
