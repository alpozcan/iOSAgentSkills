# Makefile as the Single Entry Point for iOS Project Workflows

## Context
A modular iOS project built with Tuist has many moving parts: dependency installation, project generation, building, launching in multiple debug modes (pro, free, onboarding, developer mode), running unit tests, running snapshot tests per module, managing simulators, streaming logs, and resetting app data. Without a single entry point, every team member (and every AI agent) has to remember or look up the exact `xcodebuild` incantation, the right simulator name, the correct `--only-testing` target, and the Tuist lifecycle order. A Makefile solves this — it's the universal CLI that every developer already knows.

## Pattern

### Why Makefile Over Scripts

Shell scripts already exist in `Scripts/` — but they're discovery problems. A Makefile has a single entry point (`make`), self-documents via `make help`, supports tab completion, and composes targets with dependencies (`make fresh` = build + reset + launch). It wraps the scripts rather than replacing them.

### Structure

Organize targets into five groups: Setup & Build, Run Modes, Testing, Utilities, and Internal Helpers.

```makefile
# ============================================================================
# Project — Makefile
# ============================================================================

# Configuration — override at invocation: make run SIMULATOR="iPhone Air"
WORKSPACE     := MyApp.xcworkspace
SCHEME        := MyApp
BUNDLE_ID     := com.company.myapp
SIMULATOR     := iPhone 17 Pro
DESTINATION   := platform=iOS Simulator,name=$(SIMULATOR)
CONFIGURATION := Debug
```

### Default Target: Self-Documenting Help

The bare `make` command should print every available target with descriptions. Use ANSI colors for readability:

```makefile
BOLD  := \033[1m
GREEN := \033[0;32m
BLUE  := \033[0;34m
YELLOW:= \033[1;33m
NC    := \033[0m

.PHONY: help
help:
	@echo ""
	@echo "$(BOLD)MyApp Makefile$(NC)"
	@echo ""
	@echo "$(BOLD)Setup & Build$(NC)"
	@echo "  $(GREEN)make setup$(NC)         Full clean: tuist clean → install → generate"
	@echo "  $(GREEN)make build$(NC)         Build the app (Debug)"
	@echo "  $(GREEN)make clean$(NC)         Clean Tuist cache + DerivedData"
	@echo ""
	@echo "$(BOLD)Run Modes$(NC)"
	@echo "  $(GREEN)make run$(NC)           Pro Debug (skip onboarding)"
	@echo "  $(GREEN)make free$(NC)          Free tier (skip onboarding)"
	@echo "  $(GREEN)make onboarding$(NC)    Pro Debug + show onboarding"
	@echo "  $(GREEN)make fresh$(NC)         Reset data → Pro Debug"
	@echo ""
	@echo "$(BOLD)Testing$(NC)"
	@echo "  $(GREEN)make test$(NC)          Run all unit tests"
	@echo "  $(GREEN)make snapshots$(NC)     Run all snapshot tests"
	@echo ""
	@echo "  SIMULATOR = $(SIMULATOR)"
	@echo "  Override: make run SIMULATOR=\"iPhone Air\""
```

### Setup & Build Targets

The Tuist lifecycle has a strict order: clean → install → generate → build. Encode this in `make setup`:

```makefile
.PHONY: setup
setup:
	tuist clean
	tuist install
	tuist generate --no-open
	@echo "$(GREEN)✅ Setup complete$(NC)"

.PHONY: generate
generate:
	tuist generate --no-open

.PHONY: build
build: _ensure-workspace
	@xcodebuild build \
		-workspace $(WORKSPACE) \
		-scheme $(SCHEME) \
		-destination '$(DESTINATION)' \
		-configuration $(CONFIGURATION) \
		CODE_SIGNING_ALLOWED=NO \
		2>&1 | tail -5

.PHONY: clean
clean:
	-tuist clean 2>/dev/null
	-rm -rf ~/Library/Developer/Xcode/DerivedData/MyApp-*
```

Key details:
- `_ensure-workspace` (see below) auto-runs `tuist generate` if workspace is missing
- `CODE_SIGNING_ALLOWED=NO` avoids signing issues on CI and for agents
- Pipe to `tail -5` to show only the final result, not thousands of compile lines
- Prefix destructive commands with `-` to continue on failure (e.g., DerivedData already gone)

### Run Modes with Launch Arguments

Each run mode builds first (Make dependency), then installs and launches with the right `--flags`:

```makefile
.PHONY: run debug
run debug: build _boot-simulator
	@$(call launch-app,--pro-debug --skip-onboarding)

.PHONY: free
free: build _boot-simulator
	@$(call launch-app,--skip-onboarding)

.PHONY: onboarding
onboarding: build _boot-simulator
	@$(call launch-app,--pro-debug --show-onboarding)

.PHONY: fresh
fresh: build _boot-simulator reset
	@sleep 1
	@$(call launch-app,--pro-debug --skip-onboarding)
```

The `launch-app` function finds the recently-built `.app` in DerivedData, installs it, and launches:

```makefile
define launch-app
	$(eval APP_PATH := $(shell find ~/Library/Developer/Xcode/DerivedData \
		-name "MyApp.app" \
		-path "*/Build/Products/Debug-iphonesimulator/*" \
		-mmin -30 2>/dev/null | head -1))
	@xcrun simctl terminate booted $(BUNDLE_ID) 2>/dev/null || true
	@sleep 0.5
	@xcrun simctl install booted "$(APP_PATH)"
	@xcrun simctl launch booted $(BUNDLE_ID) $(1)
endef
```

