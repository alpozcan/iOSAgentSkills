# AI & Intelligence — Map of Content

On-device AI architecture for privacy-first, zero-latency inference.

## The Intelligence Layer

[[08-on-device-llm-with-apple-foundation-models]] is the single skill in this category, but it connects deeply across the graph. It provides 100% on-device inference using Apple's Foundation Models framework — no model download, no API key, no data leaves the device.

The LLM engine consumes calendar context from [[12-eventkit-coredata-sync-architecture]] to analyze meeting patterns and generate insights. Its query access is gated by [[07-storekit2-intelligence-based-trial]] which tracks interaction count and pattern discoveries to determine trial maturity. Performance metrics (inference duration, token count — never prompt content) flow to [[10-privacy-first-analytics-architecture]].

The prompt system uses a dynamic gene pool with fitness-weighted selection and feedback-driven evolution — positive feedback increases a gene's fitness, negative feedback decreases it, and low-fitness genes mutate. Safety classification wraps the engine as a decorator, using localized refusal responses from [[16-localization-and-multi-language-patterns]] for harmful or crisis content.

Generated content feeds into [[11-notification-service-with-deep-linking]] for weekly summaries and meeting reminders, and could surface in [[15-widgetkit-and-app-intents-integration]] for Home Screen insights.

## How It Connects

```
12 EventKit-CoreData → 08 On-Device LLM → 11 Notifications
                            ↑                    (generated content)
                       07 StoreKit 2
                       (query limits)
                            ↓
                       10 Analytics
                       (inference metrics)
```
