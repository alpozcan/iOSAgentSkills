# Snapshot Testing for SwiftUI in Modular Tuist Apps

## Context
You're building a modular iOS app with SwiftUI and Tuist, supporting multiple languages (including RTL) and device sizes. You need visual regression testing that catches layout breaks, truncation issues, and RTL mirroring bugs before they reach production — without testing all 43 locales manually.

## Pattern

### Library Choice: swift-snapshot-testing (Point-Free)

Use **swift-snapshot-testing** (1.17+) from Point-Free. It's the de facto standard in the Swift ecosystem, endorsed by Paul Hudson, Chris Eidhof (objc.io), and John Sundell. Apple has not shipped a first-party snapshot testing framework as of 2026.

```swift
// Package.swift (Tuist-managed)
.package(url: "https://github.com/pointfreeco/swift-snapshot-testing", from: "1.17.0")
```

Why swift-snapshot-testing over `ImageRenderer` alone:
- `UIHostingController`-based rendering respects trait collections, safe area insets, navigation bars
- Built-in diffing with human-readable failure output
- Configurable precision tolerance for CI vs local rendering differences
- Automatic reference image recording when snapshots are missing (`.missing` mode)
- Full Swift Testing (`@Suite`, `@Test`) support

### Architecture: Per-Module Snapshot Tests

Each UI module owns its snapshot tests in a sibling `SnapshotTests/` directory, with reference images committed to git under `__Snapshots__/`:

```
Modules/Core/DesignSystem/
├── Sources/                          # Component source
├── Tests/                            # Unit tests
└── SnapshotTests/                    # Snapshot tests
    ├── __Snapshots__/                # Reference PNGs (git-tracked)
    │   └── NyxCardSnapshotTests/
    │       ├── defaultContent.en_iPhone16Pro.png
    │       ├── defaultContent.ar_iPhone16Pro.png
    │       └── ...
    ├── NyxCardSnapshotTests.swift
    └── NyxErrorCardSnapshotTests.swift
```

### Tuist Configuration

Create a shared `SnapshotTestSupport` static framework and per-module snapshot test targets:

```swift
// Shared test support (config, helpers, mocks)
let snapshotTestSupport: [Target] = [
    .target(
        name: "SnapshotTestSupport",
        destinations: destinations,
        product: .staticFramework,  // Not .framework — needs XCTest search paths
        bundleId: "app.myapp.ios.TestSupport.Snapshot",
        deploymentTargets: deploymentTargets,
        infoPlist: .default,
        sources: ["Modules/TestSupport/SnapshotTestSupport/Sources/**"],
        dependencies: [
            .external(name: "SnapshotTesting"),
            .target(name: "DesignSystem"),
            .target(name: "CalendarService"),
            // ... all core modules needed by mocks
        ],
        settings: .settings(
            base: [
                "ENABLE_TESTING_SEARCH_PATHS": "YES",  // Critical! Resolves XCTest/Testing
            ]
        )
    )
]

// Helper to create snapshot test targets per module
func snapshotTestTarget(
    name: String,
    modulePath: String,
    dependencies: [TargetDependency] = []
) -> Target {
    .target(
        name: "\(name)SnapshotTests",
        destinations: destinations,
        product: .unitTests,
        bundleId: "app.myapp.ios.\(name).SnapshotTests",
        deploymentTargets: deploymentTargets,
        infoPlist: .default,
        sources: ["\(modulePath)/SnapshotTests/**"],
        dependencies: [
            .target(name: name),
            .target(name: "SnapshotTestSupport"),
            .external(name: "SnapshotTesting"),
        ] + dependencies
    )
}
```

**Critical**: `SnapshotTestSupport` must be `.staticFramework` with `ENABLE_TESTING_SEARCH_PATHS: YES`. A regular `.framework` cannot resolve `XCTest` and `Testing` module dependencies that swift-snapshot-testing transitively requires.

### Strategic Language Matrix

Don't test all locales — test **5 representative languages** that cover all layout edge cases:

