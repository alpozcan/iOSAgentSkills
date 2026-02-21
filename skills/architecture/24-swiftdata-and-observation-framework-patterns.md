---
title: "SwiftData & Observation Framework Patterns"
description: "How to use @Model, @Observable, @Query, and @ModelActor in a modular Tuist iOS app. Covers persistent model design, actor-isolated queries, CoreData migration, and replacing ObservableObject with @Observable."
---

# SwiftData & Observation Framework Patterns

## Context

SwiftData (iOS 17+) replaces CoreData's boilerplate with `@Model` macros, while `@Observable` (also iOS 17+) replaces `ObservableObject`/`@Published`. In a modular Tuist app (see [[01-tuist-modular-architecture]]), these frameworks simplify persistence and state management but require careful actor isolation and module boundary design.

## Pattern

### @Model Macro Basics

Define persistent models with `@Model`. The macro generates `PersistentModel` conformance, change tracking, and property storage automatically:

```swift
@Model
final class CalendarEventModel {
    @Attribute(.unique) var eventIdentifier: String
    var title: String
    var startDate: Date
    var endDate: Date
    var calendarName: String

    @Relationship(deleteRule: .cascade)
    var reminders: [ReminderModel]

    init(eventIdentifier: String, title: String, startDate: Date, endDate: Date, calendarName: String) {
        self.eventIdentifier = eventIdentifier
        self.title = title
        self.startDate = startDate
        self.endDate = endDate
        self.calendarName = calendarName
    }
}
```

### ModelContainer in Tuist Core Modules

The container lives in a Core persistence module, injected via the catalog (see [[02-protocol-driven-service-catalog]]):

```swift
// In PersistenceService (Core module)
public protocol PersistenceServiceProtocol: Sendable {
    var modelContainer: ModelContainer { get }
}

public final class PersistenceService: PersistenceServiceProtocol, Sendable {
    public let modelContainer: ModelContainer

    public init(inMemory: Bool = false) throws {
        let schema = Schema([CalendarEventModel.self, ReminderModel.self])
        let config = ModelConfiguration(isStoredInMemoryOnly: inMemory)
        self.modelContainer = try ModelContainer(for: schema, configurations: [config])
    }
}
```

For tests, use `inMemory: true` to get a fast, isolated store with no disk I/O:

```swift
let testService = try PersistenceService(inMemory: true)
```

### Actor-Isolated Queries

SwiftData's `ModelContext` is not `Sendable`. Use `@ModelActor` for background work — it creates its own `ModelContext` bound to the actor (see [[06-actor-based-concurrency-patterns]] for actor isolation rules):

```swift
@ModelActor
actor CalendarDataStore {
    func upsertEvents(_ events: [CalendarEvent]) throws {
        for event in events {
            let descriptor = FetchDescriptor<CalendarEventModel>(
                predicate: #Predicate { $0.eventIdentifier == event.id }
            )
            if let existing = try modelContext.fetch(descriptor).first {
                existing.title = event.title
                existing.startDate = event.startDate
            } else {
                let model = CalendarEventModel(from: event)
                modelContext.insert(model)
            }
        }
        try modelContext.save()
    }
}
```

The `@ModelActor` macro synthesizes an `init(modelContainer:)` initializer and a `modelContext` property, so you never need to pass a context across actor boundaries.

### CoreData to SwiftData Migration

Gradual migration strategy for apps currently using CoreData (see [[12-eventkit-coredata-sync-architecture]]):

```swift
// Stage 1: Map existing NSManagedObject to SwiftData schema
let schema = Schema([CalendarEventModel.self])
let config = ModelConfiguration(url: existingStoreURL)

// Stage 2: Use SchemaMigrationPlan for versioned migrations
enum MigrationPlan: SchemaMigrationPlan {
    static var schemas: [any VersionedSchema.Type] { [SchemaV1.self, SchemaV2.self] }
    static var stages: [MigrationStage] { [migrateV1toV2] }

    static let migrateV1toV2 = MigrationStage.lightweight(fromVersion: SchemaV1.self, toVersion: SchemaV2.self)
}
```

