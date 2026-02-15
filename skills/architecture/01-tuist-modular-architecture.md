# Tuist-Based Modular iOS Architecture

## Context
When building a production iOS app with 10+ modules, you need a declarative, reproducible build system that enforces strict dependency boundaries and eliminates Xcode project merge conflicts.

## Pattern

Use **Tuist** with a single `Project.swift` that defines **Core** and **Feature** module layers using helper functions to enforce naming conventions, test target generation, and dependency declarations.

### Module Hierarchy

```
Core Modules (frameworks)     → No feature dependencies, only ServiceProvider + other Core
Feature Modules (frameworks)  → Depend on Core modules, never on other Features
App Target                    → Depends on everything, wires DI
Widget / Intent Extensions    → Lightweight, share data via App Groups
```

### Helper Functions for Module Declaration

```swift
func coreFramework(
    name: String,
    dependencies: [TargetDependency] = [],
    resources: ResourceFileElements? = nil,
    hasTests: Bool = true
) -> [Target] {
    var targets: [Target] = [
        .target(
            name: name,
            destinations: destinations,
            product: .framework,
            bundleId: "app.myapp.ios.Core.\(name)",
            deploymentTargets: deploymentTargets,
            infoPlist: .default,
            sources: ["Modules/Core/\(name)/Sources/**"],
            resources: resources,
            dependencies: dependencies
        )
    ]
    if hasTests {
        targets.append(
            .target(
                name: "\(name)Tests",
                destinations: destinations,
                product: .unitTests,
                bundleId: "app.myapp.ios.Core.\(name).Tests",
                deploymentTargets: deploymentTargets,
                infoPlist: .default,
                sources: ["Modules/Core/\(name)/Tests/**"],
                dependencies: [.target(name: name)]
            )
        )
    }
    return targets
}
```

### Directory Convention

```
Modules/
├── Core/
│   ├── ServiceProvider/
│   │   ├── Sources/
│   │   │   ├── ServiceProvider.swift
│   │   │   └── DependencySpec.swift
│   │   └── Tests/
│   │       └── ServiceProviderTests.swift
│   ├── CalendarService/
│   │   ├── Sources/
│   │   └── Tests/
│   └── DesignSystem/
│       ├── Sources/
│       ├── Resources/
│       └── Tests/
├── Features/
│   ├── Chat/
│   │   ├── Sources/
│   │   │   ├── ChatView.swift
│   │   │   ├── ChatViewModel.swift
│   │   │   └── ChatUIFactory.swift
│   │   ├── Resources/
│   │   └── Tests/
│   └── Settings/
│       ├── Sources/
│       └── Tests/
```

### Key Rules

1. **Every module gets a test target by default** (`hasTests: true`)
2. **Feature modules name their internal directory differently** from the framework name (e.g., framework `ChatUI`, directory `Chat`) — this is handled by the `moduleName` parameter in `featureFramework()`
3. **External dependencies are declared once** and flow through the dependency graph — only `AnalyticsService` depends on `TelemetryDeck`; no other module needs to
4. **The app target is the only target that depends on ALL modules** — it serves as the composition root

### Dependency Graph Pattern

```swift
let coreTargets: [Target] =
    coreFramework(name: "ServiceProvider")  // No dependencies — leaf module
    + coreFramework(name: "CalendarService", dependencies: [.target(name: "ServiceProvider")])
    + coreFramework(name: "CalendarStore", dependencies: [
        .target(name: "ServiceProvider"),
        .target(name: "CalendarService")
    ])
    + coreFramework(name: "LLMEngine", dependencies: [
        .target(name: "ServiceProvider"),
        .target(name: "CalendarService")
    ])
    + coreFramework(name: "DesignSystem", dependencies: [
        .target(name: "CalendarService")
    ], resources: [.glob(pattern: "Modules/Core/DesignSystem/Resources/**")])
```

## Why This Matters

- **No Xcode project file in Git** — Tuist generates it from `Project.swift`, eliminating merge conflicts entirely
- **Compile-time dependency enforcement** — if `ChatUI` tries to import `SettingsUI`, the build fails
- **Parallel builds** — independent modules compile concurrently
- **Test isolation** — each module's tests only depend on that module, preventing test pollution
- **Onboarding speed** — `tuist generate` gives any engineer a working project in seconds

## Anti-Patterns

- Don't create circular dependencies between Core modules
- Don't let Feature modules depend on other Feature modules
- Don't put shared model types in Feature modules — they belong in Core
- Don't skip test targets to "save time" — the `hasTests` flag exists for modules genuinely without testable logic (rare)
