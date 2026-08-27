---
name: ios-app-store-launch
description: Launches an iOS app on the App Store from Xcode project to live listing via CLI and App Store Connect API. Use when user says "submit to App Store", "publish my iOS app", "launch on App Store", "upload build to Apple", "set up App Store listing", "submit app for review", or "update my App Store app". Covers build signing, metadata, screenshots, privacy questionnaire, pricing, and review submission. Do NOT use for Android/Google Play submissions or general Xcode build troubleshooting.
---

# iOS App Store Launch

Take any iOS app from Xcode project to live on the App Store using CLI and the App Store Connect API. Browser automation (Claude-in-Chrome) needed only for: privacy questionnaire, content rights, and final submission confirmation.

Battle-tested on a real submission (Cortex iOS, 2026-03-04) -- zero to "Waiting for Review" in one session.

## Prerequisites

```
[ ] Apple Developer account ($99/year, developer.apple.com)
[ ] App Store Connect API key (.p8) with Admin or App Manager role
    - Create at: https://appstoreconnect.apple.com/access/integrations/api
    - Save: Key ID, Issuer ID, .p8 file
[ ] Xcode with CLI tools (xcode-select --install)
[ ] PyJWT: pip3 install PyJWT cryptography requests
[ ] App icon in Assets.xcassets (1024x1024)
[ ] Screenshots ready OR simulator available
[ ] Optional: brew install --cask transporter (upload fallback)
```

Store API key:
```bash
mkdir -p ~/.appstoreconnect/private_keys
cp ~/Downloads/AuthKey_XXXXXXXXXX.p8 ~/.appstoreconnect/private_keys/
```

## Workflow Overview

| Phase | What | Method | Reference |
|-------|------|--------|-----------|
| 1 | Build, sign, upload | xcodebuild CLI | [build-and-upload.md](references/build-and-upload.md) |
| 2 | API setup + JWT auth | Python | [api-setup.md](references/api-setup.md) |
| 3 | Create version, assign build | ASC API | [api-setup.md](references/api-setup.md) |
| 4 | Metadata (description, keywords) | ASC API | [metadata-and-assets.md](references/metadata-and-assets.md) |
| 5 | Age rating, categories, app info | ASC API | [metadata-and-assets.md](references/metadata-and-assets.md) |
| 6 | Screenshots (3-step upload) | ASC API | [metadata-and-assets.md](references/metadata-and-assets.md) |
| 7 | Pricing, review contact | ASC API or browser | [submission-workflow.md](references/submission-workflow.md) |
| 8 | Privacy questionnaire | Browser only | [submission-workflow.md](references/submission-workflow.md) |
| 9 | Content rights | Browser only | [submission-workflow.md](references/submission-workflow.md) |
| 10 | Submit for review | ASC API (3-step) | [submission-workflow.md](references/submission-workflow.md) |

**Validation:** After each phase, verify success before proceeding. Each reference file includes validation criteria.

## Critical Gotchas

These cause the most wasted time. Read before starting:

1. **Use `reviewSubmissions` API, NOT `appStoreVersionSubmissions`** -- the old endpoint is deprecated and returns 403. The correct flow is 3 steps: create submission, add items, confirm.

2. **Content Rights is on App Information page**, not the version page. Apple blocks submission if missing.

3. **Sign-in required blocks submission if credentials are empty.** Either provide demo credentials OR uncheck and explain in Notes.

4. **iPad screenshots required** even for iPhone-focused apps, unless `TARGETED_DEVICE_FAMILY` is `"1"`.

5. **Age rating field types are mixed** -- some are string enums (`"NONE"`) and some are booleans. Wrong types cause silent 409 errors.

6. **Privacy page URL changed**: use `/apps/{id}/distribution/privacy` (not `/app-store/privacy`).

7. **Stale draft submissions** accumulate and return 409 on cancel. Always use "Create New Submission" to bypass.

For the full gotchas list, see [gotchas.md](references/gotchas.md).

## Submit for Review (Phase 10)

**CRITICAL: Use `reviewSubmissions`, NOT `appStoreVersionSubmissions`.**

