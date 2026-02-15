# UI Testing for Regression & Smoke Testing

## Context
You're building a modular iOS app with Tuist and need comprehensive UI tests that run on every PR to catch regressions. Tests must cover critical user journeys: navigation, chat, settings, keyboard behavior, and tab switching. The test suite must be fast, deterministic, and maintainable across iOS versions and device sizes.

## Pattern

### Test Base Class with Launch Arguments

Create a shared `WythnosUITestCase` that configures the app for reproducible testing:

```swift
class WythnosUITestCase: XCTestCase {
    var app: XCUIApplication!
    
    /// Override to customize launch arguments per test class
    var launchArguments: [String] { 
        ["--uitesting", "--skip-onboarding", "--pro-debug"] 
    }
    
    override func setUpWithError() throws {
        continueAfterFailure = false
        app = XCUIApplication()
        app.launchArguments = launchArguments
        app.launch()
    }
    
    // MARK: - Navigation Helpers
    
    func tapTab(_ identifier: String) {
        let tab = app.buttons[identifier]
        if tab.waitForExistence(timeout: 5) {
            tab.tap()
            return
        }
        
        // Fallback: find by label
        let labelMap: [String: String] = [
            "tab_insights": "Insights",
            "tab_chat": "W Chat",
            "tab_settings": "Settings"
        ]
        
        if let label = labelMap[identifier] {
            let tabByLabel = app.buttons.matching(NSPredicate(format: "label == %@", label)).firstMatch
            if tabByLabel.waitForExistence(timeout: 3) {
                tabByLabel.tap()
            }
        }
    }
    
    func assertScreenVisible(_ id: String, timeout: TimeInterval = 10) {
        let element = app.otherElements[id]
        XCTAssertTrue(element.waitForExistence(timeout: timeout), "\(id) should be visible")
    }
    
    func waitForText(_ text: String, timeout: TimeInterval = 10) -> Bool {
        let predicate = NSPredicate(format: "label CONTAINS[c] %@", text)
        let element = app.staticTexts.matching(predicate).firstMatch
        return element.waitForExistence(timeout: timeout)
    }
}
```

### Launch Arguments for Test Modes

| Argument | Effect |
|----------|--------|
| `--uitesting` | Minimal mock services, clean state, deterministic data |
| `--skip-onboarding` | Skip onboarding flow, go directly to main app |
| `--pro-debug` | Rich mock data, Pro subscription unlocked |
| `--show-onboarding` | Force onboarding to show even if completed |
| `--locale=tr` | Override locale for localization testing |

### Critical User Journey Tests

**1. App Launch & Navigation**

```swift
func testAppLaunchesSuccessfully() {
    XCTAssertTrue(app.wait(for: .runningForeground, timeout: 10))
}

func testTabNavigation() {
    tapTab("tab_insights")
    XCTAssertTrue(app.otherElements["insights_view"].waitForExistence(timeout: 5))
    
    tapTab("tab_chat")
    XCTAssertTrue(app.buttons["chat_history_button"].waitForExistence(timeout: 5))
    
    tapTab("tab_settings")
    XCTAssertTrue(app.otherElements["settings_view"].waitForExistence(timeout: 5))
}
```

**2. Chat Flow (Critical Path)**

```swift
func testChatSendMessageShowsUserBubble() {
    sleep(3) // Wait for splash screen
    
    let inputField = app.textViews["chat_input_field"]
    XCTAssertTrue(inputField.waitForExistence(timeout: 10))
    
    inputField.tap()
    inputField.typeText("What's on my calendar today?")
    
    let sendButton = app.buttons["chat_send_button"]
    XCTAssertTrue(sendButton.isEnabled)
    sendButton.tap()
    
    // User message must appear
    let userMessage = app.otherElements["chat_message_user_0"]
    XCTAssertTrue(userMessage.waitForExistence(timeout: 10))
    
    // AI response should follow
    let aiMessage = app.otherElements["chat_message_ai_1"]
    XCTAssertTrue(aiMessage.waitForExistence(timeout: 15))
}

func testChatFeedbackButtons() {
    // Send message first
    // ...
    
    // Test feedback buttons work
    let thumbsUp = app.buttons["chat_feedback_positive_1"]
    if thumbsUp.waitForExistence(timeout: 5) {
        thumbsUp.tap() // Should not crash
    }
    
    let thumbsDown = app.buttons["chat_feedback_negative_1"]
    if thumbsDown.waitForExistence(timeout: 5) {
        thumbsDown.tap() // Should not crash
    }
}
```

**3. Chat History**

