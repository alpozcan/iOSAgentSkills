# Swift Testing Framework Patterns and TDD in Modular iOS Apps

## Context
You're building a modular iOS app where every Core and Feature module has its own test target. You need fast, deterministic tests that run in parallel, use the modern Swift Testing framework (`@Test`, `#expect`), and validate real business logic — not just UI snapshots.

## Pattern

### Test Framework Choice: Swift Testing Over XCTest

Use the **Swift Testing framework** (import Testing, `@Suite`, `@Test`, `#expect`) for all unit tests. Reserve XCTest only for:
- UI tests (XCUITest requires it)
- Tests that need `setUp()`/`tearDown()` lifecycle (e.g., file system cleanup)
- Legacy tests being migrated

```swift
import Testing
@testable import SubscriptionService

@Suite("TrialManager Tests")
struct TrialManagerTests {
    @Test("New user starts in trial state")
    func newUserStartsInTrial() async {
        let defaults = UserDefaults(suiteName: #function)!  // Isolated storage
        let manager = TrialManager(defaults: defaults)
        #expect(await manager.currentState == .trial)
        #expect(await manager.hasProAccess == true)
    }
    
    @Test("Maturity threshold at 50 interactions")
    func maturityAt50Interactions() async {
        let defaults = UserDefaults(suiteName: #function)!
        let manager = TrialManager(defaults: defaults)
        #expect(await manager.checkMaturity(totalInteractions: 50, discoveredPatterns: 0) == true)
        #expect(await manager.checkMaturity(totalInteractions: 49, discoveredPatterns: 0) == false)
    }
}
```

### Test Isolation via UserDefaults Suites

When testing code that uses `UserDefaults`, create isolated stores per test using `#function` as the suite name:

```swift
let defaults = UserDefaults(suiteName: #function)!
let tracker = DailyQueryTracker(defaults: defaults)
```

This prevents test pollution across parallel test runs.

### Test Categories by Module Type

**1. Service/Core module tests — test business logic directly:**
```swift
@Suite("DailyQueryTracker Tests")
struct DailyQueryTrackerTests {
    @Test("Recording query decrements remaining")
    func recordingQueryDecrements() {
        let defaults = UserDefaults(suiteName: #function)!
        let tracker = DailyQueryTracker(defaults: defaults)
        _ = tracker.recordQuery()
        #expect(tracker.queriesRemaining == 2)
    }
    
    @Test("Limit reached after exhausting queries")
    func limitReached() {
        let defaults = UserDefaults(suiteName: #function)!
        let tracker = DailyQueryTracker(defaults: defaults)
        _ = tracker.recordQuery()
        _ = tracker.recordQuery()
        _ = tracker.recordQuery()
        #expect(tracker.isLimitReached == true)
        #expect(tracker.recordQuery() == false)
    }
}
```

**2. Catalog tests — verify registration, resolution, singleton behavior (see [[02-protocol-driven-service-catalog]]):**
```swift
@Suite("Catalog Tests")
struct CatalogTests {
    init() { Catalog.main.clear() }

    @Test("Obtain calls factory each time (not singleton)")
    func obtainCallsFactoryEachTime() {
        var count = 0
        Catalog.main.supply(TestBlueprint.self) { _ in
            count += 1; return TestServiceImpl(value: "call-\(count)")
        }
        let s1 = Catalog.main.obtain(TestBlueprint.self)
        let s2 = Catalog.main.obtain(TestBlueprint.self)
        #expect(s1.value == "call-1")
        #expect(s2.value == "call-2")
    }
}
```

**3. LLM/AI engine tests — validate prompt construction, gene pool evolution:**
```swift
@Suite("DynamicPromptEngine Tests")
struct DynamicPromptEngineTests {
    @Test("Intent classification works for different queries")
    func intentClassification() {
        let engine = DynamicPromptEngine()
        #expect(engine.classifyIntent("Do I have any conflicts?").category == .conflictDetection)
        #expect(engine.classifyIntent("When am I free?").category == .freeTimeAnalysis)
        #expect(engine.classifyIntent("Çakışma var mı?").category == .conflictDetection)  // Turkish
    }
    
    @Test("Evolution stage advances with interaction count")
    func evolutionStageAdvancement() {
        let engine = DynamicPromptEngine()
        engine.setInteractionCount(0);   #expect(engine.currentEvolutionStage == .learning)
        engine.setInteractionCount(50);  #expect(engine.currentEvolutionStage == .adapting)
        engine.setInteractionCount(200); #expect(engine.currentEvolutionStage == .predicting)
        engine.setInteractionCount(500); #expect(engine.currentEvolutionStage == .mastered)
    }
    
    @Test("Positive feedback increases gene fitness")
    func positiveFeedback() {
        let pool = PromptGenePool()
        let gene = PromptGene(type: .systemPersona, content: "Test", fitnessScore: 0.5)
        pool.addGene(gene)
        let tracker = PromptEvolutionTracker(genePool: pool)
        
        tracker.evolveFromFeedback(
            prompt: SynthesizedPrompt(components: [gene], metadata: .init()),
            response: "test",
            feedback: UserFeedback(isPositive: true)
        )
        
        let updated = pool.genesForType(.systemPersona).first!
        #expect(updated.fitnessScore > 0.5)
        #expect(updated.positiveReactions == 1)
    }
}
```

