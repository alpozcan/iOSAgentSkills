---
title: "Structured Logging and Log Levels for iOS Apps"
description: "Per-module os.Logger configuration with subsystem/category organization, strict log level discipline, privacy annotations for PII protection, log forging prevention, and OSLogStore integration for debug builds."
---

# Structured Logging and Log Levels for iOS Apps

## Context

Effective debugging in a modular iOS app requires structured logging with proper privacy controls, subsystem/category organization per module, and log level discipline. Apple's unified logging system (`os.Logger`) is the correct choice — it's low-overhead, privacy-aware, and integrates with Console.app and Instruments. Using `print()` or `NSLog()` in production is unacceptable: they always format strings, lack privacy annotations, and provide no filtering capability. In a [[01-tuist-modular-architecture|Tuist modular project]], each module should own its Logger configuration so that subsystem filtering in Console.app maps directly to your module boundaries.

## Pattern

### Per-Module Logger Configuration

Each Tuist module defines its own Logger with a subsystem matching the bundle ID and categories for functional areas:

```swift
// In CalendarService module
import os

extension Logger {
    private static let subsystem = "com.app.CalendarService"

    static let sync = Logger(subsystem: subsystem, category: "sync")
    static let permissions = Logger(subsystem: subsystem, category: "permissions")
    static let queries = Logger(subsystem: subsystem, category: "queries")
}

// In ChatUI module
extension Logger {
    private static let subsystem = "com.app.ChatUI"

    static let viewModel = Logger(subsystem: subsystem, category: "viewModel")
    static let input = Logger(subsystem: subsystem, category: "input")
}

// In LLMEngine module
extension Logger {
    private static let subsystem = "com.app.LLMEngine"

    static let inference = Logger(subsystem: subsystem, category: "inference")
    static let genePool = Logger(subsystem: subsystem, category: "genePool")
    static let safety = Logger(subsystem: subsystem, category: "safety")
}
```

### Log Level Discipline

Strict rules for when to use each level:

```swift
// .debug — Development-only noise. Not persisted by default. High-frequency OK.
Logger.sync.debug("Fetching events for range \(startDate, privacy: .private) to \(endDate, privacy: .private)")

// .info — Interesting lifecycle events. Persisted only when log is collected.
Logger.sync.info("Calendar sync started for \(calendarCount, privacy: .public) calendars")

// .notice (default) — Significant state changes. Always persisted.
Logger.sync.notice("Calendar sync completed: \(insertedCount, privacy: .public) new, \(updatedCount, privacy: .public) updated")

// .error — Recoverable errors. Always persisted. Triggers attention in Console.
Logger.sync.error("Calendar sync failed: \(error.localizedDescription, privacy: .public)")

// .fault — Unrecoverable errors indicating programmer error. Always persisted.
Logger.sync.fault("Calendar store persistent container failed to load — this should never happen in production")
```

Use this decision tree when choosing a log level:

- Will this line fire > 100 times/sec? → `.debug`
- Is this a lifecycle event (start/stop/state change)? → `.info`
- Is this a significant outcome the user might care about? → `.notice`
- Did something go wrong but we can recover? → `.error`
- Did something go wrong that indicates a bug? → `.fault`

### Privacy Annotations

Privacy annotations are critical for protecting user data in logs. Getting this wrong means PII ends up in diagnostic reports that Apple and enterprise MDM systems can collect:

```swift
// NEVER log PII or calendar content at public visibility
Logger.sync.info("Syncing event: \(event.title, privacy: .private)")          // ✅ Redacted in production
Logger.sync.info("Syncing event: \(event.title)")                              // ❌ Defaults to private, BUT...
Logger.sync.info("Syncing event: \(event.title, privacy: .public)")            // ❌❌ NEVER DO THIS

// Use .public only for non-sensitive operational data
Logger.sync.info("Sync duration: \(duration, privacy: .public)ms, events: \(count, privacy: .public)")

// Hash sensitive values for correlation without exposure
Logger.sync.info("Processing user: \(userId, privacy: .private(mask: .hash))")
```

### Log Forging Prevention

User-controlled strings in logs can inject fake log entries. This is a real security concern in apps that process user input — such as chat interfaces or search fields:

```swift
// BAD — user input could contain newlines that create fake log entries
Logger.input.info("User query: \(userInput, privacy: .private)")
// If userInput = "hello\n[FAULT] SECURITY BREACH DETECTED" → misleading logs

// GOOD — sanitize or use structured logging
Logger.input.info("User query received, length: \(userInput.count, privacy: .public)")
// Or if you must log content:
let sanitized = userInput.replacingOccurrences(of: "\n", with: "\\n")
    .replacingOccurrences(of: "\r", with: "\\r")
Logger.input.debug("User query: \(sanitized, privacy: .private)")
```

### OSLogStore for In-App Log Retrieval

Read logs programmatically for debug modes (see [[09-debug-modes-and-mock-service-strategy]] for the debug mode activation pattern):

```swift
#if DEBUG
func exportRecentLogs() throws -> [String] {
    let store = try OSLogStore(scope: .currentProcessIdentifier)
    let position = store.position(timeIntervalSinceLatestBoot: -300) // Last 5 min
    let entries = try store.getEntries(at: position)
        .compactMap { $0 as? OSLogEntryLog }
        .filter { $0.subsystem.hasPrefix("com.app.") }
        .map { "[\($0.level.rawValue)] \($0.subsystem)/\($0.category): \($0.composedMessage)" }
    return entries
}
#endif
```

