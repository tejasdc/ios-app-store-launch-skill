# TestFlight Internal-only + WebView-shell apps

Two workflows the base skill didn't cover, both of which we hit shipping
`thnkring` on 2026-09-17. Source: `thinkering` session
`10839320-5726-4578-807d-97e8f321b6ea` and follow-up `a258e2da-…`.

## 1. When TestFlight Internal-only is the right target

The base skill goes App Store submission. **TestFlight Internal Testing** is
much lighter — up to 100 devices on your team, no Apple review, unlimited
builds per day. Right choice for:

- Personal apps used only by you and a couple of trusted teammates
- Development shells around a hosted web app (where the "app" is really the URL)
- Pre-launch testing where you don't want a public listing

**What TestFlight Internal doesn't need** (all of these are ONLY required for
External Testing, App Store, or once you promote):

- Screenshots (iPhone or iPad)
- App description / marketing copy
- Age rating declaration
- Privacy questionnaire
- Content rights (App Information page)
- Pricing configuration
- Category assignment
- Review contact info

You still need: a 1024×1024 icon (Apple displays it in TestFlight), a valid
build, a beta group, and testers.

### The lightweight ship path

1. **Reserve the bundle ID via API** — `POST /v1/bundleIds` with
   `{identifier, name, platform: "IOS"}`. Returns the resource id (10-char
   uppercase), which is what you use for later capability calls.

2. **Create the app record** — `POST /v1/apps` returns **403 FORBIDDEN_ERROR**
   with `"The resource 'apps' does not allow 'CREATE'"`. Apple explicitly
   forbids app-record creation via API. Do it in the web UI (the `+` next to
   "Apps") and query the returned app id via `GET /v1/apps?filter[bundleId]=…`.

3. **Sign, archive, upload** — `xcodebuild archive` with
   `-allowProvisioningUpdates -authenticationKeyPath -authenticationKeyID
   -authenticationKeyIssuerID` auto-provisions the Distribution profile via
   the API key. `xcrun altool --upload-app` uploads.

4. **Attach to Internal group** — create a `betaGroup` with
   `isInternalGroup: true`, then `POST
   /v1/betaGroups/{id}/relationships/builds` to attach the new build after
   Apple processing completes (VALID state).

5. **Add a tester** — for team members (Admin/App Manager/Developer), adding
   them via `POST /v1/betaTesters` returns **409 STATE_ERROR** with
   `"Tester(s) cannot be assigned"`. This is because team members can't be
   external testers. Do it in the TestFlight UI: navigate to the group, click
   `+` under Testers, pick from the eligible-team-members list.

## 2. WebView-shell apps (WKWebView wrapping a PWA)

A native shell whose only job is to WKWebView-wrap a hosted web app for iOS.
This unlocks share sheet, widgets, Watch, App Intents, Siri Shortcuts, and
passkeys — all things a home-screen PWA can't do.

### Architecture that works

- SwiftUI `@main` app, one `WKWebView` in a `UIViewRepresentable`
- `WKUserContentController` bridge: JS calls `window.<app>.native.*` methods
  that `postMessage` to Swift; Swift replies via
  `webView.evaluateJavaScript("__bridgeReply(id, ok, payload)")` resolving
  the JS Promise
- `WKAppBoundDomains` in Info.plist restricts to the domain(s) that need
  passkeys / service workers; off-domain navigations fall through to
  `UIApplication.shared.open(...)`
- Widget / Watch / Share Extension / Siri intents: separate Xcode targets in
  the same project, data-shared via App Group. WebView doesn't help here —
  they're always native Swift.

### The safe-area trap (three broken TestFlight builds worth of lessons)

**Do NOT** `.ignoresSafeArea(.all)` the WebView unless the PWA declares
`viewport-fit=cover` AND its top chrome pads with `env(safe-area-inset-top)`.
Most PWAs don't do both, and the shell result is either:

- **PWA content overlaps the iOS status bar** (WebView drew above the safe
  area, PWA had no top padding)
- **White band at the top** (SwiftUI's default window background bleeds
  through above where the WebView stops)

**What works instead:**

- `ZStack { Color(hex).ignoresSafeArea(); WebViewContainer(...) }` — dark
  layer extends to physical edges; WebView respects safe area naturally, like
  Safari does
- Read the PWA's actual body background color at load time
  (`getComputedStyle(document.body).backgroundColor`) and paint the shell
  bands with that exact value — not pure black, not pure white
- `webView.isOpaque = false` + `backgroundColor` matching the PWA so no white
  flash appears during load or pull-to-refresh
- `webView.scrollView.contentInsetAdjustmentBehavior = .never`,
  `showsVerticalScrollIndicator = false`, `bounces = false` — the outer
  WKWebView must not add insets or rubber-band over a background color that
  doesn't match the PWA
- `UIStatusBarStyle: UIStatusBarStyleLightContent` for a dark app so the
  system glyphs stay readable

### Verify visually before every upload

The three broken builds we shipped all "compiled clean." The right gate is a
simulator screenshot + programmatic assertion, run BEFORE `xcodebuild
archive`:

