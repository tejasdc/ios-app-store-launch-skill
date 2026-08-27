# Gotchas, Errors, and Battle-Tested Lessons

## Contents
- API error reference table
- Browser automation pitfalls
- Lessons from real submissions
- Category IDs

---

## API Error Reference

| Error | Endpoint | Root Cause | Fix |
|-------|----------|-----------|-----|
| `403: "The specified resource does not allow this request"` | `POST /v1/appStoreVersionSubmissions` | Endpoint DEPRECATED, only DELETE allowed | Use `POST /v1/reviewSubmissions` (3-step flow) |
| `409 Conflict` | Any PATCH | Stale data or type mismatch in attributes | Re-GET the resource; check string vs boolean types |
| `403: "This request is not allowed"` | Any write endpoint | JWT expired or wrong role | Regenerate token; ensure Admin or App Manager role |
| `404 Not Found` | `/v1/appInfos/{id}/ageRatingDeclaration` | Declaration not created yet | Create a version first -- it auto-creates the declaration |
| `422: "screenshot dimensions"` | `PATCH /v1/appScreenshots/{id}` | PNG dimensions don't match display type | Verify: `sips -g pixelWidth -g pixelHeight file.png` |
| `ENTITY_ERROR: uploaded is invalid` | `PATCH /v1/appScreenshots/{id}` | MD5 checksum mismatch | Recompute: `hashlib.md5(open(f,'rb').read()).hexdigest()` |
| `Build not assignable` | `PATCH /v1/appStoreVersions/{id}` | Build still processing or flagged invalid | Wait for `processingState: VALID`; check ASC web UI |
| `Cannot create version` | `POST /v1/appStoreVersions` | Editable version already exists | Use `get_or_create_version()` to check first |

---

## Browser Automation Pitfalls

### Content Rights is NOT on the version page

Content Rights lives under **App Information** (left sidebar), not the version's "Prepare for Submission" page. Apple blocks submission with: "You must set up Content Rights Information in App Information."

### Sign-in required blocks submission if credentials are empty

If "Sign-in required" is checked but username/password fields are blank, Apple blocks submission. Either:
- Provide demo credentials (preferred -- create test account for reviewers)
- Uncheck "Sign-in required" and explain in Notes: "App requires an existing account"

### Privacy questionnaire has hidden sub-steps

For EACH data type selected:
1. Select purpose (App Functionality, Analytics, etc.)
2. "Next" through 2 informational pages about tracking definitions
3. "Is this data used to track users?" (typically No)
4. "Is this data linked to user identity?" (typically Yes)
5. Save

4 data types = ~20 clicks, not 4.

### ASC URL changes

Some URLs use new path structure. If you get 404s:
- Privacy: `/apps/{id}/distribution/privacy` (not `/app-store/privacy`)
- Version: `/apps/{id}/distribution/ios/version/inflight`

---

## Lessons from Real Submissions

### "Newer Build Available" warning is safe to ignore

When submitting after multiple builds, Apple warns about newer builds. Just submit -- the warning is informational only, and it uses the build you assigned.

### Stale draft submissions accumulate

API-created submissions that failed show as "Draft iOS Submission" entries in the dropdown. They return 409 on cancel via API. **Workaround:** Always choose "Create New Submission" to bypass.

### App Availability setup is optional

The "Set Up Availability" button on the Pricing page is optional. Apps default to all 175 territories. Only use it to RESTRICT availability.

### Phone number in review contact

Use a real number, not 555-xxx. Apple reviewers may call during review.

### iPad screenshots required for universal apps

Required unless you explicitly set `TARGETED_DEVICE_FAMILY: "1"` (iPhone-only) in your build settings. Upload at least 2 iPad screenshots (2048x2732).

### Age rating "unrestricted web access" = 17+

If your app has a WKWebView with unrestricted URLs, age rating becomes 17+ in most regions. This is expected behavior -- it will not cause rejection.

### Build processing takes time

Builds take 5-30 minutes to process after upload. Poll `GET /v1/builds?filter[app]={appId}` for status. If stuck over 1 hour, re-upload.

### Expedited review requests

For emergency bug fixes after submission:
```
https://developer.apple.com/contact/app-store/?topic=expedite
```
Web form only -- no API endpoint for this.

---

## Category IDs

Common `primaryCategoryId` / `secondaryCategoryId` values:

```
BUSINESS            DEVELOPER_TOOLS     EDUCATION
ENTERTAINMENT       FINANCE             FOOD_AND_DRINK
GAMES               GRAPHICS_AND_DESIGN HEALTH_AND_FITNESS
LIFESTYLE           MEDICAL             MUSIC
NAVIGATION          NEWS                PHOTO_AND_VIDEO
PRODUCTIVITY        REFERENCE           SHOPPING
SOCIAL_NETWORKING   SPORTS              TRAVEL
UTILITIES           WEATHER
```

Full list available via: `GET /v1/appCategories`