```swift
public enum SnapshotConfig {
    public static let allLocales: [SnapshotLocale] = [
        SnapshotLocale(name: "en", locale: Locale(identifier: "en_US"), layoutDirection: .leftToRight),
        SnapshotLocale(name: "ar", locale: Locale(identifier: "ar_SA"), layoutDirection: .rightToLeft),  // RTL mirror
        SnapshotLocale(name: "de", locale: Locale(identifier: "de_DE"), layoutDirection: .leftToRight),  // Long compound words
        SnapshotLocale(name: "ja", locale: Locale(identifier: "ja_JP"), layoutDirection: .leftToRight),  // CJK wider glyphs
        SnapshotLocale(name: "tr", locale: Locale(identifier: "tr_TR"), layoutDirection: .leftToRight),  // Dotted I edge cases
    ]
}
```

This covers: LTR baseline, full RTL mirroring, word truncation stress, CJK character widths, and locale-specific casing. Similar scripts (French≈German for layout, Hindi≈Arabic for RTL) are implicitly covered.

### Device Sizes

Test 3 sizes: smallest (truncation), most popular (baseline), largest (spacing):

```swift
public static let iPhoneSE = ViewImageConfig.iPhoneSe(.portrait)  // 375pt
public static let iPhone16Pro = ViewImageConfig(...)               // 393pt
public static let iPhone16ProMax = ViewImageConfig(...)            // 430pt
```

### Helper Functions

Two assertion helpers for the two main use cases:

```swift
/// Full-screen snapshot at a specific device size
@MainActor
public func assertComponentSnapshot<V: View>(
    of view: V,
    named name: String,
    device: ViewImageConfig = SnapshotConfig.iPhone16Pro,
    locale: SnapshotLocale = SnapshotConfig.allLocales[0],
    file: StaticString = #file, testName: String = #function, line: UInt = #line
) {
    let wrapped = view
        .environment(\.locale, locale.locale)
        .environment(\.layoutDirection, locale.layoutDirection)
        .background(Color.black)
        .preferredColorScheme(.dark)
    let host = UIHostingController(rootView: wrapped)
    host.overrideUserInterfaceStyle = .dark
    
    assertSnapshot(
        of: host,
        as: .image(on: device, precision: 0.99, perceptualPrecision: 0.98),
        named: "\(locale.name)_\(name)",
        file: file, testName: testName, line: line
    )
}

/// Component snapshot sized to fit content (no device chrome)
@MainActor
public func assertFittingSnapshot<V: View>(
    of view: V,
    named name: String,
    width: CGFloat = 393,
    locale: SnapshotLocale = SnapshotConfig.allLocales[0],
    file: StaticString = #file, testName: String = #function, line: UInt = #line
) {
    let wrapped = view
        .frame(width: width)
        .environment(\.locale, locale.locale)
        .environment(\.layoutDirection, locale.layoutDirection)
        .background(Color.black)
        .preferredColorScheme(.dark)
    let host = UIHostingController(rootView: wrapped)
    host.overrideUserInterfaceStyle = .dark
    
    assertSnapshot(
        of: host,
        as: .image(precision: 0.99, perceptualPrecision: 0.98),
        named: "\(locale.name)_\(name)",
        file: file, testName: testName, line: line
    )
}
```

### Writing Tests

**Component test** (reusable DesignSystem components — highest ROI):

```swift
import Testing
import SnapshotTesting
import SwiftUI
@testable import DesignSystem
@testable import SnapshotTestSupport

@Suite("NyxErrorCard Snapshots")
@MainActor
struct NyxErrorCardSnapshotTests {
    init() { UIView.setAnimationsEnabled(false) }  // Deterministic

    @Test("All error types — all devices")
    func calendarAccessDenied() {
        let view = NyxErrorCard(error: .calendarAccessDenied).padding()
        for (deviceName, config) in SnapshotConfig.allDevices {
            assertFittingSnapshot(of: view, named: "calendarDenied_\(deviceName)", width: config.size?.width ?? 393)
        }
    }

    @Test("RTL layout — Arabic")
    func rtlLayout() {
        let view = NyxErrorCard(error: .calendarAccessDenied).padding()
        assertFittingSnapshot(of: view, named: "calendarDenied", locale: SnapshotConfig.allLocales[1])
    }
}
```

