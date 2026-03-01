---
title: "Performance Monitoring and Profiling Patterns"
description: "MetricKit integration, os.signpost custom intervals, launch time profiling, memory pressure tests, and privacy-aware metric reporting for modular iOS apps with on-device LLM inference and CoreData sync."
---

# Performance Monitoring and Profiling Patterns

## Context

You're building a modular iOS app with on-device LLM inference (see [[08-on-device-llm-with-apple-foundation-models]]), CoreData-backed calendar sync (see [[12-eventkit-coredata-sync-architecture]]), and real-time UI updates. Performance regressions — hang rate increases, memory spikes, slow launches — erode user trust and hurt App Store search ranking. Apple provides MetricKit, `os.signpost`, and Instruments, but integrating them into a modular Tuist architecture requires deliberate structure so that every module can be profiled independently and metrics flow through a single privacy-respecting pipeline.

## Pattern

### MetricKit Integration

Subscribe to `MXMetricManager` at app launch to receive daily performance payloads and crash diagnostics. Forward extracted metrics through the analytics layer (see [[10-privacy-first-analytics-architecture]]) — never forward raw payloads.

```swift
import MetricKit
import os

@MainActor
final class MetricSubscriber: NSObject, MXMetricManagerSubscriber {
    private let analytics: AnalyticsServiceProtocol

    init(analytics: AnalyticsServiceProtocol) {
        self.analytics = analytics
        super.init()
        MXMetricManager.shared.add(self)
    }

    deinit {
        MXMetricManager.shared.remove(self)
    }

    func didReceive(_ payloads: [MXMetricPayload]) {
        for payload in payloads {
            // Extract key metrics
            let hangTime = payload.applicationTimeMetrics?.cumulativeHangTime
            let launchTime = payload.applicationLaunchMetrics?.histogrammedTimeToFirstDraw
            // Forward to analytics (reference [[10-privacy-first-analytics-architecture]])
            analytics.track(.performanceMetric(hangTime: hangTime, launchTime: launchTime))
        }
    }

    func didReceive(_ payloads: [MXDiagnosticPayload]) {
        for payload in payloads {
            if let hangDiagnostics = payload.hangDiagnostics {
                for hang in hangDiagnostics {
                    Logger.performance.warning("Hang detected: \(hang.hangDuration, privacy: .public)")
                }
            }
        }
    }
}
```

Register the subscriber early in the app lifecycle, typically in the `App` init or `AppDelegate.application(_:didFinishLaunchingWithOptions:)`.

### os.signpost for Custom Intervals

Use `OSSignposter` to measure specific operations like LLM inference, calendar sync, or any expensive work. These intervals appear in Instruments' **Points of Interest** track, making it straightforward to correlate timing with UI hangs or energy spikes.

```swift
import os

extension Logger {
    static let performance = Logger(subsystem: "com.app.performance", category: "timing")
}

// Reuse signposter as a static property — do NOT create per call
let signposter = OSSignposter(subsystem: "com.app.performance", category: "timing")

// Measure LLM inference time
func measureInference() async throws -> String {
    let signpostID = signposter.makeSignpostID()
    let state = signposter.beginInterval("LLM Inference", id: signpostID)
    defer { signposter.endInterval("LLM Inference", state) }

    return try await llmEngine.generate(prompt: prompt)
}

// Measure calendar sync
func measureSync() async throws {
    let state = signposter.beginInterval("Calendar Sync")
    defer { signposter.endInterval("Calendar Sync", state) }

    try await calendarSyncManager.performFullSync()
}
```

In Instruments, open the **os_signpost** instrument or the **Points of Interest** track. Filter by subsystem `com.app.performance` to see each interval with its duration. Overlapping intervals from concurrent operations appear on separate lanes, making it easy to spot contention.

### Launch Time Profiling

Measure app launch in distinct phases. MetricKit captures pre-main time automatically; use signposts for everything after.

```swift
// In AppDelegate or @main App
let launchSignposter = OSSignposter(subsystem: "com.app", category: "launch")

// Phase 1: Pre-main (measured by MetricKit automatically)

// Phase 2: Catalog preparation
let diState = launchSignposter.beginInterval("Catalog Preparation")
Catalog.main.prepareAll()
launchSignposter.endInterval("Catalog Preparation", diState)

// Phase 3: First frame
let uiState = launchSignposter.beginInterval("First Frame")
// ... in onAppear of root view:
launchSignposter.endInterval("First Frame", uiState)
```

Target budgets:
- **Warm launch:** < 400ms to first frame
- **Cold launch:** < 2s to first frame
- **Catalog preparation:** < 50ms (lazy-register heavy services)

If catalog preparation grows beyond budget, switch to lazy registration — see [[06-actor-based-concurrency-patterns]] for actor-based lazy initialization that is safe across threads.

### Memory Profiling Per Module

Track memory allocations per Tuist module using Instruments' Allocations template. For automated regression detection, add memory pressure tests using Swift Testing:

