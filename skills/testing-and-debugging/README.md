# Testing & Debugging — Map of Content

Quality assurance patterns that keep modular apps reliable across CI, simulators, and Xcode Previews.

## The Testing Pyramid

[[05-swift-testing-and-tdd-patterns]] is the foundation — Swift Testing framework (`@Suite`, `@Test`, `#expect`) over XCTest for all unit tests. It defines five test categories (service logic, DI container, AI engine, sanitizer/filter, actor-based async) and the test isolation pattern using `UserDefaults(suiteName: #function)`. The mock injection strategy here works hand-in-hand with [[09-debug-modes-and-mock-service-strategy]].

[[09-debug-modes-and-mock-service-strategy]] provides the data layer for testing: a three-tier mock system where Tier 1 (UI testing) gives minimal fixed data, Tier 2 (rich debug) gives 30 days of realistic calendar patterns with Pro access, and Tier 3 (developer mode) unlocks via a secret gesture with SHA256 verification. Launch arguments (`--uitesting`, `--pro-debug`, `--skip-onboarding`) control which tier is active, making tests from [[05-swift-testing-and-tdd-patterns]] and [[18-ui-testing-regression-and-smoke]] deterministic.

## Visual & Integration Testing

[[17-snapshot-testing-with-swift-snapshot-testing]] catches visual regressions at the component level using Point-Free's swift-snapshot-testing. It tests design system components from [[04-design-system-as-core-module]] across a strategic 5-language matrix (en, ar/RTL, de/long words, ja/CJK, tr) drawn from [[16-localization-and-multi-language-patterns]], and 3 device sizes. Component-level snapshots have the highest ROI — one `NyxCard` test protects every screen that uses it.

[[18-ui-testing-regression-and-smoke]] validates critical user journeys end-to-end via XCUITest: tab navigation from [[13-swiftui-custom-tab-bar-and-navigation]], chat input behavior from [[17-safe-area-inset-stacking-and-bottom-pinned-views]], settings interactions, and keyboard behavior. It uses launch arguments from [[09-debug-modes-and-mock-service-strategy]] for deterministic state.

## How They Connect

```
09 Debug Modes (provides mock data)
 ├── 05 Swift Testing (unit tests with mock injection)
 ├── 17 Snapshot Testing (visual regression with deterministic content)
 └── 18 UI Testing (end-to-end with launch argument flags)
```