```swift
func testChatHistoryFlow() {
    // Create a session by sending a message
    let inputField = app.textViews["chat_input_field"]
    inputField.tap()
    inputField.typeText("Test message for history")
    app.buttons["chat_send_button"].tap()
    sleep(2)
    
    // Open history
    app.buttons["chat_history_button"].tap()
    
    // Verify list or empty state
    XCTAssertTrue(
        app.otherElements["chat_history_list"].waitForExistence(timeout: 5) ||
        app.otherElements["chat_history_empty_state"].waitForExistence(timeout: 5)
    )
    
    // If sessions exist, tap one to load it
    let firstSession = app.buttons.matching(NSPredicate(format: "identifier BEGINSWITH 'chat_history_session_'")).firstMatch
    if firstSession.exists {
        firstSession.tap()
        sleep(1)
        XCTAssertTrue(app.buttons["chat_history_button"].exists) // Back on chat
    }
}

func testNewChatButton() {
    // Send message to enable new chat button
    // ...
    
    app.buttons["chat_new_session_button"].tap()
    
    // Should show empty state
    XCTAssertTrue(app.otherElements["chat_empty_state"].waitForExistence(timeout: 5))
}
```

**4. Settings**

```swift
func testSettingsNameField() {
    tapTab("tab_settings")
    
    let nameField = app.textFields["settings_name_field"]
    XCTAssertTrue(nameField.waitForExistence(timeout: 5))
    
    nameField.tap()
    nameField.typeText("Test User")
    app.tap() // Dismiss keyboard
}

func testSettingsPrivacyPolicyLink() {
    tapTab("tab_settings")
    app.swipeUp() // Scroll to privacy section
    
    let privacyPolicy = app.buttons.matching(NSPredicate(format: "label CONTAINS 'Privacy'")).firstMatch
    if privacyPolicy.waitForExistence(timeout: 5) {
        privacyPolicy.tap()
        sleep(2)
        
        // Safari view appears
        let safariDone = app.buttons["Done"]
        if safariDone.waitForExistence(timeout: 3) {
            safariDone.tap()
        }
    }
}

func testSettingsFeedbackButton() {
    tapTab("tab_settings")
    app.swipeUp()
    app.swipeUp()
    
    app.buttons["settings_feedback_button"].tap()
    sleep(2)
    
    // Mail composer or picker appears
    let cancelButton = app.buttons["Cancel"]
    if cancelButton.exists { cancelButton.tap() }
}
```

**5. Keyboard Behavior**

```swift
func testKeyboardShowsAndDismisses() {
    tapTab("tab_chat")
    
    let inputField = app.textViews["chat_input_field"]
    inputField.tap()
    
    let keyboard = app.keyboards.firstMatch
    XCTAssertTrue(keyboard.waitForExistence(timeout: 5))
    
    // Dismiss by tapping scroll view
    app.scrollViews.firstMatch.tap()
    sleep(1)
}

func testKeyboardTabSwitch() {
    tapTab("tab_chat")
    
    let inputField = app.textViews["chat_input_field"]
    inputField.tap()
    sleep(1)
    
    // Switch tab while keyboard open
    tapTab("tab_settings")
    
    XCTAssertTrue(app.otherElements["settings_view"].waitForExistence(timeout: 5))
    XCTAssertFalse(app.keyboards.firstMatch.exists) // Keyboard dismisses
}
```

### Accessibility Identifiers Convention

Use consistent naming across the app:

| Element | Identifier |
|---------|------------|
| Tab buttons | `tab_{analyticsName}` (e.g., `tab_chat`, `tab_settings`) |
| Main views | `{feature}_view` (e.g., `chat_view`, `settings_view`) |
| Input fields | `{feature}_input_field` (e.g., `chat_input_field`) |
| Buttons | `{feature}_{action}_button` (e.g., `chat_send_button`, `chat_history_button`) |
| Messages | `chat_message_{role}_{index}` (e.g., `chat_message_user_0`, `chat_message_ai_1`) |
| Empty states | `{feature}_empty_state` (e.g., `chat_empty_state`) |

### SwiftUI View Identifier Application

```swift
// Views (otherElements)
.background(NyxColors.void)
.accessibilityIdentifier("chat_view")

// Containers with children (use .contain for child visibility)
.accessibilityElement(children: .contain)
.accessibilityIdentifier("chat_input_bar")

// Buttons
.accessibilityIdentifier("chat_send_button")

// Text fields
.accessibilityIdentifier("chat_input_field")
```

### Running Tests

