---
title: "EventKit to CoreData Sync Architecture"
description: "Three-layer sync pipeline from EventKit to CoreData: CalendarService abstraction, CalendarSyncManager orchestrator, CalendarStore actor. Programmatic CoreData model, upsert by ID, and in-memory stores for tests."
---

# EventKit to CoreData Sync Architecture

## Context
Your app reads calendar events from the system (EventKit) and needs to make them available for AI analysis, offline querying, and [[15-widgetkit-and-app-intents-integration|widget]] display. You need a sync layer that handles full sync, incremental sync, and background refresh — storing events in CoreData for fast local access. This data also powers [[11-notification-service-with-deep-linking|notifications]].

## Pattern

### Three-Layer Architecture

```
EventKit (System) → CalendarService (Protocol) → CalendarSyncManager (Actor) → CalendarStore (Actor + CoreData)
```

### Layer 1: CalendarService (EventKit Abstraction)

```swift
public protocol CalendarServiceProtocol: Sendable {
    func authorizationStatus() -> CalendarAuthStatus
    func requestAccess() async -> Bool
    func fetchEvents(from startDate: Date, to endDate: Date) async throws -> [CalendarEvent]
}

public final class CalendarService: CalendarServiceProtocol, @unchecked Sendable {
    private let eventStore: EKEventStore
    
    public func requestAccess() async -> Bool {
        do { return try await eventStore.requestFullAccessToEvents() }
        catch { return false }
    }
    
    public func fetchEvents(from startDate: Date, to endDate: Date) async throws -> [CalendarEvent] {
        guard authorizationStatus() == .authorized else { return [] }
        let predicate = eventStore.predicateForEvents(withStart: startDate, end: endDate, calendars: nil)
        return eventStore.events(matching: predicate).map { event in
            CalendarEvent(
                id: event.eventIdentifier ?? UUID().uuidString,
                title: event.title ?? "Untitled",
                startDate: event.startDate,
                endDate: event.endDate,
                location: event.location,
                notes: event.notes,
                calendarName: event.calendar?.title ?? "Unknown",
                isAllDay: event.isAllDay
            )
        }
    }
}
```

### Layer 2: CalendarStore (CoreData Persistence via Actor)

**Programmatic CoreData Model (no .xcdatamodeld file):**

```swift
public actor CalendarStore: CalendarStoreProtocol {
    private let container: NSPersistentContainer
    
    public init(inMemory: Bool = false) {
        let model = Self.createModel()
        container = NSPersistentContainer(name: "CalendarStore", managedObjectModel: model)
        if inMemory {
            let desc = NSPersistentStoreDescription()
            desc.type = NSInMemoryStoreType
            container.persistentStoreDescriptions = [desc]
        }
        container.loadPersistentStores { _, error in
            if let error {
                // Log critical failure — recovery handled by caller
                Logger(subsystem: "app.wythnos.ios", category: "persistence")
                    .fault("CalendarStore failed to load: \(error.localizedDescription, privacy: .public)")
            }
        }
        container.viewContext.mergePolicy = NSMergeByPropertyObjectTrumpMergePolicy
    }
    
    private static func createModel() -> NSManagedObjectModel {
        let model = NSManagedObjectModel()
        
        let eventEntity = NSEntityDescription()
        eventEntity.name = "CalendarEventEntity"
        // Programmatically define: id, title, startDate, endDate, location, notes, calendarName, isAllDay
        
        let metaEntity = NSEntityDescription()
        metaEntity.name = "SyncMetadataEntity"
        // key (String), dateValue (Date?)
        
        model.entities = [eventEntity, metaEntity]
        return model
    }
    
    // Upsert pattern — fetch-or-create by ID
    public func saveEvents(_ events: [CalendarEvent]) async throws {
        let context = container.newBackgroundContext()
        context.mergePolicy = NSMergeByPropertyObjectTrumpMergePolicy
        try await context.perform {
            for event in events {
                let request = NSFetchRequest<CalendarEventEntity>(entityName: "CalendarEventEntity")
                request.predicate = NSPredicate(format: "id == %@", event.id)
                let existing = try context.fetch(request).first ?? CalendarEventEntity(context: context)
                existing.id = event.id
                existing.title = event.title
                // ... map all fields
            }
            try context.save()
        }
    }
}
```

### Layer 3: CalendarSyncManager (Orchestrator)

