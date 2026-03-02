# UI — Map of Content

Design system infrastructure, custom navigation, layout patterns, and multi-language support for SwiftUI apps that feel premium.

## The Design Foundation

[[04-design-system-as-core-module]] defines the visual language as a Core module: color tokens, typography scales, spacing grid, reusable components, animation primitives, and a centralized haptic feedback system. Every UI skill in this graph builds on these tokens. The design system enforces dark-mode-only, 4pt grid alignment, and OLED-optimized true black backgrounds through code — not documentation.

## Navigation & Layout

[[13-swiftui-custom-tab-bar-and-navigation]] replaces the system `TabView` with a custom tab bar using `safeAreaInset(edge: .bottom)`. It uses haptics and animations from [[04-design-system-as-core-module]], handles re-tap scroll-to-top via `.id()` changes, and manages splash screen to content transitions. The tab bar itself becomes the root layout element that every other view must work within.

That creates a specific layout challenge documented in [[17-safe-area-inset-stacking-and-bottom-pinned-views]]: when you need to pin additional chrome (like a chat input bar) above the custom tab bar, nested `safeAreaInset` calls fail silently. This skill documents five approaches that don't work and the one that does — placing all bottom-pinned chrome inside a single `safeAreaInset` block at the outermost level.

## Multi-Language Support

[[16-localization-and-multi-language-patterns]] handles 43 languages through per-module string catalogs with `bundle: .module`. It goes beyond string translation into bilingual AI intent classification (English + Turkish keyword sets) and locale-aware safety refusal responses. The strategic language matrix here feeds directly into [[17-snapshot-testing-with-swift-snapshot-testing]] where 5 representative locales (en, ar, de, ja, tr) are tested for visual regressions.

## In-App Browsing

[[31-in-app-safari-for-external-links]] enforces a project-wide rule: every external URL opens inside an `SFSafariViewController` rather than ejecting the user to Safari. The skill provides a reusable `SafariView` SwiftUI wrapper, a `URL: Identifiable` conformance for `item:`-based presentation, and toolbar tinting that uses tokens from [[04-design-system-as-core-module]].

## How They Connect

```
04 Design System (tokens, components, haptics)
 ├── 13 Custom Tab Bar (uses tokens, haptics, animations)
 │    └── 17 Safe Area Stacking (solves layout problems from 13)
 ├── 16 Localization (language-agnostic tokens, locale-aware content)
 └── 31 In-App Safari (tint colors from tokens, fullScreenCover presentation)
```
