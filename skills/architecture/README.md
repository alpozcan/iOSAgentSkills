# Architecture — Map of Content

The structural backbone of a modular iOS app. These skills define how code is organized into modules, how dependencies flow between them, how services communicate safely across threads, and how the entire project is orchestrated from a single entry point.

## The Module Foundation

Everything starts with [[01-tuist-modular-architecture]]. It defines the two-layer module hierarchy — Core frameworks that own protocols and domain logic, Feature frameworks that depend on Core and expose UI through factories. The App target sits at the top as the composition root where all dependencies are wired together.

That wiring happens through [[02-protocol-driven-service-catalog]], a ~80-line catalog that uses `KeyPath`-based resolution and `NSLock` for thread safety. Each Core module declares a protocol, each Feature module declares a `Blueprint` factory, and the App target registers everything at launch.

Feature modules expose their UI through [[03-ui-composer-pattern-for-feature-modules]]. A factory accepts pre-resolved protocol dependencies via constructor injection and produces concrete SwiftUI Views — no `AnyView`, no service locator calls from inside Views. This is the public API boundary of every Feature module.

## Concurrency & Safety

[[06-actor-based-concurrency-patterns]] defines the concurrency rules every service must follow: `actor` for mutable data stores, `@MainActor` for ViewModels, `@unchecked Sendable` with `NSLock` for lightweight containers like `Catalog`. These rules ensure the catalog from [[02-protocol-driven-service-catalog]] and the data stores from [[12-eventkit-coredata-sync-architecture]] are thread-safe at compile time.

[[14-error-handling-and-typed-error-system]] provides typed errors that carry recovery actions, SF Symbol icons, and localized messages. Error types conform to `Sendable` so they can cross actor boundaries safely — a direct requirement of [[06-actor-based-concurrency-patterns]]. The error card component comes from [[04-design-system-as-core-module]].

## Project Orchestration

[[18-makefile-for-ios-project-workflows]] wraps the Tuist lifecycle, build commands, test execution, and App Store submission into self-documenting `make` targets. It depends on the project structure from [[01-tuist-modular-architecture]] and feeds into the publishing pipeline defined in [[20-fastlane-app-store-connect-publishing]].

## How They Connect

```
01 Tuist Modular Architecture
 ├── 02 Service Catalog (composition root lives in App target)
 │    ├── 03 UI Composer Pattern (composers are Blueprints)
 │    └── 06 Actor Concurrency (Catalog is @unchecked Sendable)
 ├── 14 Error Handling (Sendable errors cross actor boundaries)
 └── 18 Makefile (orchestrates the Tuist lifecycle)
```