```swift
import Testing

@Test("Calendar sync handles 10K events without excessive memory")
func largeSyncMemory() async throws {
    let store = CalendarStore(inMemory: true)
    let events = (0..<10_000).map { MockEventFactory.event(index: $0) }

    let before = reportMemory()
    try await store.upsertEvents(events)
    let after = reportMemory()

    let growth = after - before
    #expect(growth < 50_000_000) // < 50 MB for 10K events
}

func reportMemory() -> Int {
    var info = mach_task_basic_info()
    var count = mach_msg_type_number_t(MemoryLayout<mach_task_basic_info>.size) / 4
    let result = withUnsafeMutablePointer(to: &info) {
        $0.withMemoryRebound(to: integer_t.self, capacity: Int(count)) {
            task_info(mach_task_self_, task_flavor_t(MACH_TASK_BASIC_INFO), $0, &count)
        }
    }
    return result == KERN_SUCCESS ? Int(info.resident_size) : 0
}
```

Run these tests in CI (see [[18-makefile-for-ios-project-workflows]]) to catch memory regressions before they reach production.

### Privacy-Aware Metric Reporting

Performance metrics must go through the same privacy filter as all analytics. Never send raw stack traces, method names, calendar content, or prompt text. Report only aggregated, typed metrics:

```swift
struct PerformanceMetric: Sendable {
    let operation: String       // "llm_inference", "calendar_sync"
    let durationMs: Double      // Timing only
    let success: Bool           // Did it complete?
    // NO user data, NO calendar content, NO prompt text
}
```

See [[10-privacy-first-analytics-architecture]] for the full privacy filter rules and the buffer-flush pattern that batches these metrics before sending.

### Tuist Generation Time

As the project grows in modules, `tuist generate` itself can become a bottleneck. Profile it regularly:

```bash
# Measure generation time
time tuist generate --no-open

# Profile with verbose output
tuist generate --verbose 2>&1 | grep "Generated"
```

Generation time typically becomes noticeable at 15+ modules. Mitigations include focusing generation on a subset of modules during development (`tuist generate FeatureCalendar`) and caching resolved dependencies. See [[18-makefile-for-ios-project-workflows]] for Makefile targets that wrap these commands.

## Edge Cases

- **MetricKit delivery delay** — Payloads are delivered up to 24 hours after collection. Do not rely on MetricKit for real-time alerting; use signposts and local logging for immediate feedback.
- **Signpost overhead on hot paths** — `OSSignposter` has minimal overhead, but for paths called more than 1000 times per second, gate signpost calls behind `#if DEBUG` or use `signposter.isEnabled` to skip work when no profiler is attached.
- **Shared library memory in `mach_task_basic_info`** — `resident_size` includes memory from shared system libraries. Subtract a baseline measurement taken before the operation under test to get accurate per-module figures.
- **Observer effect with Instruments** — Profiling changes app behavior. Always profile in **Release** configuration (with debug symbols), not Debug. Debug builds disable compiler optimizations and inflate timing results.
- **MetricKit unavailable in Simulator** — `MXMetricManager` does not deliver payloads on simulators. Use signposts for simulator-based profiling during development; rely on MetricKit for production-only telemetry.
- **Energy diagnostics and LLM inference** — `MXCPUExceptionDiagnostic` fires when CPU usage exceeds 80% for extended periods. On-device LLM inference legitimately sustains high CPU; filter these diagnostics by operation context to avoid false alarms.
- **Thread explosion** — Too many concurrent `Task {}` blocks can exhaust the cooperative thread pool. Use `TaskGroup` with bounded concurrency (see [[06-actor-based-concurrency-patterns]]) to keep thread counts predictable during profiling.

## Why This Matters

- **App Store ranking** — Apple's "Hang Rate" metric directly impacts search ranking and user trust. A hang rate above the 50th percentile for your category pushes you down in results.
- **LLM inference regression detection** — On-device inference takes 2-10 seconds depending on model and hardware. Without monitoring, a model or OS update can silently double inference time.
- **Memory pressure and jetsam** — Calendar sync with 10K+ events can cause memory spikes that trigger jetsam kills, which appear to users as random crashes with no crash report.
- **Launch time perception** — Warm launch above 400ms or cold launch above 2 seconds degrades perceived quality. Users form opinions about app quality within the first interaction.
- **Privacy compliance** — Performance data must never leak personal calendar content, LLM prompts, or identifiable information. Structured metric types enforce this at the type level.

## Anti-Patterns

- **Using `CFAbsoluteTimeGetCurrent()` for timing** — This is wall-clock time and can jump during sleep or NTP adjustments. Use `OSSignposter` for profiling intervals or `ContinuousClock` for programmatic monotonic timing.
- **Logging raw MetricKit payloads to analytics** — Payloads may contain stack traces with method names that reveal internal architecture and could include symbols derived from user data. Extract only typed numeric metrics.
- **Measuring performance in Debug builds** — Debug builds disable compiler optimizations (`-Onone`), inflate timing by 2-10x, and include extra runtime checks. Always profile in Release configuration with debug symbols enabled.
- **Creating `OSSignposter` instances per call** — Each instantiation has overhead. Declare signposters as `static let` properties on the relevant type or as module-level constants and reuse them.
- **Ignoring jetsam reports** — Jetsam terminations do not generate standard crash reports. They indicate the app exceeded memory limits under pressure. Check MetricKit's `MXDiskWriteExceptionDiagnostic` and memory metrics, and validate with memory pressure tests.
- **Profiling with Instruments attached and expecting normal behavior** — The profiler adds overhead to every allocation, thread switch, and signpost emission. Use Instruments to find bottlenecks, then validate fixes with standalone timing tests that run without a profiler attached.
