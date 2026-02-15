# Contributing to iOS Agent Skills

Thank you for your interest in contributing! This project is a community-driven collection of iOS development skills for AI coding agents.

## Adding a New Skill

### 1. Choose the Right Category

| Category | What belongs here |
|----------|-------------------|
| `skills/architecture/` | App structure, dependency management, concurrency, error handling |
| `skills/ui/` | Design systems, navigation, layout, localization, accessibility |
| `skills/testing-and-debugging/` | Test patterns, mocking strategies, debugging tools |
| `skills/platform-frameworks/` | StoreKit, EventKit, WidgetKit, Notifications, Analytics |
| `skills/ai-and-intelligence/` | On-device ML, LLM integration, intent classification |

If your skill doesn't fit any existing category, propose a new one in your PR description.

### 2. Follow the File Naming Convention

```
XX-descriptive-name.md
```

- `XX` is a two-digit number (next available in the overall sequence)
- Use lowercase kebab-case for the descriptive name
- Example: `17-accessibility-voiceover-patterns.md`

### 3. Follow the Skill Template

Every skill should have this structure:

```markdown
# Title

## Context
Explain WHEN and WHY this pattern is needed. What problem does it solve?

## Pattern
The concrete implementation with Swift code examples.
Use ### subheadings to break down the pattern into layers/steps.

## Rules / Anti-patterns (optional but encouraged)
- ✅ DO: Concrete guidance
- ❌ DON'T: Common mistakes to avoid
```

### 4. Quality Checklist

- [ ] Code examples compile (or are clearly pseudocode)
- [ ] Swift 6+ with strict concurrency where applicable
- [ ] No third-party dependencies unless the skill is specifically about that framework
- [ ] Self-contained — the skill makes sense without reading other skills
- [ ] Cross-references to related skills use relative links

### 5. Update the Root README

Add your skill to the appropriate table in `README.md`.

## Improving Existing Skills

Bug fixes, clarifications, and additional examples are always welcome. For significant rewrites, open an issue first to discuss the changes.

## Code of Conduct

Be respectful, constructive, and inclusive. We're here to help each other build better iOS apps.
