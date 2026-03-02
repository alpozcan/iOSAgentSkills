# Architecture Skills

Foundational patterns for building modular, testable, and scalable iOS applications.

## Skills

Everything starts with [[01-tuist-modular-architecture]]. It defines the two-layer module hierarchy — Core frameworks that own protocols and domain logic, Feature frameworks that depend on Core and expose UI through composers. The App target sits at the top as the composition root where all dependencies are wired together.

That wiring happens through [[02-protocol-driven-service-catalog]], a ~80-line service catalog that uses `KeyPath`-based resolution and `NSLock` for thread safety. Each Core module declares a protocol, each Feature module declares a `Blueprint` factory, and the App target registers everything at launch.

Feature modules expose their UI through [[03-ui-composer-pattern-for-feature-modules]]. A composer accepts pre-resolved protocol dependencies via constructor injection and produces concrete SwiftUI Views — no `AnyView`, no service locator calls from inside Views. This is the public API boundary of every Feature module.

## Concurrency & Safety

[[06-actor-based-concurrency-patterns]] defines the concurrency rules every service must follow: `actor` for mutable data stores, `@MainActor` for ViewModels, `@unchecked Sendable` with `NSLock` for lightweight containers like `Catalog`. These rules ensure the catalog from [[02-protocol-driven-service-catalog]] and the data stores from [[12-eventkit-coredata-sync-architecture]] are thread-safe at compile time.

[[14-error-handling-and-typed-error-system]] provides typed errors that carry recovery actions, SF Symbol icons, and localized messages. Error types conform to `Sendable` so they can cross actor boundaries safely — a direct requirement of [[06-actor-based-concurrency-patterns]]. The error card component comes from [[04-design-system-as-core-module]].

## Project Orchestration

[[18-makefile-for-ios-project-workflows]] wraps the Tuist lifecycle, build commands, test execution, and App Store submission into self-documenting `make` targets. It depends on the project structure from [[01-tuist-modular-architecture]] and feeds into the publishing pipeline defined in [[20-fastlane-app-store-connect-publishing]].

## How They Connect

```
┌─────────────────────────────────────────────┐
│                   App Layer                  │
│  (Composition Root, Catalog Preparation — #02)  │
├──────────┬──────────┬───────────────────────┤
│ Feature A│ Feature B│  Feature C            │
│ (UI Factory — #03)  │  (UI Factory — #03)   │
├──────────┴──────────┴───────────────────────┤
│              Core Services                   │
│  (Actors — #06, Typed Errors — #14)         │
├─────────────────────────────────────────────┤
│           Tuist Module Graph (#01)           │
└─────────────────────────────────────────────┘
```
