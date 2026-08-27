# Build, Sign, and Upload

## Contents
- Create exportOptions.plist
- Archive and upload commands
- Verify upload
- Transporter fallback
- Common build errors

---

## Create exportOptions.plist

Create in project root. The `destination: upload` key makes `-exportArchive` upload directly to App Store Connect.

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>method</key>
    <string>app-store-connect</string>
    <key>teamID</key>
    <string>YOUR_TEAM_ID</string>
    <key>signingStyle</key>
    <string>automatic</string>
    <key>uploadBitcode</key>
    <false/>
    <key>uploadSymbols</key>
    <true/>
    <key>destination</key>
    <string>upload</string>
</dict>
</plist>
```

Without `destination: upload`, you get a local `.ipa` and must upload separately via Transporter.

## Archive and Upload

```bash
# Step 1: Archive
xcodebuild archive \
  -project YourApp.xcodeproj \
  -scheme YourApp \
  -configuration Release \
  -archivePath build/YourApp.xcarchive \
  -destination 'generic/platform=iOS' \
  CODE_SIGN_STYLE=Automatic \
  DEVELOPMENT_TEAM=YOUR_TEAM_ID \
  -allowProvisioningUpdates

# Step 2: Export + Upload (one command does both)
xcodebuild -exportArchive \
  -archivePath build/YourApp.xcarchive \
  -exportPath build/export \
  -exportOptionsPlist exportOptions.plist \
  -allowProvisioningUpdates
```

**Variants:**
- If using xcodegen: run `xcodegen generate` before archive to regenerate `.xcodeproj`
- If using a workspace (CocoaPods, SPM): replace `-project` with `-workspace YourApp.xcworkspace`

**Validation:** Output shows `** EXPORT SUCCEEDED **`. Build appears in App Store Connect within 5-15 minutes.

## Verify Upload

After successful export, the build goes through Apple's processing pipeline (validation, binary analysis). Check status:
- **App Store Connect web UI:** TestFlight > Builds
- **API:** `GET /v1/builds?filter[app]={appId}` -- look for `processingState: VALID`

Processing typically takes 5-15 minutes. If stuck over 1 hour, re-upload.

## Transporter Fallback

If `xcodebuild -exportArchive` upload fails (auth issues, network timeouts), use Apple's Transporter:

```bash
brew install --cask transporter
```

Drag the `.ipa` file onto Transporter. It handles authentication via Apple ID.

To get a local `.ipa` instead of uploading: remove the `destination: upload` line from exportOptions.plist, then export normally. The `.ipa` will be in `build/export/`.

## Common Build Errors

| Error | Fix |
|-------|-----|
| `No profiles for 'com.example.app' were found` | Add `-allowProvisioningUpdates` and set `DEVELOPMENT_TEAM` |
| `exportArchive: The data couldn't be read` | Check exportOptions.plist syntax and team ID |
| `Code signing is required for product type 'Application'` | Set `CODE_SIGN_STYLE=Automatic` and `DEVELOPMENT_TEAM=...` |
| `Provisioning profile doesn't include signing certificate` | Open Xcode once to refresh certs, or revoke/recreate in portal |
| Upload fails with auth error | With `destination: upload`, uses Xcode API key settings. Without it, needs app-specific password |