Gate behind `#if DEBUG` — log store access in production requires the `com.apple.logging.local-store` entitlement.

### Console.app and `log` CLI Filtering

Filter logs effectively during development:

```bash
# Stream logs from simulator for a specific subsystem
log stream --predicate 'subsystem == "com.app.CalendarService"' --level debug

# Filter by category
log stream --predicate 'subsystem == "com.app.LLMEngine" AND category == "inference"'

# Filter errors and faults only
log stream --predicate 'subsystem BEGINSWITH "com.app." AND messageType >= 16'

# Export to file for sharing
log collect --device-name "iPhone" --start "2024-01-15 10:00:00" --output logs.logarchive
```

Integrate with your Makefile (see [[18-makefile-for-ios-project-workflows]]):

```makefile
.PHONY: logs
logs: ## Stream app logs from simulator
	log stream --predicate 'subsystem BEGINSWITH "com.app."' --level debug
```

### Logging in Actors

`os.Logger` is `Sendable` and safe to use in actors (see [[06-actor-based-concurrency-patterns]] — Logger is one of the few Apple types that's truly Sendable):

```swift
actor CalendarSyncManager {
    private let logger = Logger(subsystem: "com.app.CalendarService", category: "sync")

    func performSync() async throws {
        logger.info("Starting sync")
        // ... sync logic
        logger.notice("Sync completed successfully")
    }
}
```

## Edge Cases

- **`privacy: .auto` behavior differs by context:** The default for string interpolation redacts in production but shows values in Console.app when connected to Xcode. Always test with both a connected and disconnected device to verify your privacy annotations behave as expected.
- **`OSLogStore` entitlement requirement:** `OSLogStore` access requires the `com.apple.logging.local-store` entitlement for production apps. Only use it in `#if DEBUG` builds. Attempting to use it without the entitlement silently fails or throws.
- **Log volume and disk I/O:** `.debug` logs are not persisted and have near-zero overhead when not being observed, but `.info` and above ARE persisted. Logging 1000+ `.info` messages/sec will impact disk I/O and can cause hangs on the main thread.
- **Lazy string interpolation:** String interpolation in Logger is lazy — the message is only formatted if the log level is active. Don't pre-format strings: `logger.debug("\(expensiveComputation())")` only runs `expensiveComputation()` if debug logging is active. This is a major performance advantage over `print()`.
- **Logger instances are lightweight:** `Logger` instances are just subsystem + category strings internally. Creating them in `init` is fine — no need to share them globally or treat them as expensive resources.
- **Multiline log messages:** Multiline messages render poorly in Console.app. Keep log messages to a single line for readability and reliable parsing.
- **`OSLogEntryLog.composedMessage` and redaction:** `composedMessage` returns the formatted message with redacted values shown as `<private>`. This is useful for debugging the privacy annotations themselves — if you see actual values where you expected `<private>`, your annotation is wrong.
- **Error type logging:** When logging errors from [[14-error-handling-and-typed-error-system|typed error systems]], log the error's `localizedDescription` at `.public` privacy (it contains no PII) but log any associated context at `.private`.

## Why This Matters

- **Near-zero overhead:** `os.Logger` has near-zero overhead when not observed — unlike `print()` which always formats strings and writes to stdout
- **Privacy protection:** Privacy annotations prevent accidental PII exposure in crash logs and diagnostic reports that Apple or enterprise MDM systems collect
- **Instant noise filtering:** Subsystem/category organization lets you filter noise instantly in Console.app during debugging — jump straight to the module that's misbehaving
- **Zoom-in debugging:** Log levels enable progressive debugging — start with `.error` to find what went wrong, then enable `.debug` for the specific subsystem to understand why
- **Instruments integration:** Structured logging feeds into MetricKit and Instruments for performance analysis (see [[25-performance-monitoring-and-profiling-patterns]])
- **Security:** Log forging prevention protects against misleading entries in security-sensitive logs — especially important in apps that process untrusted user input
- **Modular architecture alignment:** Per-module subsystems mirror your [[01-tuist-modular-architecture|Tuist module boundaries]], making it trivial to trace issues to the responsible module

## Anti-Patterns

- Don't use `print()` or `NSLog()` in production code — they always format strings, don't support privacy annotations, and provide no subsystem/category filtering
- Don't use `.public` privacy for user-generated content, calendar titles, names, or any PII — use `.private` or `.private(mask: .hash)` for correlation
- Don't create a custom logging wrapper that loses type safety or lazy evaluation — use `os.Logger` directly. Wrappers that accept `String` instead of `OSLogMessage` defeat the lazy formatting optimization
- Don't log at `.error` level for expected conditions (network timeout, missing calendar permission, rate limit) — use `.info` or `.notice`. Reserve `.error` for genuinely unexpected failures
- Don't log full request/response bodies at `.info` level — use `.debug` so they're not persisted and don't impact disk I/O
- Don't forget `privacy:` annotations — the default `.auto` is safe but explicit annotations are better for code review and make intent clear
- Don't use string concatenation for log messages — use string interpolation so `os.Logger` can apply lazy formatting and privacy redaction: `logger.info("count: \(count)")` not `logger.info("\("count: " + String(count))")`
- Don't log inside tight loops at `.info` or above — the disk I/O will cause hangs (see [[25-performance-monitoring-and-profiling-patterns]] for profiling these issues)