**Screen test** (full feature views — use mock ViewModels):

```swift
@Suite("ChatView Snapshots")
@MainActor
struct ChatViewSnapshotTests {
    init() { UIView.setAnimationsEnabled(false) }

    @Test("Conversation — all devices")
    func conversationWithMessages() {
        let vm = makeViewModel()
        vm.messages = [
            ChatMessage(content: "What's on my calendar?", isUser: true),
            ChatMessage(content: "You have **3 meetings** starting at 9 AM.", isUser: false),
        ]
        let view = NavigationStack { ChatView(viewModel: vm) }
        for (deviceName, config) in SnapshotConfig.allDevices {
            assertComponentSnapshot(of: view, named: "conversation_\(deviceName)", device: config)
        }
    }

    private func makeViewModel() -> ChatViewModel {
        ChatViewModel(
            store: SnapshotMockCalendarStore(),
            llmEngine: SnapshotMockLLMEngine(),
            // ... inject all mocks
        )
    }
}
```

### Mock Strategy for Snapshots

Create lightweight mocks in the shared `SnapshotTestSupport` module. They must:
- Return fixed, deterministic data (no `Date()`, no random values)
- Conform to all protocol requirements
- Be `Sendable` (actors for stateful protocols, structs for stateless)

```swift
public actor SnapshotMockCalendarStore: CalendarStoreProtocol {
    public func fetchEvents(from: Date, to: Date) async throws -> [CalendarEvent] {
        MockEventFactory.workDay()  // Fixed reference date, deterministic events
    }
    // ... implement all protocol methods
}
```

### Recording and Verification

swift-snapshot-testing 1.18+ defaults to `.missing` mode — **automatically records** when no reference image exists, then compares on subsequent runs:

```bash
# First run: records all reference images (tests report as "failed" to flag new recordings)
xcodebuild test -scheme DesignSystem -only-testing:DesignSystemSnapshotTests \
  -destination 'platform=iOS Simulator,name=iPhone 17 Pro'

# Second run: compares against references (tests pass)
xcodebuild test -scheme DesignSystem -only-testing:DesignSystemSnapshotTests \
  -destination 'platform=iOS Simulator,name=iPhone 17 Pro'

# Force re-record all (when visual changes are intentional):
SNAPSHOT_TESTING_RECORD=all xcodebuild test ...
```

### CI Considerations

1. **Pin the simulator**: Always use the same device on CI — different simulators produce different pixels
2. **Set precision tolerance**: `0.99` precision + `0.98` perceptual precision handles minor anti-aliasing differences
3. **Never auto-record on CI**: The default `.missing` mode records locally; on CI, missing references should fail
4. **Commit `__Snapshots__/`**: Reference images must be in git so diffs are visible in PRs
5. **Parallel execution**: Snapshot tests are stateless — enable Xcode parallel testing

## Why This Matters

- **Visual regressions caught automatically**: A one-line color change in the design system is caught across all 6 modules
- **RTL bugs caught early**: Arabic layout mirroring is tested in isolation, not discovered by a user
- **Truncation prevention**: German compound words + iPhone SE width = guaranteed truncation detection
- **Component-level tests have highest ROI**: One NyxCard snapshot protects every screen that uses it
- **Per-module ownership**: Each team/module maintains its own snapshots — no monolithic test target

## Anti-Patterns

- Don't use `.framework` for the test support module — it can't resolve XCTest. Use `.staticFramework` with `ENABLE_TESTING_SEARCH_PATHS: YES`
- Don't use `ImageRenderer` for complex views — it doesn't handle navigation bars, safe areas, or trait collections correctly
- Don't test all 43 languages — 5 strategic representatives cover all layout edge cases
- Don't leave animations enabled — they make snapshots non-deterministic. Call `UIView.setAnimationsEnabled(false)` in test init
- Don't use `Date()` or random values in mocks — snapshots must be 100% reproducible
- Don't gitignore `__Snapshots__/` — reference images must be committed for PR diffing
- Don't skip the `named:` parameter in parameterized tests — without it, snapshots overwrite each other
- Don't run recording on CI — only record locally, verify on CI