SwiftData and CoreData can coexist against the same underlying store file as long as the schemas remain compatible. This allows incremental migration — one model at a time — rather than a risky big-bang rewrite.

### @Observable Replacing ObservableObject

`@Observable` (iOS 17+) replaces `ObservableObject`/`@Published` with fine-grained observation:

```swift
// OLD (ObservableObject)
class ChatViewModel: ObservableObject {
    @Published var messages: [ChatMessage] = []
    @Published var isLoading: Bool = false
}
// Usage: @StateObject var vm = ChatViewModel()

// NEW (@Observable)
@Observable
final class ChatViewModel {
    var messages: [ChatMessage] = []
    var isLoading: Bool = false
}
// Usage: @State var vm = ChatViewModel()
```

Key difference: `@Observable` tracks property access at the view level — only views that actually read `messages` re-render when `messages` changes. No `@Published` wrappers needed. SwiftUI uses the `withObservationTracking` API under the hood to detect which properties each view reads.

### @Query for SwiftUI Views

`@Query` brings database queries directly into views, replacing `NSFetchedResultsController` and manual `@FetchRequest`:

```swift
struct EventListView: View {
    @Query(filter: #Predicate<CalendarEventModel> { $0.startDate > Date() },
           sort: \.startDate)
    private var upcomingEvents: [CalendarEventModel]

    var body: some View {
        List(upcomingEvents) { event in
            EventRow(event: event)
        }
    }
}
```

`@Query` automatically re-evaluates whenever `ModelContext.save()` is called, keeping the view in sync with the database.

## Edge Cases

- `@Model` classes must be `final` — the macro generates conformances that don't work with inheritance
- `#Predicate` doesn't support all Swift expressions — complex filtering must be done post-fetch on the returned array
- `ModelContext` is not `Sendable` — never pass it across actor boundaries. Use `@ModelActor` instead
- SwiftData and CoreData can coexist using the same underlying store file — but the schema must be compatible between both frameworks
- `@Query` re-evaluates on every `ModelContext.save()` — frequent saves cause excessive view updates. Batch writes into a single `save()` call
- CloudKit sync with SwiftData requires `ModelConfiguration(cloudKitDatabase:)` — but conflicts are harder to resolve than with CoreData's `NSMergePolicy`
- Undo/redo: `ModelContext` supports `undoManager` but it's opt-in and must be configured at container creation
- `@Attribute(.unique)` violations throw at save time — always handle with `do/catch`, not force-try (see [[14-error-handling-and-typed-error-system]])

## Why This Matters

- SwiftData eliminates ~60% of CoreData boilerplate — no `.xcdatamodeld` files, no `NSFetchRequest`, no `NSManagedObjectContext` threading rules
- `@Observable` reduces unnecessary view re-renders — only properties that a view actually accesses trigger updates, unlike `ObservableObject` which notifies on any `@Published` change
- `@ModelActor` provides compile-time thread safety that CoreData's `perform {}` blocks can't guarantee
- In-memory containers (`isStoredInMemoryOnly: true`) make tests fast and isolated with zero disk I/O

## Anti-Patterns

- Don't use `@Model` on structs — it only works on classes (the macro adds reference semantics for change tracking)
- Don't pass `ModelContext` between actors — use `@ModelActor` or create a new context from the container
- Don't mix `@Observable` with `ObservableObject` in the same view hierarchy — pick one pattern per feature module
- Don't use `@Query` in ViewModels — it's designed for Views. Use `ModelContext.fetch()` in ViewModels
- Don't skip the migration plan when changing your schema — data loss is silent without `SchemaMigrationPlan`
- Don't use `@Observable` for types that need to be `Sendable` across actors — `@Observable` uses reference semantics and is not `Sendable`
