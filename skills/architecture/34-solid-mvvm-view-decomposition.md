---
title: "SOLID Patterns, MVVM Enforcement, and View Decomposition"
description: "Enforce SOLID principles in SwiftUI MVVM apps: single-responsibility ViewModels, protocol-driven dependencies, reusable views under 200-300 LOC, and systematic decomposition strategies."
---

# SOLID Patterns, MVVM Enforcement, and View Decomposition

## Context
SwiftUI apps grow organically. ViewModels accumulate responsibilities, views exceed 500+ LOC, and services become tightly coupled. This skill provides concrete patterns for enforcing SOLID principles, maintaining clean MVVM boundaries, and keeping views decomposed and reusable.

## SOLID in SwiftUI MVVM

### S — Single Responsibility

**ViewModels own exactly one screen's state.** If a ViewModel serves multiple screens, split it.

```swift
// BAD: God ViewModel
@Observable
final class MapViewModel {
    // Map state, search state, timeline state, bookmarks, filters,
    // measurement, clustering, review prompts...
    // 200+ lines, 9 services
}

// GOOD: One ViewModel per screen concern
@Observable final class MapViewModel { ... }        // Map display + selection
@Observable final class SearchViewModel { ... }     // Search query + results
@Observable final class TimelineViewModel { ... }   // Timeline playback + quest selection
@Observable final class BookmarksViewModel { ... }  // Bookmark CRUD + navigation
```

**Rule of thumb:** If a ViewModel has more than 3-4 injected services, it has too many responsibilities.

**Services own exactly one domain.** A `FilterService` filters. A `BookmarkService` persists bookmarks. A `LocationDataService` provides location data. They never reach into each other's domains.

### O — Open/Closed

Extend behavior through protocols and composition, not by modifying existing types.

```swift
// Protocol allows new data sources without modifying FilterService
protocol FilterableDataSource {
    var allMapItems: [MapItem] { get }
    func filtered(by state: FilterState) -> [MapItem]
}

// Each data service conforms independently
extension LocationDataService: FilterableDataSource { ... }
extension EventDataService: FilterableDataSource { ... }
```

### L — Liskov Substitution

All protocol conformances must be fully substitutable. Test with mocks that exercise edge cases.

```swift
// Protocol contract
protocol LocationDataServiceProtocol: Sendable {
    var allLocations: [LocationMarker] { get }
    func location(byId id: String) -> LocationMarker?
}

// Mock must behave identically for all callers
struct MockLocationDataService: LocationDataServiceProtocol {
    var allLocations: [LocationMarker] = []
    func location(byId id: String) -> LocationMarker? {
        allLocations.first { $0.id == id }
    }
}
```

### I — Interface Segregation

Don't force consumers to depend on methods they don't use.

```swift
// BAD: One fat protocol
protocol MapServiceProtocol {
    var allLocations: [LocationMarker] { get }
    var allEvents: [EventMarker] { get }
    var allPaths: [CharacterPath] { get }
    func filter(...) -> [MapItem]
    func search(...) -> [SearchResult]
    func bookmark(...) -> Bookmark
}

// GOOD: Segregated interfaces
protocol LocationProviding: Sendable {
    var allLocations: [LocationMarker] { get }
}
protocol EventProviding: Sendable {
    var allEvents: [EventMarker] { get }
}
protocol Searchable {
    func search(query: String) -> [SearchResult]
}
```

### D — Dependency Inversion

ViewModels depend on protocols, not concrete types. Inject at the composition root.

```swift
@Observable
final class SearchViewModel {
    private let locationService: LocationProviding
    private let eventService: EventProviding

    init(locationService: LocationProviding, eventService: EventProviding) {
        self.locationService = locationService
        self.eventService = eventService
    }
}
```

## MVVM Enforcement Rules

### Layer Responsibilities

| Layer | Owns | Never Does |
|-------|------|------------|
| **View** | Layout, styling, user gestures, navigation triggers | Business logic, data fetching, state mutation beyond @State |
| **ViewModel** | Screen state, user action handling, data transformation for display | Direct UI code (Color, Font), navigation presentation |
| **Model** | Domain types, validation rules | UI or ViewModel references |
| **Service** | Data access, persistence, external APIs | UI state, ViewModel references |

