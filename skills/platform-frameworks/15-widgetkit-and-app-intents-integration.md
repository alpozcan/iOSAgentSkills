---
title: "WidgetKit and App Intents Integration"
description: "Lightweight Home Screen widgets as a separate Tuist target with TimelineProvider, multi-size views, App Group data sharing, AppIntent for Siri Shortcuts, and 15-minute refresh cadence."
---

# WidgetKit and App Intents Integration

## Context
You need lightweight Home Screen widgets and Siri Shortcuts that display calendar data from your app's CoreData store. Widget content may vary based on [[07-storekit2-intelligence-based-trial|trial state]]. Widgets must refresh periodically, display correctly in multiple sizes, and use the same design language as the main app.

## Pattern

### Widget Extension as Separate Target

```swift
// In Project.swift — widget is a separate [[01-tuist-modular-architecture|Tuist]] target, not part of the app framework graph
let widgetTarget: [Target] = [
    .target(
        name: "WythnosWidgets",
        destinations: destinations,
        product: .appExtension,
        bundleId: "app.wythnos.ios.widgets",
        sources: ["WythnosWidgets/Sources/**"],
        dependencies: []  // Lightweight — no framework dependencies
    )
]
```

### TimelineProvider Pattern

```swift
struct TomorrowProvider: TimelineProvider {
    func placeholder(in context: Context) -> TomorrowEntry {
        TomorrowEntry(date: Date(), eventCount: 4, summary: "Moderately busy", nextEvent: "Team Standup at 9:00 AM")
    }

    func getSnapshot(in context: Context, completion: @escaping (TomorrowEntry) -> Void) {
        // Quick data for widget gallery preview
        completion(TomorrowEntry(date: Date(), eventCount: 4, summary: "Moderately busy", nextEvent: "Team Standup at 9:00 AM"))
    }

    func getTimeline(in context: Context, completion: @escaping (Timeline<TomorrowEntry>) -> Void) {
        // In production: fetch from shared CoreData via App Groups
        let entry = TomorrowEntry(
            date: Date(),
            eventCount: fetchEventCount(),
            summary: generateSummary(),
            nextEvent: fetchNextEvent()
        )
        
        let nextUpdate = Calendar.current.date(byAdding: .minute, value: 15, to: Date()) ?? Date().addingTimeInterval(900)
        let timeline = Timeline(entries: [entry], policy: .after(nextUpdate))
        completion(timeline)
    }
}
```

### Multi-Size Widget Views

```swift
struct TomorrowWidgetView: View {
    var entry: TomorrowProvider.Entry
    @Environment(\.widgetFamily) var family

    var body: some View {
        switch family {
        case .systemSmall: smallView
        case .systemMedium: mediumView
        default: smallView
        }
    }

    private var smallView: some View {
        VStack(spacing: 8) {
            HStack {
                Image(systemName: "sun.horizon").font(.system(size: 16))
                Text("Tomorrow").font(.system(size: 12, weight: .medium))
                Spacer()
            }
            .foregroundColor(.white.opacity(0.7))
            
            Text("\(entry.eventCount)")
                .font(.system(size: 34, weight: .bold, design: .monospaced))
                .foregroundColor(.green)
            
            Text(entry.summary)
                .font(.system(size: 11))
                .foregroundColor(.white.opacity(0.8))
        }
        .padding(12)
        .containerBackground(.black, for: .widget)
    }
}
```

### App Intent for Siri Shortcuts

```swift
import AppIntents

struct AskWythnosIntent: AppIntent {
    static var title: LocalizedStringResource = "Ask Wythnos"
    static var description: IntentDescription = "Ask a question about your calendar"
    
    @Parameter(title: "Question")
    var question: String
    
    func perform() async throws -> some IntentResult & ProvidesDialog {
        // Use shared data store to generate response
        let response = try await generateResponse(for: question)
        return .result(dialog: "\(response)")
    }
}
```

### Widget Configuration

```swift
@main
struct TomorrowWidget: Widget {
    let kind: String = "TomorrowWidget"

    var body: some WidgetConfiguration {
        StaticConfiguration(kind: kind, provider: TomorrowProvider()) { entry in
            TomorrowWidgetView(entry: entry)
        }
        .configurationDisplayName("Tomorrow Preview")
        .description("See what's coming up tomorrow")
        .supportedFamilies([.systemSmall, .systemMedium])
    }
}
```

### Data Sharing via App Groups

For widgets to access the main app's [[12-eventkit-coredata-sync-architecture|CoreData store]]:
1. Enable App Groups capability for both app and widget targets
2. Use a shared `NSPersistentContainer` pointed at the App Group directory
3. Widget reads data; main app writes data

## Edge Cases

- **Stale widget data:** Widgets refresh on a 15-minute cadence at best, but iOS may delay refreshes to save battery. Show a "Last updated" timestamp in the widget and use `.atEnd` reload policy for time-sensitive data so the system refreshes when the current timeline expires.
- **First-install empty state:** On first launch, the CoreData store is empty and the widget has no data to display. Return a meaningful placeholder (e.g., "Open Wythnos to sync your calendar") instead of showing zeros or blank content.
- **App Group container migration:** If you change the App Group identifier, existing widget data becomes inaccessible. Plan for migration by checking both old and new container URLs on launch.
- **Missing calendar permission in timeline provider:** The widget extension runs as a separate process and may not have calendar access. Always check authorization status in `getTimeline()` and return a "Grant calendar access in the app" entry if denied.
- **Widget configuration intent validation:** `AskWythnosIntent` receives user input in the `question` parameter. Sanitize this input before passing to any data query or LLM — treat it as untrusted. Avoid using it in string interpolation for NSPredicate or os.Logger at `.public` privacy level.

## Why This Matters

- **Widgets are separate processes** — they can't import app frameworks, so keep them lightweight
- **`containerBackground(.black, for: .widget)`** is the iOS 17+ way to set widget backgrounds
- **`.monospaced` numbers** align properly in widget layouts
- **15-minute refresh** balances freshness with battery life
- **App Intents** enable Siri integration without building a separate Intents extension

## Anti-Patterns

- Don't import heavy app frameworks in widget targets — keep widgets self-contained
- Don't make network calls in widgets — use cached/shared data
- Don't use dynamic text sizes that break in small widgets — use fixed system fonts
- Don't forget placeholder — it's shown during widget loading and in the gallery
- Don't assume App Group data exists — always handle empty/missing CoreData stores in the widget
- Don't use unsanitized App Intent parameters in queries or log messages — treat all intent input as untrusted