1. Build for Simulator (Release, unsigned)
2. Boot the newest iPhone simulator
3. Install + launch against the production URL
4. Wait for first paint
5. `xcrun simctl io booted screenshot` to PNG
6. Analyze the PNG (PIL / numpy) for:
   - No white/light band at top or bottom
   - Status bar area is dark AND has light glyphs on it (both bands present)
   - PWA top content isn't in the middle 30–70% of the status bar band
   - Bottom tab bar area is dark
7. Exit non-zero on any failure; the shell script gates the archive on this

Wire the check into the upload script; only `SKIP_VERIFY=1` (or the
equivalent) bypasses. This is the only way to catch layout regressions from a
Swift refactor.

### Passkey / WebAuthn from the WebView

Two sides required for `navigator.credentials.get()` inside the WKWebView to
find a passkey for the domain:

1. **Client entitlement:**
   `com.apple.developer.associated-domains` with
   `webcredentials:<domain>` (passkeys) and `applinks:<domain>`
   (Universal Links, since users tap URLs in messages)

2. **Server AASA file** at
   `https://<domain>/.well-known/apple-app-site-association`, served with
   `Content-Type: application/json`, no BOM:

   ```json
   {
     "applinks": {
       "apps": [],
       "details": [{"appIDs": ["<TEAM>.<BUNDLE>"], "components": [{"/": "*"}]}]
     },
     "webcredentials": { "apps": ["<TEAM>.<BUNDLE>"] }
   }
   ```

Register the ASSOCIATED_DOMAINS capability on the bundle id via
`POST /v1/bundleIdCapabilities` with body
`{data:{type:"bundleIdCapabilities", attributes:{capabilityType:"ASSOCIATED_DOMAINS"}, relationships:{bundleId:{data:{type:"bundleIds", id:<resource_id>}}}}}`.
Bundles get IN_APP_PURCHASE automatically; anything else must be POSTed here.

Apple caches AASA aggressively. When you change it, iOS may take a few
minutes to refetch (Settings → Developer → Reset Advanced Website is the
force-refresh escape hatch).

## 3. Xcode 26 gotchas we hit

### `Copy failed` / `rsync: --extended-attributes: unknown option`

Xcode 26's `-exportArchive` invokes rsync internally. On a Mac with Homebrew
rsync 3.4.1 in PATH ahead of `/usr/bin/openrsync`, Homebrew rsync passes an
option openrsync doesn't understand, and export fails with a generic "Copy
failed."

**Fix:** put `/usr/bin` first in PATH before any `xcodebuild` invocation
that produces an IPA:

```bash
export PATH="/usr/bin:/bin:$PATH"
```

This surfaced every time export ran on a fresh Homebrew Mac in Sep 2026.
Watch for it whenever a distribution log mentions rsync.

### iPad orientation validation

Universal builds (`TARGETED_DEVICE_FAMILY = "1,2"`) require all four
orientations in `UISupportedInterfaceOrientations~ipad`, INCLUDING
`UIInterfaceOrientationPortraitUpsideDown`. altool rejects with a 409
otherwise. Fix in Info.plist / project.yml:

```yaml
"UISupportedInterfaceOrientations~ipad":
  - UIInterfaceOrientationPortrait
  - UIInterfaceOrientationPortraitUpsideDown
  - UIInterfaceOrientationLandscapeLeft
  - UIInterfaceOrientationLandscapeRight
```

Or restrict to iPhone-only with `TARGETED_DEVICE_FAMILY: "1"`.

### Python 3.14 breaks system pip

macOS Homebrew Python 3.14 (Sep 2026) ships a `_macos.py` version parser
that crashes on the macOS build strings, so `pip install` fails hard. Use
`uv` or Python 3.12 / 3.13 instead:

```bash
uv venv --python python3.12 /tmp/asc-venv
uv pip install --python /tmp/asc-venv/bin/python PyJWT cryptography requests
```

### `.p8` in a project repo

The base skill's setup — "Save: Key ID, Issuer ID, .p8 file" — makes it easy
to `git add` the .p8 by accident. Add BOTH globally:

```bash
# Global gitignore (~/.gitignore_global)
*.p8
AuthKey_*.p8
```

AND in every project you touch. The key is a JWT signer with Admin scope; a
leaked one lets anyone sign builds and modify listings until you rotate.

## 4. TestFlight auto-update behavior for the user

- **Global "Automatic Updates for New Apps"** toggle only affects apps
  installed AFTER it was turned on. It does NOT retroactively opt in
  existing installed apps.
- **Per-app "Automatic Updates"** toggle on the app's TestFlight page is
  what actually gates auto-install for that app.
- Even with both ON, TestFlight auto-install is an opportunistic background
  task: runs when the phone is locked, on Wi-Fi, ideally charging, and idle.
  Multi-hour lag is normal. For an immediate install, tap Update in the
  TestFlight app.
- Users often complain "auto-update is broken." Ship guidance in your
  runbook so testers know: first build is always a manual tap; after that,
  subsequent builds auto-install once they've flipped the per-app toggle.