```python
# 3-step submission process:

# Step 1: Create review submission
submission = asc_post("/v1/reviewSubmissions", {
    "data": {
        "type": "reviewSubmissions",
        "relationships": {
            "app": {"data": {"type": "apps", "id": app_id}}
        }
    }
})
submission_id = submission["data"]["id"]

# Step 2: Add version as submission item
asc_post("/v1/reviewSubmissionItems", {
    "data": {
        "type": "reviewSubmissionItems",
        "relationships": {
            "reviewSubmission": {"data": {"type": "reviewSubmissions", "id": submission_id}},
            "appStoreVersion": {"data": {"type": "appStoreVersions", "id": version_id}}
        }
    }
})

# Step 3: Confirm submission
asc_patch(f"/v1/reviewSubmissions/{submission_id}", {
    "data": {
        "type": "reviewSubmissions",
        "id": submission_id,
        "attributes": {"submitted": True}
    }
})
```

**Validation:** App status changes to "Waiting for Review".

## Pre-Submission Checklist

Before submitting, verify all of these:
- Build assigned and in VALID state
- Description and supportUrl set for en-US
- iPhone screenshots uploaded (APP_IPHONE_67 minimum)
- iPad screenshots uploaded (required unless iPhone-only build)
- Age rating declaration complete (all 24 fields)
- Review contact info set (real phone number)
- Pricing configured
- Privacy questionnaire published
- Content Rights set (under App Information, not the version page)

Run `verify_submission_readiness()` from [submission-workflow.md](references/submission-workflow.md) to check programmatically.

## Update Workflow (v1.1+)

For subsequent versions, the process is shorter:

1. Bump build number, rebuild, and upload (Phase 1)
2. Create new version via API: `POST /v1/appStoreVersions`
3. Wait for build processing, assign to version
4. Set `whatsNew` in version localization (REQUIRED for updates)
5. Screenshots carry forward -- only update if changed
6. Submit via 3-step `reviewSubmissions` flow

```python
version_id = get_or_create_version(app_id, "1.1")
build_id = wait_for_build(app_id, "2")
assign_build(version_id, build_id)
set_version_metadata(version_id, {"whatsNew": "Bug fixes and improvements."})
submit_for_review(app_id, version_id)
```

## Typical Timeline

| Phase | Duration |
|-------|----------|
| Build + upload | 5-15 min |
| Build processing (Apple side) | 5-15 min |
| API metadata setup | 5-10 min |
| Privacy questionnaire (browser) | 5-10 min |
| Pricing + content rights | 1-2 min |
| App Review (Apple) | 24-48 hours (first), sometimes same-day |

**Total hands-on: ~30-45 min first submission, ~10 min for updates.**

## Expedited Review

For emergency bug fixes: `https://developer.apple.com/contact/app-store/?topic=expedite` (web form only).

## Examples

### Example 1: First-time submission of a new free app

User says: "Submit my iOS app to the App Store for the first time"

1. Verify prerequisites (API key, Xcode, etc.)
2. Run Phases 1-10 sequentially
3. Set pricing to Free
4. Submit for review

Result: App enters "Waiting for Review" state.

### Example 2: Submitting a version update

User says: "Push a new version of my app to the App Store"

1. Bump build number, rebuild, upload (Phase 1)
2. Create new version, assign build (Phases 2-3)
3. Set `whatsNew` field (Phase 4)
4. Submit for review (Phase 10)

Result: New version enters review queue. Screenshots carry forward.

## Reference Index

| File | Contents |
|------|----------|
| [build-and-upload.md](references/build-and-upload.md) | exportOptions.plist, xcodebuild commands, Transporter fallback, build errors |
| [api-setup.md](references/api-setup.md) | JWT auth, helper functions (asc_get/post/patch/delete), find app, create version, assign build |
| [metadata-and-assets.md](references/metadata-and-assets.md) | Localization, app info, categories, age rating, screenshot upload, review contact |
| [submission-workflow.md](references/submission-workflow.md) | Pricing, privacy questionnaire, content rights, pre-flight verification, submit for review |
| [gotchas.md](references/gotchas.md) | All battle-tested lessons, API error reference table, browser automation pitfalls |
