# EventKit to CoreData Sync Architecture

## Context
Your app reads calendar events from the system (EventKit) and needs to make them available for AI analysis, offline querying, and widget display. You need a sync layer that handles full sync, incremental sync, and background refresh — storing events in CoreData for fast local access.

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
            if let error { fatalError("CalendarStore failed: \(error)") }
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
        let startDate = calendar.date(byAdding: .day, value: -90, to: Date())!
        let endDate = calendar.date(byAdding: .day, value: 7, to: Date())!
        
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
        let startDate = min(lastSync, Calendar.current.date(byAdding: .day, value: -7, to: Date())!)
        let endDate = Calendar.current.date(byAdding: .day, value: 7, to: Date())!
        
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

## Why This Matters

- **Actor isolation** for CoreData prevents threading crashes — `newBackgroundContext()` + `perform` is the correct pattern
- **Programmatic CoreData model** avoids `.xcdatamodeld` — better for modular Tuist projects where models live in framework targets
- **Upsert by ID** prevents duplicate events when syncing
- **`inMemory: true` constructor** enables fast, isolated unit tests without file system side effects
- **Incremental sync** is O(recent events) instead of O(all events), critical for large calendars
- **Pure value-type domain model** (`CalendarEvent`) crosses actor boundaries safely

## Anti-Patterns

- Don't use `viewContext` for writes — always use `newBackgroundContext()` for save operations
- Don't fetch EKEvents on the main thread — EventKit operations should be `async`
- Don't store `EKEvent` objects — convert to your `CalendarEvent` value type immediately
- Don't skip `mergePolicy` — `NSMergeByPropertyObjectTrumpMergePolicy` handles concurrent upserts correctly
- Don't use `.xcdatamodeld` in framework targets — use programmatic model creation
