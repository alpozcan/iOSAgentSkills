---
title: "Local Notification Service with LLM-Generated Content and Deep Linking"
description: "Actor-based notification service scheduling AI-generated local notifications (weekly summaries, meeting reminders). Deep linking to 11 native video apps, engagement-gated permission requests, and smart meeting scoring."
---

# Local Notification Service with LLM-Generated Content and Deep Linking

## Context
You need a notification system that sends contextual, AI-generated local push notifications (weekly summaries, meeting reminders with video call join buttons, AI insight milestones). Notifications must support deep linking to in-app screens and native video call apps (Zoom, Meet, Teams).

## Pattern

### Notification Service as Actor

```swift
public actor NotificationService: NotificationServiceProtocol {
    private let notificationCenter: UNUserNotificationCenter
    private let contentGenerator: NotificationContentGeneratorProtocol
    private let calendarStore: CalendarStoreProtocol  // see [[12-eventkit-coredata-sync-architecture]]
    private let deepLinkHandler: DeepLinkHandlerProtocol
    private let preferencesStore: NotificationPreferencesStore
    
    // Recurring weekly summary (e.g., Sunday 6 PM)
    public func scheduleWeeklySummary() async throws {
        let prefs = await preferences
        guard prefs.weeklySummaryEnabled else { return }
        
        var components = DateComponents()
        components.weekday = prefs.weeklySummaryDay
        components.hour = prefs.weeklySummaryHour
        
        let events = try await calendarStore.fetchEvents(from: weekAgo, to: now)
        let content = try await contentGenerator.generateWeeklySummary(events: events, locale: .current)
        
        let trigger = UNCalendarNotificationTrigger(dateMatching: components, repeats: true)
        try await notificationCenter.add(createRequest(id: "weekly_summary", content: content, trigger: trigger))
    }
    
    // Meeting reminders with video join deep links
    public func scheduleMeetingReminder(for event: CalendarEvent) async throws {
        let prefs = await preferences
        guard prefs.meetingRemindersEnabled else { return }
        
        // Skip if no video link and user only wants video meetings
        let meetingLink = DeepLinkHandler.extractMeetingLink(from: event)
        if prefs.onlyWithVideoLinks && meetingLink == nil { return }
        
        // Schedule X minutes before
        guard let triggerDate = Calendar.current.date(
            byAdding: .minute, value: -prefs.meetingReminderMinutes, to: event.startDate
        ), triggerDate > Date() else { return }
        
        let content = try await contentGenerator.generateMeetingReminder(
            event: event, minutesBefore: prefs.meetingReminderMinutes, locale: .current
        )
        
        let trigger = UNCalendarNotificationTrigger(
            dateMatching: Calendar.current.dateComponents([.year, .month, .day, .hour, .minute], from: triggerDate),
            repeats: false
        )
        
        try await notificationCenter.add(createRequest(
            id: "meeting_\(event.id)", content: content, trigger: trigger,
            type: .meetingReminder, deepLink: meetingLink
        ))
    }
}
```

### Deep Link Handler with Platform Detection

```swift
public enum MeetingPlatform: String, Sendable, CaseIterable {
    case zoom, googleMeet, microsoftTeams, webex, skype, jitsi, slackHuddle, discord
    
    public var urlScheme: String {
        switch self {
        case .zoom: return "zoomus://"
        case .googleMeet: return "com.google.meet://"
        case .microsoftTeams: return "msteams://"
        // ...
        }
    }
}

public struct DeepLinkHandler: DeepLinkHandlerProtocol {
    // Pattern-based detection across 11 platforms
    private static let meetingPatterns: [(MeetingPlatform, [String])] = [
        (.zoom, ["zoom.us/j/", "zoom.us/s/", "zoom.us/my/"]),
        (.googleMeet, ["meet.google.com/"]),
        (.microsoftTeams, ["teams.microsoft.com/l/meetup-join"]),
        // ...
    ]
    
    // Extract meeting link from event location or notes
    public static func extractMeetingLink(from event: CalendarEvent) -> URL? {
        let text = [event.location, event.notes].compactMap { $0 }.joined(separator: " ")
        let detector = try? NSDataDetector(types: NSTextCheckingResult.CheckingType.link.rawValue)
        let matches = detector?.matches(in: text, range: NSRange(location: 0, length: text.utf16.count))
        
        for match in matches ?? [] {
            if let url = match.url {
                for (_, patterns) in meetingPatterns {
                    if patterns.contains(where: { url.absoluteString.contains($0) }) {
                        return convertToNativeScheme(url)
                    }
                }
            }
        }
        return nil
    }
    
    // Convert web URLs to native app schemes
    private static func convertToNativeScheme(_ url: URL) -> URL {
        if let platform = detectPlatform(from: url) {
            switch platform {
            case .zoom:
                if let meetingID = extractZoomMeetingID(from: url) {
                    return URL(string: "zoomus://zoom.us/join?confno=\(meetingID)") ?? url
                }
            case .microsoftTeams:
                return URL(string: url.absoluteString.replacingOccurrences(of: "https://", with: "msteams://")) ?? url
            default: break
            }
        }
        return url
    }
}
```

> **Security note:** The `meetingPatterns` are hardcoded string literals, which is intentional — they should never be constructed from user input. The `NSDataDetector` approach for URL extraction is safe because it uses Apple's built-in URL detection rather than custom regex, avoiding ReDoS vulnerabilities. However, validate extracted URLs before opening them — a malicious calendar event could contain a `javascript:` or `data:` URL scheme.

