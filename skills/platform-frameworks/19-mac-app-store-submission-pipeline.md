# macOS App Store Submission Pipeline

## Context
You need to submit a macOS app to the App Store and also distribute it directly via GitHub Releases. This involves creating certificates, provisioning profiles, code signing, notarization, and uploading — all from the command line. Apple enforces separate signing identities for App Store vs. direct distribution, and each channel has its own workflow.

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

Create `fastlane/AuthKey.json`:

```json
{
  "key_id": "YOUR_KEY_ID",
  "issuer_id": "your-issuer-uuid",
  "key_filepath": "AuthKey_YOUR_KEY_ID.p8"
}
```

Place the `.p8` file in the `fastlane/` directory. Ensure both are in `.gitignore`.

### Fastlane API Key Helper

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

## App Store Submission Pipeline

### 1. Archive

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

### 2. Export as .pkg

Create `ExportOptions-AppStore.plist`:

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

```bash
xcodebuild -exportArchive \
  -archivePath build/MyApp-AppStore.xcarchive \
  -exportPath build/appstore \
  -exportOptionsPlist ExportOptions-AppStore.plist \
  -allowProvisioningUpdates
```

### 3. Upload via Fastlane

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

### Common Issues

- **ITMS-90238: Invalid signature** — Archive was signed with Development cert instead of Distribution. Ensure the export step re-signs with Apple Distribution.
- **Provisioning profile doesn't include signing certificate** — Delete stale profiles from `~/Library/Developer/Xcode/UserData/Provisioning Profiles/` and re-archive with `-allowProvisioningUpdates`.
- **Conflicting provisioning settings** — Remove any hardcoded `PROVISIONING_PROFILE_SPECIFIER` or `CODE_SIGN_IDENTITY` from the project file; let automatic signing handle it.
- **Build number already used** — Increment `CURRENT_PROJECT_VERSION` for each upload. App Store Connect rejects duplicate build numbers.

## Direct Distribution Pipeline (GitHub Releases)

### 1. Archive with Developer ID

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

### 2. Create Distribution Artifacts

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

### 3. Notarize

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

### 4. Upload to GitHub Release

```bash
# Create release and upload assets
gh release create v1.0.0 \
  build/MyApp-1.0.0-universal.dmg \
  build/MyApp-1.0.0-universal.zip \
  --title "v1.0.0" \
  --notes "Release notes here"
```

## Makefile Integration

Add these targets to your project Makefile for one-command workflows:

```makefile
# ─── Configuration ─────────────────────────────────────────────
PROJECT       := MyApp.xcodeproj
SCHEME        := MyApp
TEAM_ID       := YOUR_TEAM_ID
BUNDLE_ID     := com.your.app
VERSION       := $(shell grep MARKETING_VERSION project.yml | head -1 | awk -F'"' '{print $$2}')
BUILD_NUMBER  := $(shell date +%Y%m%d%H%M)

# ─── App Store ─────────────────────────────────────────────────

.PHONY: appstore appstore-archive appstore-export appstore-upload

appstore: appstore-archive appstore-export appstore-upload ## Full App Store pipeline

appstore-archive: ## Archive for App Store
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

appstore-export: ## Export .pkg for App Store
	rm -rf build/appstore
	xcodebuild -exportArchive \
		-archivePath build/$(SCHEME)-AppStore.xcarchive \
		-exportPath build/appstore \
		-exportOptionsPlist ExportOptions-AppStore.plist \
		-allowProvisioningUpdates

appstore-upload: ## Upload .pkg to App Store Connect via fastlane
	fastlane upload

# ─── Direct Distribution ──────────────────────────────────────

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

## Entitlements

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

## Checklist

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
