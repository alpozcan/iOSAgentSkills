# Localization and Multi-Language Patterns for iOS

## Context
Your app targets 43 languages and uses AI that must handle multi-language input (e.g., Turkish/English mixing). Localized strings must work across modular frameworks, and the AI intent classification must recognize keywords in multiple languages.

## Pattern

### String Localization in Modular Frameworks

Each module uses `bundle: .module` (Tuist-generated) for its own string catalogs:

```swift
// In ChatUI module
Text(String(localized: "chat.premium.required", bundle: .module))

// In MainTabView (app module)
var label: String {
    switch self {
    case .insights: return String(localized: "tab.insights", defaultValue: "Insights", bundle: .module)
    case .settings: return String(localized: "tab.settings", defaultValue: "Settings", bundle: .module)
    }
}
```

### Multi-Language AI Intent Classification

The intent classifier must recognize keywords in every supported language:

```swift
public func classifyIntent(_ query: String) -> UserIntent {
    let lower = query.lowercased()
    
    // English + Turkish conflict detection
    if lower.contains("conflict") || lower.contains("overlap") || lower.contains("clash") ||
       lower.contains("çakışma") || lower.contains("çatışma") {
        return .conflictCheck
    }
    
    // English + Turkish free time
    if lower.contains("free") || lower.contains("available") || lower.contains("open") ||
       lower.contains("boş") || lower.contains("müsait") {
        return .freeTimeSearch(duration: nil)
    }
    
    // English + Turkish productivity
    if lower.contains("productiv") || lower.contains("efficient") ||
       lower.contains("verimlilik") || lower.contains("yoğun") {
        return .productivityAnalysis
    }
}
```

### Domain Classifier with Bilingual Keywords

```swift
public struct DomainClassifier: Sendable {
    private let calendarKeywords: [String] = [
        // English
        "calendar", "schedule", "event", "meeting", "appointment",
        "today", "tomorrow", "week", "month",
        "monday", "tuesday", "wednesday", "thursday", "friday",
        "free time", "available", "busy", "overlap",
        "zoom", "meet", "teams",
        
        // Turkish
        "takvim", "toplantı", "randevu", "zaman", "hafta", "gün",
        "bugün", "yarın", "gelecek", "geçen", "saat", "ne zaman"
    ]
}
```

### Localized Safety Refusal Responses

```swift
private let responses: [String: [RefusalType: String]] = [
    "en": [
        .offTopic: "I'm Wythnos, your calendar intelligence assistant. I specialize in scheduling...",
        .selfHarm: "I'm not equipped to help with this, but support is available..."
    ],
    "tr": [
        .offTopic: "Ben Wythnos, takvim zekası asistanınızım. Programlama konusunda uzmanım...",
        .selfHarm: "Bu konuda yardımcı olabilecek donanıma sahip değilim, ama destek mevcut..."
    ]
]

public func generate(for type: RefusalType, locale: Locale) -> String {
    let lang = locale.language.languageCode?.identifier ?? "en"
    return responses[lang]?[type] ?? responses["en"]?[type] ?? defaultResponse
}
```

### Gene Pool with Locale Tracking

```swift
public struct PromptGene: Codable, Identifiable, Sendable {
    public let locale: String  // "en", "tr", etc.
    // Genes are locale-aware — Turkish-speaking users get Turkish-locale genes
}

// Seed data includes Turkish-specific genes
@Test("Seed pool includes Turkish locale genes")
func turkishGenesPresent() {
    let seeds = SeedGeneFactory.createSeedPool()
    let turkishGenes = seeds.filter { $0.locale == "tr" }
    #expect(!turkishGenes.isEmpty)
}
```

### Test Coverage for Multi-Language

```swift
@Test("Turkish intent classification works")
func turkishIntentClassification() {
    let engine = DynamicPromptEngine()
    #expect(engine.classifyIntent("Çakışma var mı?").category == .conflictDetection)
    #expect(engine.classifyIntent("Ne zaman müsaitim?").category == .freeTimeAnalysis)
}
```

## Why This Matters

- **`bundle: .module`** ensures each framework resolves its own string catalog, not the app's
- **Bilingual intent classification** handles code-switching (Turkish/English in same query)
- **Locale-aware gene pool** means the AI's personality adapts to the user's language
- **Localized safety responses** are critical — a Turkish-speaking user in crisis must see resources in Turkish
- **Test coverage for each language** catches regressions in keyword-based classification

## Anti-Patterns

- Don't use `NSLocalizedString` — use `String(localized:bundle:)` (modern API)
- Don't hardcode English-only keywords in AI classifiers
- Don't assume Locale.current returns a 2-letter code — use `locale.language.languageCode?.identifier`
- Don't put all strings in the app target — each module owns its own localizations
- Don't skip localization tests — they're the only way to catch missing translations before release