### Configurable Platform Patterns

Extract hardcoded URL patterns into a configurable constant for easier updates:

```swift
public struct MeetingPlatformConfig: Sendable {
    public static let `default` = MeetingPlatformConfig(platforms: [
        MeetingPlatformPattern(platform: .zoom, patterns: ["zoom.us/j/", "zoom.us/s/", "zoom.us/my/"]),
        MeetingPlatformPattern(platform: .googleMeet, patterns: ["meet.google.com/"]),
        MeetingPlatformPattern(platform: .microsoftTeams, patterns: ["teams.microsoft.com/l/meetup-join"]),
        // ... other platforms
    ])

    public let platforms: [MeetingPlatformPattern]
}
```

### Notification Preferences with Smart Defaults

```swift
public struct NotificationPreferences: Codable, Sendable {
    public var weeklySummaryEnabled: Bool        // true
    public var weeklySummaryDay: Int             // 1 (Sunday)
    public var weeklySummaryHour: Int            // 18
    
    public var meetingRemindersEnabled: Bool     // true
    public var meetingReminderMinutes: Int       // 10
    public var onlyWithVideoLinks: Bool          // true (don't notify for in-person meetings)
    
    public var insightNotificationsEnabled: Bool // true
    public var insightNotificationFrequency: InsightFrequency // .smart
    
    public enum InsightFrequency: String, Codable, Sendable {
        case smart   // ML-driven, contextual
        case daily   // Max 1/day
        case weekly  // Max 1/week
        case off
    }
}
```

### Permission Request Timing (Engagement-Gated)

Don't request notification permission on first launch. Wait until the user has demonstrated engagement:

```swift
// EngagementTracker determines when to ask for permission:
// - 7+ app opens, OR
// - User performed a calendar modification, OR
// - User viewed 3+ insights

// In WythnosApp:
private func setupNotifications() async {
    let status = await notificationService.getAuthorizationStatus()
    guard status == .authorized || status == .provisional else { return }
    // Only schedule if already authorized — don't request here
    try? await notificationService.scheduleWeeklySummary()
    try? await notificationService.scheduleWeekPreview()
}
```

### Smart Meeting Importance Scoring

```swift
public struct SmartMeetingDetector {
    public static func importanceScore(for event: CalendarEvent) -> Int {
        var score = 0
        if DeepLinkHandler.extractMeetingLink(from: event) != nil { score += 10 }
        let importantKeywords = ["interview", "review", "presentation", "demo", "client"]
        if importantKeywords.contains(where: { event.title.lowercased().contains($0) }) { score += 5 }
        if event.durationMinutes > 60 { score += 3 }
        if event.location != nil { score += 2 }
        return score
    }
}
```

## Edge Cases

- **Meeting link false positives:** The keyword-based meeting detection can match non-meeting URLs (e.g., a blog post about Zoom at `zoom.us/blog/...`). Mitigate by checking URL path structure — meeting URLs typically contain numeric IDs or specific path segments (`/j/`, `/l/meetup-join`).
- **Native app not installed (web URL fallback):** When converting to a native scheme (e.g., `zoomus://`), check if the app can open the URL with `UIApplication.shared.canOpenURL()` before attempting. Fall back to the original HTTPS URL if the app isn't installed.
- **64 notification limit:** iOS limits apps to 64 pending local notifications. For users with many meetings, prioritize by importance score and batch schedule within the limit. Remove expired notifications before scheduling new ones.
- **Timezone changes:** If the user travels across timezones, previously scheduled notification triggers may fire at wrong local times. Re-schedule all notifications on `NSSystemTimeZoneDidChange` notification or on app foreground.
- **All-day event reminder timing:** All-day events have a `startDate` at midnight. Scheduling a reminder 10 minutes before midnight is rarely useful. Skip all-day events for meeting reminders or use a custom time (e.g., 8 AM on the event day).
- **URL scheme validation:** Before opening extracted URLs, validate the scheme is in an allowlist (`https`, `zoomus`, `msteams`, etc.). Never open `javascript:`, `data:`, or `file:` URLs from calendar event content — this prevents potential URL scheme attacks.

## Why This Matters

- **[[08-on-device-llm-with-apple-foundation-models|LLM-generated]] notification content** is personalized and contextual, not templated
- **Deep linking to native video apps** (Zoom, Meet, Teams) lets users join meetings from the notification
- **Engagement-gated permission requests** result in higher opt-in rates than asking on first launch
- **[[06-actor-based-concurrency-patterns|`actor` isolation]]** prevents race conditions when scheduling/canceling notifications concurrently
- **Preferences persistence** via `Codable` + `UserDefaults` with smart defaults

## Anti-Patterns

- Don't request notification permission on first launch — gate it behind engagement metrics
- Don't hardcode notification content — generate it contextually with the LLM
- Don't schedule all-day events for meeting reminders
- Don't schedule reminders for past events — always check `triggerDate > Date()`
- Don't forget to re-supply dependent services when mocking for tests
- Don't open URLs extracted from calendar events without validating the scheme — reject `javascript:`, `data:`, and `file:` schemes
- Don't schedule more than 64 notifications — prioritize by importance score and clean up expired ones