```swift
public actor CalendarSyncManager {
    public enum SyncState: Sendable {
        case idle
        case syncing(progress: String)
        case completed(eventCount: Int)
        case failed(Error)
    }
    
    private let calendarService: CalendarServiceProtocol
    private let store: CalendarStoreProtocol
    
    // Full sync: 90 days back + 7 days forward
    public func sync() async -> SyncState {
        let calendar = Calendar.current
        guard let startDate = calendar.date(byAdding: .day, value: -90, to: Date()),
              let endDate = calendar.date(byAdding: .day, value: 7, to: Date()) else {
            return .failed(CalendarSyncError.dateCalculationFailed)
        }
        
        do {
            let events = try await calendarService.fetchEvents(from: startDate, to: endDate)
            try await store.saveEvents(events)
            await store.setLastSyncDate(Date())
            return .completed(eventCount: events.count)
        } catch {
            return .failed(error)
        }
    }
    
    // Incremental sync: since last sync + rolling 7-day window
    public func incrementalSync() async -> SyncState {
        let lastSync = await store.lastSyncDate() ?? .distantPast
        guard let weekAgo = Calendar.current.date(byAdding: .day, value: -7, to: Date()),
              let endDate = Calendar.current.date(byAdding: .day, value: 7, to: Date()) else {
            return .failed(CalendarSyncError.dateCalculationFailed)
        }
        let startDate = min(lastSync, weekAgo)
        
        do {
            let events = try await calendarService.fetchEvents(from: startDate, to: endDate)
            try await store.saveEvents(events)
            await store.setLastSyncDate(Date())
            return .completed(eventCount: events.count)
        } catch {
            return .failed(error)
        }
    }
}
```

### Sendable Domain Model (Pure Value Type)

```swift
public struct CalendarEvent: Sendable, Identifiable, Hashable {
    public let id: String
    public let title: String
    public let startDate: Date
    public let endDate: Date
    public let location: String?
    public let notes: String?
    public let calendarName: String
    public let isAllDay: Bool
    
    public var durationMinutes: Int {
        Int(endDate.timeIntervalSince(startDate) / 60)
    }
}
```

### Testing with In-Memory Store

```swift
// Create an in-memory store for tests
let store = CalendarStore(inMemory: true)
try await store.saveEvents(mockEvents)
let fetched = try await store.fetchEvents(from: startDate, to: endDate)
#expect(fetched.count == mockEvents.count)
```

## Edge Cases

- **Permission revoked mid-sync:** The user can revoke calendar access in Settings while a sync is in progress. `fetchEvents()` returns an empty array when unauthorized — handle this gracefully by checking `authorizationStatus()` before and after fetch, and notify the user via [[14-error-handling-and-typed-error-system|typed errors]].
- **Large calendar (10K+ events):** Full sync of 90 days can return thousands of events. Batch `saveEvents()` into chunks of 500 to avoid CoreData memory pressure. Use `autoreleasepool` inside the batch loop.
- **Concurrent sync guard:** Multiple sync triggers (app foreground + widget refresh + background fetch) can race. Add an `isSyncing` flag to `CalendarSyncManager` and return `.idle` if a sync is already in progress.
- **Nil `eventIdentifier`:** `EKEvent.eventIdentifier` can be nil for events that haven't been saved to the calendar store. The current code falls back to `UUID().uuidString`, which creates duplicates on re-sync. Consider using a composite key (title + startDate + calendar) for deduplication when `eventIdentifier` is nil.
- **CoreData merge conflicts:** When the main app and widget extension write to the same persistent store simultaneously, merge conflicts occur. `NSMergeByPropertyObjectTrumpMergePolicy` resolves this by favoring the latest write, but log conflicts for debugging.
- **NSPredicate injection:** The `NSPredicate(format: "id == %@", event.id)` pattern uses format string substitution which is safe. Never use string interpolation like `NSPredicate(format: "id == '\(event.id)'")` — this allows predicate injection if `event.id` contains special characters.

## Why This Matters

- **[[06-actor-based-concurrency-patterns|Actor isolation]]** for CoreData prevents threading crashes — `newBackgroundContext()` + `perform` is the correct pattern
- **Programmatic CoreData model** avoids `.xcdatamodeld` — better for modular [[01-tuist-modular-architecture|Tuist]] projects where models live in framework targets
- **Upsert by ID** prevents duplicate events when syncing
- **`inMemory: true` constructor** enables fast, isolated [[05-swift-testing-and-tdd-patterns|unit tests]] without file system side effects
- **Incremental sync** is O(recent events) instead of O(all events), critical for large calendars
- **Pure value-type domain model** (`CalendarEvent`) crosses actor boundaries safely

## Anti-Patterns

- Don't use `viewContext` for writes — always use `newBackgroundContext()` for save operations
- Don't fetch EKEvents on the main thread — EventKit operations should be `async`
- Don't store `EKEvent` objects — convert to your `CalendarEvent` value type immediately
- Don't skip `mergePolicy` — `NSMergeByPropertyObjectTrumpMergePolicy` handles concurrent upserts correctly
- Don't use `.xcdatamodeld` in framework targets — use programmatic model creation
- Don't use string interpolation in NSPredicate format strings — always use `%@` substitution to prevent predicate injection
- Don't sync without checking `authorizationStatus()` first — permission can be revoked at any time
