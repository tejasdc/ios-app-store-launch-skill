# Iterating on devices: TestFlight uploads vs direct installs from a Mac

Two valid ways to get a build onto your own iPhone and Apple Watch. TestFlight is the durable,
shareable known-good build; a direct install from a Mac is the fast loop. Choose per change;
don't send every fix through TestFlight. Source: thnkr.ing, September 21, 2026 (its runbook
`docs/runbooks/ios-shell.md` holds the project specifics).

## TestFlight uploads

- **Apple caps uploads per app per rolling day and does not publish the number.** Apple's upload
  help page names no limit. The refusal is `409 … Upload limit reached. The upload limit for your
  application has been reached. Please wait 1 day and try again.` (altool STATE_ERROR.VALIDATION_ERROR).
  Observed: 23 uploads accepted in about 19 hours, then refused. Forum reports range from 7 to 19,
  with no Apple staff answer. Builds Apple rejected after upload appear to count. A slot frees
  about 24 hours after the oldest upload in the window. Waiting is the only fix; nothing suggests
  any risk to the account or TestFlight access.
- **So one uploader batches.** When several agents or sessions ship to the same app, exactly one
  owns uploads. The others push to the main branch and hand over their commit. Batch at most a
  few uploads a day, hours apart. Count this app's uploads from `GET /v1/apps/{id}/buildUploads`
  before assuming a slot is free.
- **Acceptance is not installable.** Apple can reject after a clean upload. For example, ITMS-90626
  rejected builds because an App Intent's title or description contained "iPhone". Poll
  `buildUploads` or the build's processing state until it is installable or rejected, and report
  the reason.
- TestFlight is the only way onto a Watch that lacks Developer Mode, and the build to leave
  installed overnight.

## Direct install from a Mac (development-signed)

- Build with automatic signing and an App Store Connect API key (`-allowProvisioningUpdates
  -allowProvisioningDeviceRegistration`). Install with `xcrun devicectl device install app`. It
  takes minutes, has no cap, and works over Wi-Fi once the device is paired (cable plus Trust the
  first time).
- **The build destination and devicectl name devices differently.** `xcodebuild -destination
  id=` needs the hardware UDID. `devicectl --device` takes the CoreDevice identifier. Read the UDID
  from `devicectl device info details`.
- **It replaces the TestFlight copy** (same bundle ID). Going back means reinstalling from TestFlight.
- **Push environment.** A development-signed app needs `aps-environment` set to `development` and
  registers tokens with Apple's sandbox push service. Put the value in a build setting
  (`$(VAR)` inside the entitlements file) instead of hard-coding `production`. The production
  service answers a sandbox token `400 BadDeviceToken`. A server that treats that as
  "unregistered" silently forgets the device. Retry once on the other service and remember
  which one worked.
- **Apple Watch needs Developer Mode, and the option only appears after Xcode has connected to
  the Watch.** Settings → Privacy & Security on the Watch shows nothing until then. Xcode reaches
  the Watch through its paired iPhone. A stale Mac-side pairing can make the Watch refuse
  connections (CoreDevice `RemotePairingError 1007`). An unpaired Watch is not discoverable
  until Xcode's Devices window drives the pairing.
- **Xcode must support the paired iPhone's iOS version.** Xcode 26.3 installed apps on an iOS 27
  phone but could not reach the Watch through it. The fix was Xcode 27, which requires macOS
  26.6 or later.
- **A development install can remove the Watch app** when the Watch can't take the development
  copy (no Developer Mode). The next TestFlight install restores it.
- **Keep the old Xcode beside the new one.** The App Store treats a renamed Xcode as installed
  and won't add a second copy. Keep an APFS clone (`cp -cR`, near-zero space) as
  `Xcode-<old>.app` and let the App Store upgrade `Xcode.app`. Pin scripts to a specific copy
  with `DEVELOPER_DIR`. Installing from the App Store needs the owner's password.

## macOS updates that won't install

- A Mac can stay stuck on an old macOS. Software Update repeatedly failed with
  `SUMacControllerError 7703` ("The available software updates have changed… asset reload not
  found"), recorded under `DDMPersistedErrorKey` in `/Library/Preferences/com.apple.SoftwareUpdate`.
  It discarded a download made with `softwareupdate -d`.
- The full installer takes a separate path and worked: `softwareupdate --list-full-installers`,
  then `softwareupdate --fetch-full-installer --full-installer-version <v>` (no sudo), then run
  "Install macOS …" from Applications (owner password). Check the staged version inside
  `SharedSupport.dmg` (`OSVersion`/`Build` in `com_apple_MobileAsset_MacSoftwareUpdate.xml`)
  before telling anyone it's ready. Leave about 25 GB free.

## Choosing

| Change | Path |
| --- | --- |
| Hosted web content only (web-shell apps) | Neither: deploy the server |
| Native app code, trying it now | Direct install |
| Watch app, Watch without Developer Mode | TestFlight |
| Push / Live Activity behaviour | Either, if the server handles both push environments |
| The build to leave installed, or to share | TestFlight |
