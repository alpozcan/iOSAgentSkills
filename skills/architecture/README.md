# Architecture Skills

Foundational patterns for building modular, testable, and scalable iOS applications.

## Skills

- [01 — Tuist-Based Modular Architecture](01-tuist-modular-architecture.md) — Declarative build system with Core/Feature module layers
- [02 — Protocol-Driven Dependency Injection](02-protocol-driven-dependency-injection.md) — Compile-time safe DI without third-party frameworks
- [03 — UI Factory Pattern](03-ui-factory-pattern-for-feature-modules.md) — Constructor-injected factories for Feature module composition
- [06 — Actor-Based Concurrency](06-actor-based-concurrency-patterns.md) — Thread-safe services using Swift actors
- [14 — Typed Error System](14-error-handling-and-typed-error-system.md) — Structured errors with recovery actions
- [18 — Makefile for iOS Project Workflows](18-makefile-for-ios-project-workflows.md) — Single entry point for build, run, test, and snapshot workflows

## How These Fit Together

```
┌─────────────────────────────────────────────┐
│                   App Layer                  │
│  (Composition Root, DI Registration — #02)  │
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