**4. Sanitizer/filter tests — validate output hygiene:**
```swift
// XCTest used here for setUp/tearDown pattern
final class ResponseSanitizerTests: XCTestCase {
    private var sut: ResponseSanitizer!
    override func setUp() { sut = ResponseSanitizer() }
    
    func testExtractsFollowUpQuestion() {
        let input = "I can help.Follow-up question: \"What about next week?\""
        let result = sut.sanitize(input)
        XCTAssertEqual(result.mainContent, "I can help.")
        XCTAssertEqual(result.followUpQuestion, "What about next week?")
    }
    
    func testRealWorldBug() {
        // Regression test from actual screenshot
        let input = "Sorry, I can't help with that.Most useful part: Acknowledging the limitation..."
        let result = sut.sanitize(input)
        XCTAssertFalse(result.mainContent.contains("Most useful part:"))
    }
}
```

**5. Actor-based async tests:**
```swift
@Suite("InMemoryTrainingStore Tests")
struct InMemoryTrainingStoreTests {
    @Test("Untrained pairs tracking")
    func untrainedPairs() async {
        let store = InMemoryTrainingStore()
        let pairs = (0..<5).map { i in TrainingPair(instruction: "test \(i)") }
        await store.appendTrainingPairs(pairs)
        #expect(await store.untrainedPairCount() == 5)
        await store.markAsTrained(Array(pairs.prefix(3)))
        #expect(await store.untrainedPairCount() == 2)
    }
}
```

### UI Test Base Class Pattern

```swift
class WythnosUITestCase: XCTestCase {
    var app: XCUIApplication!
    
    var launchArguments: [String] { ["--uitesting", "--skip-onboarding"] }
    
    override func setUpWithError() throws {
        continueAfterFailure = false
        app = XCUIApplication()
        app.launchArguments = launchArguments
        app.launch()
    }
    
    // Shared navigation helpers
    func navigateToChat() { tapTab("tab_chat") }
    func waitForText(_ text: String, timeout: TimeInterval = 10) -> Bool { ... }
    func assertScreenVisible(_ id: String) { ... }
}
```

### Stub Service Strategy for Tests

**Minimal stubs (UI testing) — fast, predictable:**
```swift
struct StubLLMEngine: LLMEngineProtocol {
    func generateResponse(prompt: String) async throws -> String {
        try? await Task.sleep(for: .milliseconds(200))
        return "Stub AI response for UI testing."
    }
}
```

**Rich stubs (debug mode) — realistic data patterns:**
```swift
actor PreviewCalendarStore: CalendarStoreProtocol {
    init() { self.events = generatePreviewEvents() }

    private func generatePreviewEvents() -> [CalendarEvent] {
        // 30 days of realistic work/personal events
        // Standups Mon/Wed/Fri, deep work blocks, 1:1s, weekend gym
    }
}
```

## Why This Matters

- **Swift Testing is 2-3x faster** than XCTest for pure logic tests due to parallel execution
- **`UserDefaults(suiteName: #function)`** guarantees zero test interference
- **Regression tests from real bugs** (screenshot-driven) prevent regressions
- **Stub override pattern** (`Catalog.main.supply` in test setup) enables full integration testing without external services — see [[09-debug-modes-and-mock-service-strategy]] for the three-tier stub system
- **Async-first testing** with `async` test functions validates actor-based concurrency correctly

## Anti-Patterns

- Don't share mutable state between `@Test` functions — each test should create its own instances
- Don't use `XCTAssert*` in Swift Testing suites — use `#expect`
- Don't mock everything — test real business logic with real implementations where possible (e.g., `DynamicPromptEngine`)
- Don't skip actor isolation in tests — use `await` to test actors correctly
- Don't write tests that depend on execution order — Swift Testing runs tests in parallel by default
