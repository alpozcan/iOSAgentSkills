---
title: "Fastlane App Store Connect Publishing Pipeline"
description: "End-to-end pipeline for App Store publishing and direct distribution: certificate management, API key auth, fastlane lanes, xcodebuild workflows, ExportOptions plist, notarization, GitHub Releases, Makefile integration, and setup script."
---

# Fastlane App Store Connect Publishing Pipeline

## Context

You need to publish an iOS or macOS app to App Store Connect — creating the app listing, uploading builds to TestFlight, managing localized metadata across dozens of languages, and orchestrating a full release — all from the command line. For macOS, you may also need to distribute directly via GitHub Releases with Developer ID signing and notarization.

Xcode's organizer works for one-off uploads, but it's manual, non-reproducible, and hostile to CI/CD. Fastlane automates the entire pipeline: app registration, build upload, metadata sync, and release — with a single `fastlane release` command.

This skill covers the complete publishing pipeline: certificate creation, App Store Connect API authentication, fastlane lane definitions, xcodebuild archive workflows, direct distribution with notarization, and a setup script for onboarding new team members or CI environments.

## Certificate Landscape

macOS distribution requires **three different certificates**, each for a specific purpose:

| Certificate | Purpose | Required For |
|---|---|---|
| Apple Development | Local development & testing | Xcode debug builds |
| Apple Distribution | App Store / TestFlight | App Store Connect uploads |
| Developer ID Application | Direct distribution (website, GitHub) | DMG/ZIP downloads outside App Store |
| Mac Installer Distribution | Creating `.pkg` installers | App Store `.pkg` packaging (auto-managed by Xcode) |

**Key rule:** An App Store–signed binary **cannot** be opened outside TestFlight/App Store. A Developer ID–signed binary **cannot** be uploaded to App Store Connect. You always need both if distributing through both channels.

## Certificate Creation

### Generate a CSR (Certificate Signing Request)

A single CSR can be reused across multiple certificate types:

```bash
openssl req -new -newkey rsa:2048 -nodes \
  -keyout ~/Desktop/Distribution.key \
  -out ~/Desktop/CertificateSigningRequest.certSigningRequest \
  -subj "/emailAddress=you@example.com/CN=Your Name/C=US"
```

### Import the Private Key into Keychain

The private key must be in Keychain for the certificate to be valid for signing:

```bash
# Convert to p12 (Keychain-compatible format)
openssl pkcs12 -export -legacy \
  -inkey ~/Desktop/Distribution.key \
  -out ~/Desktop/Distribution.p12 \
  -nocerts -passout pass:temppass

# Import into login keychain
security import ~/Desktop/Distribution.p12 \
  -k ~/Library/Keychains/login.keychain-db \
  -P "temppass" \
  -T /usr/bin/codesign \
  -T /usr/bin/productsign
```

### Create Certificates on Apple Developer Portal