```bash
# Run all UI tests
xcodebuild test -workspace App.xcworkspace -scheme AppUITests \
    -destination 'platform=iOS Simulator,name=iPhone 17 Pro'

# Run specific test class
xcodebuild test -workspace App.xcworkspace -scheme AppUITests \
    -destination 'platform=iOS Simulator,name=iPhone 17 Pro' \
    -only-testing:AppUITests/RegressionTests

# Run single test
xcodebuild test -workspace App.xcworkspace -scheme AppUITests \
    -destination 'platform=iOS Simulator,name=iPhone 17 Pro' \
    -only-testing:AppUITests/RegressionTests/testChatSendMessageShowsUserBubble

# Parallel testing (faster)
xcodebuild test -workspace App.xcworkspace -scheme AppUITests \
    -destination 'platform=iOS Simulator,name=iPhone 17 Pro' \
    -parallel-testing-enabled YES
```

### Makefile Integration

```makefile
# UI Testing
test-ui:
	xcodebuild test -workspace Wythnos.xcworkspace -scheme WythnosUITests \
		-destination 'platform=iOS Simulator,name=iPhone 17 Pro'

test-ui-parallel:
	xcodebuild test -workspace Wythnos.xcworkspace -scheme WythnosUITests \
		-destination 'platform=iOS Simulator,name=iPhone 17 Pro' \
		-parallel-testing-enabled YES \
		-parallel-testing-worker-count 4

test-ui-specific TEST:
	xcodebuild test -workspace Wythnos.xcworkspace -scheme WythnosUITests \
		-destination 'platform=iOS Simulator,name=iPhone 17 Pro' \
		-only-testing:WythnosUITests/$(TEST)

# Smoke test (critical path only)
test-smoke:
	xcodebuild test -workspace Wythnos.xcworkspace -scheme WythnosUITests \
		-destination 'platform=iOS Simulator,name=iPhone 17 Pro' \
		-only-testing:WythnosUITests/RegressionTests/testAppLaunchesSuccessfully \
		-only-testing:WythnosUITests/RegressionTests/testTabNavigation \
		-only-testing:WythnosUITests/RegressionTests/testChatSendMessageShowsUserBubble
```

### CI Integration

```yaml
# GitHub Actions
- name: Run UI Tests
  run: |
    xcodebuild test \
      -workspace Wythnos.xcworkspace \
      -scheme WythnosUITests \
      -destination 'platform=iOS Simulator,name=iPhone 17 Pro,OS=18.2' \
      -resultBundlePath TestResults.xcresult \
      CODE_SIGNING_ALLOWED=NO
      
- name: Upload Test Results
  if: always()
  uses: actions/upload-artifact@v4
  with:
    name: test-results
    path: TestResults.xcresult
```

### Handling Flaky Tests

**1. Use `waitForExistence` instead of `XCTAssertTrue(element.exists)`**

```swift
// Bad - can flake on timing
XCTAssertTrue(app.buttons["send"].exists)

// Good - waits for element
XCTAssertTrue(app.buttons["send"].waitForExistence(timeout: 5))
```

**2. Add appropriate sleep for animations**

```swift
// Splash screen animation
sleep(3) // Wait for 3 pulses + fade

// Keyboard animation
sleep(1) // Wait for keyboard to fully appear
```

**3. Use predicates for flexible matching**

```swift
// Find by partial label match
let button = app.buttons.matching(NSPredicate(format: "label CONTAINS 'Privacy'")).firstMatch

// Find by identifier prefix
let session = app.buttons.matching(NSPredicate(format: "identifier BEGINSWITH 'chat_history_session_'")).firstMatch
```

**4. Handle optional flows gracefully**

```swift
// Don't fail if element doesn't exist in all scenarios
if emptyState.waitForExistence(timeout: 3) {
    // Handle empty state
} else {
    // Handle content state
}
```

## Why This Matters

- **Regression tests catch bugs before merge** - Every PR runs critical path tests
- **Accessibility identifiers enable reliable testing** - Not dependent on localized strings
- **Mock services via launch arguments** - Deterministic, fast, no network calls
- **Parallel testing reduces CI time** - Split tests across simulators
- **Keyboard and tab switching tests** - Common sources of state bugs
- **Feedback button tests** - Ensure core interactions work

## Anti-Patterns

- Don't use `sleep()` excessively - prefer `waitForExistence` with appropriate timeouts
- Don't hardcode localized strings in tests - use accessibility identifiers
- Don't create separate apps for tests - use launch arguments on the same app
- Don't run UI tests on every file change - run on PR, merge, or release
- Don't skip tests on failure - fix the flakiness or the bug
- Don't use `XCTAssertEqual` for UI elements - use existence and visibility checks
- Don't test implementation details - test user-visible behavior