### Testing Targets

For unit tests, use `-only-testing:` to explicitly list test targets (avoids running snapshot tests or UI tests accidentally):

```makefile
.PHONY: test
test: _ensure-workspace
	@xcodebuild test \
		-workspace $(WORKSPACE) \
		-scheme $(SCHEME) \
		-destination '$(DESTINATION)' \
		-only-testing:CatalogTests \
		-only-testing:CalendarServiceTests \
		-only-testing:DesignSystemTests \
		2>&1 | grep -E "Test run|error:" | tail -20
```

For snapshot tests, loop over each module's scheme since snapshot test targets are associated with their module schemes:

```makefile
.PHONY: snapshots
snapshots: _ensure-workspace
	@for scheme in DesignSystem ChatUI InsightsUI OnboardingUI SettingsUI TomorrowUI; do \
		echo "  → $${scheme}SnapshotTests"; \
		xcodebuild test \
			-workspace $(WORKSPACE) \
			-scheme $$scheme \
			-destination '$(DESTINATION)' \
			-only-testing:$${scheme}SnapshotTests \
			2>&1 | grep -E "Test run" | tail -1; \
	done

.PHONY: snapshots-record
snapshots-record: _ensure-workspace
	@for scheme in DesignSystem ChatUI InsightsUI OnboardingUI SettingsUI TomorrowUI; do \
		SNAPSHOT_TESTING_RECORD=all xcodebuild test \
			-workspace $(WORKSPACE) \
			-scheme $$scheme \
			-destination '$(DESTINATION)' \
			-only-testing:$${scheme}SnapshotTests \
			2>&1 | grep -E "Test run" | tail -1; \
	done
```

Add per-module targets for fast iteration during development:

```makefile
.PHONY: snapshots-design
snapshots-design: _ensure-workspace
	@xcodebuild test \
		-workspace $(WORKSPACE) \
		-scheme DesignSystem \
		-destination '$(DESTINATION)' \
		-only-testing:DesignSystemSnapshotTests \
		2>&1 | grep -E "passed|failed|Test run" | tail -15
```

### Utility Targets

```makefile
.PHONY: open
open:
	open $(WORKSPACE)

.PHONY: simulator
simulator: _boot-simulator
	@open -a Simulator

.PHONY: kill
kill:
	@xcrun simctl terminate booted $(BUNDLE_ID) 2>/dev/null || true

.PHONY: reset
reset:
	@xcrun simctl terminate booted $(BUNDLE_ID) 2>/dev/null || true
	@sleep 1
	@xcrun simctl uninstall booted $(BUNDLE_ID) 2>/dev/null || true

.PHONY: logs
logs:
	@xcrun simctl spawn booted log stream \
		--level debug \
		--predicate 'subsystem == "$(BUNDLE_ID)"'
```

### Internal Helper Targets

Prefix internal targets with `_` to signal they're not user-facing:

```makefile
.PHONY: _ensure-workspace
_ensure-workspace:
	@if [ ! -d "$(WORKSPACE)" ]; then \
		echo "→ Workspace not found, running tuist generate..."; \
		tuist generate --no-open; \
	fi

.PHONY: _boot-simulator
_boot-simulator:
	@xcrun simctl boot "$(SIMULATOR)" 2>/dev/null || true
	@open -a Simulator 2>/dev/null || true
```

### Simulator Override

The `SIMULATOR` variable can be overridden at invocation without editing the Makefile:

```bash
make run SIMULATOR="iPhone Air"
make snapshots SIMULATOR="iPhone 17 Pro Max"
make build SIMULATOR="iPad Pro 13-inch (M5)"
```

This works because Make substitutes `$(SIMULATOR)` into the `$(DESTINATION)` string.

## Why This Matters

- **Single discovery point**: `make` shows everything — no hunting through Scripts/, README, or wiki
- **Correct dependency ordering**: `make fresh` = build → boot → reset → launch, always in the right order
- **AI agent friendly**: An agent can run `make` to discover all targets, then execute the right one
- **Composable**: `make fresh` chains `build`, `_boot-simulator`, and `reset` via Make dependencies
- **Overridable**: `SIMULATOR=` lets you target any device without editing files
- **CI-compatible**: Same Makefile works locally and in GitHub Actions / Xcode Cloud
- **Self-documenting**: The `help` target doubles as living documentation

## Anti-Patterns

- Don't put the full `xcodebuild` output in the terminal — pipe through `tail -5` or `grep` for signal, not noise
- Don't hardcode the simulator UDID — use the device name and let `xcodebuild` resolve it
- Don't forget `.PHONY` on every target — without it, Make checks for files named `build`, `clean`, etc.
- Don't make `help` a dependency of other targets — it should only run when explicitly invoked or as the default
- Don't duplicate logic between the Makefile and Scripts/ — have the Makefile call the scripts, or replace them entirely
- Don't skip `CODE_SIGNING_ALLOWED=NO` for local builds — it avoids signing configuration headaches for agents and CI
- Don't use `&&` chains inside Make recipes when Make dependencies express the same thing — use target dependencies instead (`fresh: build _boot-simulator reset`)