1. Go to [Certificates, Identifiers & Profiles](https://developer.apple.com/account/resources/certificates/list)
2. Click **"+"** to create each certificate type
3. Upload the same CSR for each
4. Download the `.cer` file and double-click to install in Keychain

**For Developer ID Application:** Select **G2 Sub-CA** (Xcode 11.4.1 or later) when prompted.

### Verify Certificates

```bash
# List all signing identities
security find-identity -v -p codesigning

# Expected output should include:
# "Apple Development: Your Name (XXXXXXXXXX)"
# "Apple Distribution: Your Name (TEAM_ID)"
# "Developer ID Application: Your Name (TEAM_ID)"
```

## App Store Connect API Key

Create an API key for non-interactive authentication (required for CI and CLI uploads):

1. Go to [App Store Connect → Users and Access → Integrations → Keys](https://appstoreconnect.apple.com/access/integrations/api)
2. Click **"+"**, name it (e.g., "CI"), select **Admin** role
3. Download the `.p8` file (one-time download)
4. Note the **Key ID** and **Issuer ID**

### Store for Fastlane

Store the key config in `fastlane/AuthKey.json`:

```json
{
  "key_id": "XXXXXXXXXX",
  "issuer_id": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx",
  "key_filepath": "AuthKey_XXXXXXXXXX.p8"
}
```

Place the `.p8` file in the `fastlane/` directory. **Both files must be gitignored.**

The `api_key` helper reads this config and falls back to interactive Apple ID auth when the file is missing (useful for first-time setup on a new machine):

```ruby
def api_key
  json_path = File.expand_path("AuthKey.json", __dir__)
  if File.exist?(json_path)
    config = JSON.parse(File.read(json_path))
    app_store_connect_api_key(
      key_id: config["key_id"],
      issuer_id: config["issuer_id"],
      key_filepath: File.expand_path(config["key_filepath"], File.dirname(json_path)),
      in_house: false
    )
  else
    UI.message("No API key found — will use interactive Apple ID auth")
    nil
  end
end
```

### Store for Notarization

```bash
xcrun notarytool store-credentials "notary" \
  --key path/to/AuthKey.p8 \
  --key-id YOUR_KEY_ID \
  --issuer YOUR_ISSUER_ID
```

This stores credentials securely in the system Keychain, referenced later as `--keychain-profile "notary"`.

## Pattern

### Project Structure

```
MyApp/
├── Gemfile                         # Fastlane dependency
├── Gemfile.lock                    # Locked version
├── Makefile                        # One-command workflows
├── ExportOptions-AppStore.plist    # Archive export config
├── fastlane/
│   ├── Appfile                     # App identity
│   ├── Fastfile                    # Lane definitions
│   ├── AuthKey.json                # API key config (gitignored)
│   ├── AuthKey_XXXXXXXXXX.p8       # API key file (gitignored)
│   ├── metadata/
│   │   ├── en-US/
│   │   │   ├── name.txt
│   │   │   ├── subtitle.txt
│   │   │   ├── description.txt
│   │   │   ├── keywords.txt
│   │   │   ├── promotional_text.txt
│   │   │   ├── release_notes.txt
│   │   │   ├── privacy_url.txt
│   │   │   ├── marketing_url.txt
│   │   │   ├── support_url.txt
│   │   │   └── copyright.txt
│   │   ├── de-DE/
│   │   ├── fr-FR/
│   │   ├── ja/
│   │   └── ... (one directory per locale)
│   └── screenshots/
│       ├── en-US/
│       │   ├── 01_feature.png
│       │   └── 02_detail.png
│       └── ...
└── scripts/
    └── setup-asc.sh                # First-time setup wizard
```

### Gemfile

Pin fastlane via Bundler so every developer and CI machine uses the same version:

```ruby
source "https://rubygems.org"

gem "fastlane"
```

Run `bundle install` once to generate `Gemfile.lock`. Commit both files. Always invoke fastlane via `bundle exec fastlane` to use the locked version.

### Appfile

The `Appfile` declares app identity — used as defaults across all lanes:

```ruby
app_identifier("com.company.myapp")   # Bundle ID
apple_id("developer@company.com")     # Apple ID for App Store Connect

itc_team_id("")   # App Store Connect Team ID — auto-detected if only one team
team_id("")       # Developer Portal Team ID — auto-detected if only one team
```

If you belong to multiple teams, fill in the IDs explicitly. Find them at:
- **Team ID**: [Developer Portal → Membership](https://developer.apple.com/account/#/membership)
- **ITC Team ID**: [App Store Connect → Users and Access](https://appstoreconnect.apple.com/access/users)

### Fastfile — Complete Lane Definitions

#### iOS Platform

```ruby
default_platform(:ios)

platform :ios do

  # ─── Auth helper (defined above) ─────────────────────────────
  def api_key
    # ... (see above)
  end

  # ─── Create App on App Store Connect ─────────────────────────

  desc "Register the app on App Store Connect and Developer Portal"
  lane :create_app do
    produce(
      api_key: api_key,
      app_identifier: "com.company.myapp",
      app_name: "MyApp",
      language: "en-US",
      app_version: "1.0.0",
      sku: "myapp-ios-001",
      platform: "ios",
      enable_services: {}
    )

    UI.success("App created on App Store Connect!")
    UI.important("Next: Configure app category, age rating, and analytics in App Store Connect")
  end

  # ─── Register Bundle ID Only ─────────────────────────────────

  desc "Register the bundle ID on the Developer Portal (skip App Store Connect)"
  lane :register_bundle do
    produce(
      api_key: api_key,
      app_identifier: "com.company.myapp",
      app_name: "MyApp",
      language: "en-US",
      skip_itc: true,
      platform: "ios"
    )
    UI.success("Bundle ID registered on Developer Portal!")
  end

  # ─── Build ───────────────────────────────────────────────────

  desc "Archive the app for App Store distribution"
  lane :build do
    build_app(
      project: "MyApp.xcodeproj",        # or workspace: "MyApp.xcworkspace"
      scheme: "MyApp",
      configuration: "Release",
      clean: true,
      output_directory: "./build",
      output_name: "MyApp.ipa",
      export_method: "app-store",
      export_options: {
        signingStyle: "automatic",
        teamID: "YOUR_TEAM_ID"
      }
    )
  end

  # ─── Upload to App Store Connect (TestFlight) ───────────────

  desc "Upload the build to App Store Connect"
  lane :upload do
    upload_to_app_store(
      api_key: api_key,
      app_identifier: "com.company.myapp",
      skip_metadata: true,
      skip_screenshots: true,
      skip_app_version_update: true,
      force: true,
      ipa: "./build/MyApp.ipa",
      run_precheck_before_submit: false,
      precheck_include_in_app_purchases: false
    )
  end

  # ─── Metadata Sync ──────────────────────────────────────────

  desc "Push localized metadata and screenshots to App Store Connect"
  lane :metadata do
    deliver(
      api_key: api_key,
      app_identifier: "com.company.myapp",
      skip_binary_upload: true,
      skip_app_version_update: true,
      force: true,
      submit_for_review: false,
      automatic_release: false
    )
  end

  # ─── Full Release ───────────────────────────────────────────

  desc "Build, upload binary, and sync metadata"
  lane :release do
    build
    upload
    metadata
  end

end
```

#### macOS Platform

For macOS apps, change the platform and use `pkg` instead of `ipa`:

```ruby
default_platform(:mac)

platform :mac do
  # ... same api_key helper ...

  lane :build do
    build_mac_app(
      project: "MyApp.xcodeproj",
      scheme: "MyApp",
      configuration: "Release",
      clean: true,
      output_directory: "./build",
      output_name: "MyApp.app",
      export_method: "app-store"
    )
  end

  lane :upload do
    upload_to_app_store(
      api_key: api_key,
      app_identifier: "com.company.myapp",
      skip_metadata: true,
      skip_screenshots: true,
      skip_app_version_update: true,
      force: true,
      platform: "osx",
      pkg: "./build/appstore/MyApp.pkg",
      run_precheck_before_submit: false,
      precheck_include_in_app_purchases: false
    )
  end

  # metadata and release lanes are identical to iOS
end
```

**Key differences:** macOS uses `build_mac_app` instead of `build_app`, `platform: "osx"` in upload, and `.pkg` instead of `.ipa`.

### Localized Metadata

Fastlane's `deliver` reads metadata from `fastlane/metadata/<locale>/` directories. Each locale directory contains text files that map 1:1 to App Store Connect fields:

| File | App Store Connect Field | Max Length |
|------|------------------------|------------|
| `name.txt` | App Name | 30 chars |
| `subtitle.txt` | Subtitle | 30 chars |
| `description.txt` | Description | 4000 chars |
| `keywords.txt` | Keywords (comma-separated) | 100 chars |
| `promotional_text.txt` | Promotional Text | 170 chars |
| `release_notes.txt` | What's New | 4000 chars |
| `privacy_url.txt` | Privacy Policy URL | — |
| `marketing_url.txt` | Marketing URL | — |
| `support_url.txt` | Support URL | — |
| `copyright.txt` | Copyright | — |

#### Bootstrap metadata for all locales

Download existing metadata from App Store Connect, then edit locally:

```bash
bundle exec fastlane deliver download_metadata
```

To add a new locale, create the directory and populate the text files:

```bash
mkdir -p fastlane/metadata/de-DE
echo "MyApp" > fastlane/metadata/de-DE/name.txt
echo "Your German subtitle" > fastlane/metadata/de-DE/subtitle.txt
# ... repeat for all fields
```

Common locales to support at launch: `en-US`, `de-DE`, `fr-FR`, `es-ES`, `ja`, `ko`, `zh-Hans`, `zh-Hant`, `pt-BR`, `it`, `nl-NL`, `ru`, `tr`, `ar-SA`, `hi`. See [[16-localization-and-multi-language-patterns]] for in-app localization strategies.

#### Screenshots

Place screenshots in `fastlane/screenshots/<locale>/` with filenames sorted in display order:

```
fastlane/screenshots/en-US/
├── 01_onboarding.png
├── 02_main_screen.png
├── 03_detail_view.png
├── 04_settings.png
└── 05_premium.png
```

Required sizes depend on device class (iPhone 6.9", 6.7", 6.5", 5.5"; iPad Pro 13", 12.9"). Use [fastlane frameit](https://docs.fastlane.tools/actions/frameit/) or [screenshots](https://docs.fastlane.tools/actions/capture_screenshots/) to automate capture.

### ExportOptions Plist

For archiving via `xcodebuild` (used by Makefile targets), create `ExportOptions-AppStore.plist`:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN"
  "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>method</key>
    <string>app-store-connect</string>
    <key>teamID</key>
    <string>YOUR_TEAM_ID</string>
    <key>destination</key>
    <string>export</string>
    <key>signingStyle</key>
    <string>automatic</string>
</dict>
</plist>
```

### App Store Submission via xcodebuild

For projects that archive via `xcodebuild` instead of (or alongside) fastlane's `build_app`:

#### 1. Archive

```bash
xcodebuild clean archive \
  -project MyApp.xcodeproj \
  -scheme MyApp \
  -configuration Release \
  -archivePath build/MyApp-AppStore.xcarchive \
  -destination 'generic/platform=macOS' \
  CODE_SIGN_STYLE=Automatic \
  DEVELOPMENT_TEAM=YOUR_TEAM_ID \
  -allowProvisioningUpdates \
  CURRENT_PROJECT_VERSION=$(BUILD_NUMBER)
```

**Important:** Use `CODE_SIGN_STYLE=Automatic` so Xcode manages provisioning profiles. If you get provisioning profile conflicts, clear stale profiles:

```bash
# Find and remove stale profiles
find ~/Library/Developer/Xcode/UserData/Provisioning\ Profiles \
  -name "*.provisionprofile" -delete
```

#### 2. Export as .pkg

```bash
xcodebuild -exportArchive \
  -archivePath build/MyApp-AppStore.xcarchive \
  -exportPath build/appstore \
  -exportOptionsPlist ExportOptions-AppStore.plist \
  -allowProvisioningUpdates
```

#### 3. Upload via Fastlane

```ruby
lane :upload do
  upload_to_app_store(
    api_key: api_key,
    app_identifier: "com.your.app",
    skip_metadata: true,
    skip_screenshots: true,
    skip_app_version_update: true,
    force: true,
    platform: "osx",
    pkg: "./build/appstore/MyApp.pkg",
    run_precheck_before_submit: false,
    precheck_include_in_app_purchases: false
  )
end
```

#### Common Issues

- **ITMS-90238: Invalid signature** — Archive was signed with Development cert instead of Distribution. Ensure the export step re-signs with Apple Distribution.
- **Provisioning profile doesn't include signing certificate** — Delete stale profiles from `~/Library/Developer/Xcode/UserData/Provisioning Profiles/` and re-archive with `-allowProvisioningUpdates`.
- **Conflicting provisioning settings** — Remove any hardcoded `PROVISIONING_PROFILE_SPECIFIER` or `CODE_SIGN_IDENTITY` from the project file; let automatic signing handle it.
- **Build number already used** — Increment `CURRENT_PROJECT_VERSION` for each upload. App Store Connect rejects duplicate build numbers.

### Direct Distribution Pipeline (GitHub Releases)

For macOS apps distributed outside the App Store (website, GitHub Releases), use Developer ID signing and notarization.

#### 1. Archive with Developer ID

```bash
xcodebuild clean archive \
  -project MyApp.xcodeproj \
  -scheme MyApp \
  -configuration Release \
  -archivePath build/MyApp-DevID.xcarchive \
  -destination 'generic/platform=macOS' \
  CODE_SIGN_IDENTITY="Developer ID Application" \
  DEVELOPMENT_TEAM=YOUR_TEAM_ID \
  CODE_SIGN_STYLE=Manual
```

#### 2. Create Distribution Artifacts

```bash
APP_PATH="build/MyApp-DevID.xcarchive/Products/Applications/MyApp.app"

# Create ZIP
ditto -c -k --keepParent "$APP_PATH" build/MyApp-universal.zip

# Create DMG
mkdir -p build/dmg
cp -R "$APP_PATH" build/dmg/
ln -s /Applications build/dmg/Applications
hdiutil create -volname "MyApp" -srcfolder build/dmg \
  -ov -format UDZO build/MyApp-universal.dmg
rm -rf build/dmg
```

#### 3. Notarize

Notarization is **required** for Developer ID apps — without it, Gatekeeper blocks the app.

```bash
# Submit for notarization
xcrun notarytool submit build/MyApp-universal.zip \
  --keychain-profile "notary" \
  --wait

# Staple the ticket to the app (embeds the notarization result)
xcrun stapler staple "$APP_PATH"
```

After stapling, recreate the DMG/ZIP with the stapled app.

#### 4. Upload to GitHub Release

```bash
# Create release and upload assets
gh release create v1.0.0 \
  build/MyApp-1.0.0-universal.dmg \
  build/MyApp-1.0.0-universal.zip \
  --title "v1.0.0" \
  --notes "Release notes here"
```

### Entitlements

Use separate entitlements for Debug (development extras) and Release (App Store clean):

**Debug (`MyApp.entitlements`):**
```xml
<dict>
    <key>com.apple.security.app-sandbox</key>
    <true/>
    <key>com.apple.security.files.user-selected.read-only</key>
    <true/>
    <!-- Development-only entitlements -->
    <key>com.apple.security.temporary-exception.mach-lookup.global-name</key>
    <array>
        <string>com.apple.screencapture.interactive</string>
    </array>
</dict>
```

**Release (`MyApp_Release.entitlements`):**
```xml
<dict>
    <key>com.apple.security.app-sandbox</key>
    <true/>
    <key>com.apple.security.files.user-selected.read-only</key>
    <true/>
</dict>
```

Configure in `project.yml` (XcodeGen):
```yaml
settings:
  configs:
    Debug:
      CODE_SIGN_ENTITLEMENTS: MyApp/MyApp.entitlements
    Release:
      CODE_SIGN_ENTITLEMENTS: MyApp/MyApp_Release.entitlements
```

### Makefile Integration

Add these targets to your project [[18-makefile-for-ios-project-workflows|Makefile]] for one-command workflows:

```makefile
# ─── Configuration ─────────────────────────────────────────────
PROJECT       := MyApp.xcodeproj
SCHEME        := MyApp
TEAM_ID       := YOUR_TEAM_ID
BUNDLE_ID     := com.company.myapp
VERSION       := $(shell grep MARKETING_VERSION project.yml | head -1 | awk -F'"' '{print $$2}')
BUILD_NUMBER  := $(shell date +%Y%m%d%H%M)

# ─── App Store (iOS) ──────────────────────────────────────────

.PHONY: appstore appstore-archive appstore-export appstore-upload

appstore: appstore-archive appstore-export appstore-upload ## Full App Store submission

appstore-archive: ## Archive for App Store
	xcodebuild clean archive \
		-project $(PROJECT) \
		-scheme $(SCHEME) \
		-configuration Release \
		-archivePath build/$(SCHEME)-AppStore.xcarchive \
		-destination 'generic/platform=iOS' \
		CODE_SIGN_STYLE=Automatic \
		DEVELOPMENT_TEAM=$(TEAM_ID) \
		-allowProvisioningUpdates \
		CURRENT_PROJECT_VERSION=$(BUILD_NUMBER)

appstore-export: ## Export .ipa for App Store
	rm -rf build/appstore
	xcodebuild -exportArchive \
		-archivePath build/$(SCHEME)-AppStore.xcarchive \
		-exportPath build/appstore \
		-exportOptionsPlist ExportOptions-AppStore.plist \
		-allowProvisioningUpdates

appstore-upload: ## Upload to App Store Connect via fastlane
	bundle exec fastlane upload

metadata: ## Sync metadata to App Store Connect
	bundle exec fastlane metadata

# ─── App Store (macOS) ─────────────────────────────────────────

.PHONY: appstore-mac appstore-mac-archive appstore-mac-export

appstore-mac: appstore-mac-archive appstore-mac-export appstore-upload ## Full macOS App Store pipeline

appstore-mac-archive: ## Archive for macOS App Store
	xcodebuild clean archive \
		-project $(PROJECT) \
		-scheme $(SCHEME) \
		-configuration Release \
		-archivePath build/$(SCHEME)-AppStore.xcarchive \
		-destination 'generic/platform=macOS' \
		CODE_SIGN_STYLE=Automatic \
		DEVELOPMENT_TEAM=$(TEAM_ID) \
		-allowProvisioningUpdates \
		CURRENT_PROJECT_VERSION=$(BUILD_NUMBER)

appstore-mac-export: ## Export .pkg for macOS App Store
	rm -rf build/appstore
	xcodebuild -exportArchive \
		-archivePath build/$(SCHEME)-AppStore.xcarchive \
		-exportPath build/appstore \
		-exportOptionsPlist ExportOptions-AppStore.plist \
		-allowProvisioningUpdates

# ─── Direct Distribution (GitHub Releases) ────────────────────

.PHONY: github github-archive github-sign github-notarize github-dmg github-release

github: github-archive github-sign github-notarize github-dmg ## Full GitHub release pipeline

github-archive: ## Archive with Developer ID
	xcodebuild clean archive \
		-project $(PROJECT) \
		-scheme $(SCHEME) \
		-configuration Release \
		-archivePath build/$(SCHEME)-DevID.xcarchive \
		-destination 'generic/platform=macOS' \
		CODE_SIGN_IDENTITY="Developer ID Application" \
		DEVELOPMENT_TEAM=$(TEAM_ID) \
		CODE_SIGN_STYLE=Manual

github-sign: ## Copy and verify Developer ID signed app
	rm -rf build/release
	mkdir -p build/release
	cp -R build/$(SCHEME)-DevID.xcarchive/Products/Applications/$(SCHEME).app build/release/
	codesign --verify --deep --strict build/release/$(SCHEME).app
	@echo "✓ Code signature valid"

github-notarize: ## Notarize the app with Apple
	ditto -c -k --keepParent build/release/$(SCHEME).app build/$(SCHEME)-notarize.zip
	xcrun notarytool submit build/$(SCHEME)-notarize.zip \
		--keychain-profile "notary" --wait
	xcrun stapler staple build/release/$(SCHEME).app
	rm build/$(SCHEME)-notarize.zip
	@echo "✓ Notarization complete and stapled"

github-dmg: ## Create DMG and ZIP for distribution
	ditto -c -k --keepParent build/release/$(SCHEME).app \
		build/$(SCHEME)-$(VERSION)-universal.zip
	mkdir -p build/dmg
	cp -R build/release/$(SCHEME).app build/dmg/
	ln -s /Applications build/dmg/Applications
	hdiutil create -volname "$(SCHEME)" -srcfolder build/dmg \
		-ov -format UDZO build/$(SCHEME)-$(VERSION)-universal.dmg
	rm -rf build/dmg
	@echo "✓ DMG and ZIP created"

github-release: github ## Create GitHub release with assets
	gh release create v$(VERSION) \
		build/$(SCHEME)-$(VERSION)-universal.dmg \
		build/$(SCHEME)-$(VERSION)-universal.zip \
		--title "v$(VERSION)" \
		--generate-notes
```

### Setup Script

A first-time setup script helps onboard new developers and configure CI environments:

```bash
#!/bin/bash
set -euo pipefail

cd "$(dirname "$0")/.."

echo ""
echo "╔══════════════════════════════════════════════════╗"
echo "║       App Store Connect — First-Time Setup       ║"
echo "╚══════════════════════════════════════════════════╝"
echo ""

# ─── Auth method selection ─────────────────────────────────────

echo "Choose authentication method:"
echo ""
echo "  1) App Store Connect API Key (.p8)  — recommended for CI"
echo "  2) Apple ID password (interactive)  — simplest for first run"
echo ""
read -rp "Enter 1 or 2: " AUTH_METHOD

if [ "$AUTH_METHOD" = "1" ]; then

    if [ -f "fastlane/AuthKey.json" ]; then
        echo ""
        echo "✅ API key config already exists at fastlane/AuthKey.json"
    else
        echo ""
        echo "To create an API key:"
        echo "  1. Go to https://appstoreconnect.apple.com/access/integrations/api"
        echo "  2. Click 'Generate API Key'"
        echo "  3. Name: 'CI', Role: 'Admin'"
        echo "  4. Download the .p8 file (one-time download!)"
        echo "  5. Note the Key ID and Issuer ID"
        echo ""

        read -rp "Key ID (e.g., XXXXXXXXXX): " KEY_ID
        read -rp "Issuer ID (e.g., xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx): " ISSUER_ID
        read -rp "Path to .p8 file: " P8_PATH

        P8_DEST="fastlane/AuthKey_${KEY_ID}.p8"
        cp "$P8_PATH" "$P8_DEST"
        echo "  Copied .p8 → $P8_DEST"

        cat > fastlane/AuthKey.json <<EOF
{
  "key_id": "${KEY_ID}",
  "issuer_id": "${ISSUER_ID}",
  "key_filepath": "${P8_DEST}"
}
EOF
        echo "  Created fastlane/AuthKey.json"
    fi

    echo ""
    echo "Creating app on App Store Connect..."
    bundle exec fastlane create_app

elif [ "$AUTH_METHOD" = "2" ]; then
    echo ""
    echo "You will be prompted for your Apple ID password and possibly 2FA."
    echo ""
    bundle exec fastlane create_app
else
    echo "❌ Invalid choice. Exiting."
    exit 1
fi

echo ""
echo "════════════════════════════════════════════════════════════"
echo ""
echo "✅ Setup complete! Next steps:"
echo ""
echo "  1. Configure in App Store Connect:"
echo "     • App category and age rating"
echo "     • App Review contact information"
echo "     • Privacy policy URL"
echo ""
echo "  2. Enable analytics (App Store Connect → App → Analytics):"
echo "     ✅ Crashes"
echo "     ✅ Energy Diagnostics"
echo "     ✅ Disk Write Diagnostics"
echo "     ✅ Hang Rate"
echo "     ✅ Launch Time"
echo "     ✅ Memory"
echo ""
echo "  3. Run your first pipeline:"
echo "     make appstore            # full submission pipeline"
echo "     fastlane upload          # upload build only"
echo "     fastlane metadata        # sync metadata only"
echo ""
```

Save as `scripts/setup-asc.sh` and make executable with `chmod +x scripts/setup-asc.sh`.

### .gitignore Entries

Always gitignore sensitive authentication files:

```gitignore
# Fastlane auth — never commit
fastlane/AuthKey*.json
fastlane/AuthKey*.p8
*.p12
*.cer

# Fastlane temp
fastlane/report.xml
fastlane/Preview.html
fastlane/test_output
```

### Pre-submission Checklist

Before first submission:
- [ ] Apple Developer Program membership active ($99/year)
- [ ] Apple Development certificate created and installed
- [ ] Apple Distribution certificate created and installed
- [ ] Developer ID Application certificate created and installed (for GitHub)
- [ ] App Store Connect API key created and stored
- [ ] Notarytool credentials stored in Keychain
- [ ] App created on App Store Connect (`fastlane create_app` or manually)
- [ ] App screenshots uploaded (1280×800 or 2560×1600 for macOS)
- [ ] Privacy policy URL set
- [ ] App Review contact information filled in
- [ ] `ExportOptions-AppStore.plist` created in project root
- [ ] `.gitignore` includes `AuthKey*.json`, `*.p8`, `*.p12`, `*.cer`

## Rules

- **Always use API key auth for CI** — Apple ID auth requires interactive 2FA and session cookies that expire. API keys are stateless and never expire (until revoked).
- **Always invoke via `bundle exec fastlane`** — Running bare `fastlane` uses whatever global version is installed. Bundler ensures every environment runs the same version.
- **Never commit `.p8` files or `AuthKey.json`** — These grant full admin access to your App Store Connect account. Store them in CI secrets (GitHub Actions secrets, etc.) and write them to disk at build time.
- **Increment build number on every upload** — App Store Connect rejects duplicate `CURRENT_PROJECT_VERSION` values. Use a timestamp (`date +%Y%m%d%H%M`) or CI build number.
- **Use `skip_metadata: true` on upload lanes** — Uploading a binary and syncing metadata are separate concerns. Keep them in separate lanes so you can upload a hotfix build without touching metadata.
- **Use `force: true` on `deliver` / `upload_to_app_store`** — Without it, fastlane prompts for interactive confirmation, which breaks CI.
- **Keep metadata in source control** — The `fastlane/metadata/` directory is your source of truth for App Store listings. Review metadata changes in PRs just like code changes.
- **Use `deliver download_metadata` to bootstrap** — Don't manually create metadata files. Download what's already on App Store Connect, then edit locally.
- An App Store–signed binary cannot run outside TestFlight/App Store; a Developer ID–signed binary cannot be uploaded to App Store Connect — always maintain both signing pipelines if distributing through both channels.
- **Notarize all Developer ID builds** — Without notarization, Gatekeeper blocks the app on modern macOS. Always notarize and staple before distributing.
- ❌ **Don't hardcode API key values in the Fastfile** — Always read from a gitignored JSON file or environment variables.
- ❌ **Don't use `deliver` with `submit_for_review: true` in automation without safeguards** — Accidental review submissions waste App Review time and can block your pipeline. Keep submission as a manual step or behind an explicit flag.
- ❌ **Don't skip `run_precheck_before_submit: false` understanding** — Precheck validates metadata locally before upload. Disabling it speeds up uploads but skips validation. Enable it for release lanes, disable for quick TestFlight uploads.