### ViewModel Patterns

```swift
// ViewModel exposes display-ready state, not raw model data
@Observable
final class LocationDetailViewModel {
    // Display state
    var title: String = ""
    var subtitle: String = ""
    var significance: String = ""
    var isBookmarked: Bool = false

    // Actions
    func toggleBookmark() { ... }
    func shareLocation() -> ShareContent { ... }

    // Private services
    private let bookmarkService: BookmarkServiceProtocol
    private let location: LocationMarker
}
```

### What Stays in the View

- `@State` for transient UI state (scroll position, sheet presentation, animation flags)
- `@Environment` reads
- Layout composition
- Gesture handlers that call ViewModel actions

## View Decomposition: The 200-300 LOC Rule

### When to Split

A view MUST be split when:
1. It exceeds **300 LOC** (hard limit)
2. It has **3+ levels of nesting** in the body
3. It contains **2+ distinct visual sections** that could be reused
4. It has **complex conditional rendering** (3+ branches)

### Decomposition Strategies

**Strategy 1: Extract computed properties first**
```swift
// Before: 400 LOC view with inline styling
var body: some View {
    VStack {
        // 50 lines of header
        // 80 lines of content
        // 40 lines of footer
    }
}

// After: Computed properties for sections
var body: some View {
    VStack {
        headerSection
        contentSection
        footerSection
    }
}

private var headerSection: some View { ... }  // 50 LOC
private var contentSection: some View { ... } // 80 LOC
private var footerSection: some View { ... }  // 40 LOC
```

**Strategy 2: Extract reusable components**
```swift
// When a section appears in 2+ places, extract to its own struct
struct MEEventCard: View {
    let event: TimelineEvent
    let onTap: () -> Void

    var body: some View { ... }  // Self-contained, reusable
}
```

**Strategy 3: Extract subviews with bindings for interactive sections**
```swift
struct FilterChipRow: View {
    @Binding var activeFilters: Set<Race>
    let allRaces: [Race]

    var body: some View { ... }
}
```

### Naming Conventions

| Type | Pattern | Example |
|------|---------|---------|
| Screen-level view | `{Feature}View` | `TimelineView`, `SearchView` |
| Tab wrapper | `{Feature}TabView` | `TimelineTabView`, `MapTabView` |
| Reusable component | `ME{Component}` | `MEEventCard`, `MEFilterChip` |
| Detail view | `{Model}DetailView` | `LocationDetailView`, `EventDetailView` |
| Section (private) | `{description}Section` (computed property) | `headerSection`, `questSelector` |

### File Organization

```
Views/
  Tabs/           # Tab-level containers (< 100 LOC each)
  Timeline/       # Feature views
    TimelineView.swift         # Main view (< 300 LOC)
    TimelineEventCard.swift    # Extracted card component
    TimelineQuestSelector.swift # Extracted selector
  Detail/         # Detail views
  Common/         # Shared components (ME-prefixed)
  Navigation/     # Tab bar, routing
```

## Audit Checklist

When refactoring an existing codebase, check each file against:

- [ ] **ViewModel LOC:** No ViewModel exceeds 150 LOC (extract helpers/services)
- [ ] **View LOC:** No view exceeds 300 LOC (decompose into sections/subviews)
- [ ] **Service count per ViewModel:** Max 3-4 injected services
- [ ] **Protocol coverage:** Every service has a protocol; ViewModels depend on protocols
- [ ] **No cross-layer leaks:** Views don't do business logic; ViewModels don't reference UIKit/SwiftUI types
- [ ] **Data service instantiation:** Shared instances, not duplicated per consumer
- [ ] **Naming consistency:** Follows ME-prefix for design system, {Feature}View for screens

## Connections

- [[01-tuist-modular-architecture]] — module boundaries enforce SOLID at the package level
- [[02-protocol-driven-service-catalog]] — service catalog is the composition root for dependency inversion
- [[03-ui-composer-pattern-for-feature-modules]] — composers are the MVVM wiring layer
- [[06-actor-based-concurrency-patterns]] — actor isolation rules apply to service layer
- [[24-swiftdata-and-observation-framework-patterns]] — @Observable is the ViewModel backbone
