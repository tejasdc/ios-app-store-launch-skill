# Metadata, Assets, and Configuration

## Contents
- Version metadata (description, keywords, localization)
- App-level info (categories, copyright, privacy URL)
- Age rating declaration (mixed field types)
- Screenshots (3-step upload, dimensions table, simulator generation)
- Review contact info

---

## Version Metadata (Localization)

```python
def set_version_metadata(version_id, metadata):
    """
    Set version localization metadata.

    metadata keys:
      - description (required, 4000 chars max)
      - keywords (optional, 100 chars max, comma-separated)
      - whatsNew (required for v1.1+ updates)
      - promotionalText (optional, 170 chars, updateable without new version)
      - supportUrl (required)
      - marketingUrl (optional)
    """
    data = asc_get(f"/v1/appStoreVersions/{version_id}/appStoreVersionLocalizations")
    localizations = data["data"]

    en_loc = next((loc for loc in localizations if loc["attributes"]["locale"] == "en-US"), None)

    if not en_loc:
        payload = {
            "data": {
                "type": "appStoreVersionLocalizations",
                "attributes": {"locale": "en-US", **metadata},
                "relationships": {
                    "appStoreVersion": {
                        "data": {"type": "appStoreVersions", "id": version_id}
                    }
                }
            }
        }
        data = asc_post("/v1/appStoreVersionLocalizations", payload)
        loc_id = data["data"]["id"]
    else:
        loc_id = en_loc["id"]
        payload = {
            "data": {
                "type": "appStoreVersionLocalizations",
                "id": loc_id,
                "attributes": metadata
            }
        }
        asc_patch(f"/v1/appStoreVersionLocalizations/{loc_id}", payload)

    print(f"Set metadata for en-US (ID: {loc_id})")
    return loc_id
```

**Validation:** `GET /v1/appStoreVersionLocalizations/{id}` returns all fields populated.

---

## App-Level Info

```python
def set_app_info(app_id, categories=None, copyright_text=None, privacy_url=None):
    """
    Set app-level info (categories, copyright, privacy policy URL).

    Common category IDs:
      DEVELOPER_TOOLS, PRODUCTIVITY, UTILITIES, BUSINESS, EDUCATION,
      ENTERTAINMENT, FINANCE, HEALTH_AND_FITNESS, LIFESTYLE,
      SOCIAL_NETWORKING, REFERENCE, TRAVEL, WEATHER

    Full list: GET /v1/appCategories
    """
    data = asc_get(f"/v1/apps/{app_id}/appInfos")
    app_info = data["data"][0]
    app_info_id = app_info["id"]

    if categories:
        cat_payload = {
            "data": {
                "type": "appInfos",
                "id": app_info_id,
                "relationships": {}
            }
        }
        if "primaryCategoryId" in categories:
            cat_payload["data"]["relationships"]["primaryCategory"] = {
                "data": {"type": "appCategories", "id": categories["primaryCategoryId"]}
            }
        if "secondaryCategoryId" in categories:
            cat_payload["data"]["relationships"]["secondaryCategory"] = {
                "data": {"type": "appCategories", "id": categories["secondaryCategoryId"]}
            }
        asc_patch(f"/v1/appInfos/{app_info_id}", cat_payload)
        print("Updated categories")

    loc_data = asc_get(f"/v1/appInfos/{app_info_id}/appInfoLocalizations")
    en_info_loc = next((loc for loc in loc_data["data"] if loc["attributes"]["locale"] == "en-US"), None)

    if en_info_loc and privacy_url:
        asc_patch(f"/v1/appInfoLocalizations/{en_info_loc['id']}", {
            "data": {
                "type": "appInfoLocalizations",
                "id": en_info_loc["id"],
                "attributes": {"privacyPolicyUrl": privacy_url}
            }
        })
        print("Updated app info localization")

    return app_info_id
```

**Validation:** App shows correct categories and privacy URL in App Store Connect.

---

## Age Rating Declaration

Age rating has ~24 fields with **mixed types**. String enums: `"NONE"`, `"INFREQUENT_OR_MILD"`, `"FREQUENT_OR_INTENSE"`. Booleans: `True`/`False`. Mixing types causes silent 409 errors.

```python
def set_age_rating(app_info_id):
    """Set age rating. Adjust values for your app -- these are conservative defaults."""
    data = asc_get(f"/v1/appInfos/{app_info_id}/ageRatingDeclaration")
    declaration_id = data["data"]["id"]

    payload = {
        "data": {
            "type": "ageRatingDeclarations",
            "id": declaration_id,
            "attributes": {
                # STRING ENUM FIELDS: "NONE", "INFREQUENT_OR_MILD", "FREQUENT_OR_INTENSE"
                "alcoholTobaccoOrDrugUseOrReferences": "NONE",
                "contests": "NONE",
                "gamblingSimulated": "NONE",
                "horrorOrFearThemes": "NONE",
                "matureOrSuggestiveThemes": "NONE",
                "medicalOrTreatmentInformation": "NONE",
                "profanityOrCrudeHumor": "NONE",
                "sexualContentGraphicAndNudity": "NONE",
                "sexualContentOrNudity": "NONE",
                "violenceCartoonOrFantasy": "NONE",
                "violenceRealistic": "NONE",
                "violenceRealisticProlongedGraphicOrSadistic": "NONE",

                # BOOLEAN FIELDS (do NOT pass strings for these)
                "gamblingAndContests": False,
                "gambling": False,
                "ageRatingOverride": None,      # None = let Apple calculate
                "koreaAgeRatingOverride": None,
                "unrestrictedWebAccess": False,  # True = 17+ rating
                "seventeenPlus": False,
            }
        }
    }

    asc_patch(f"/v1/ageRatingDeclarations/{declaration_id}", payload)
    print(f"Age rating updated (ID: {declaration_id})")
```

**Key notes:**
- `unrestrictedWebAccess: True` (WKWebView with arbitrary URLs) bumps rating to 17+
- For WebView wrappers loading only your own site, use `False`
- `gamblingAndContests` and `gambling` are booleans despite similar fields being string enums
- Some fields may not exist in older API versions -- remove unknown attributes on error

**Validation:** `GET /v1/appInfos/{id}/ageRatingDeclaration` returns populated declaration.

---

## Screenshots

3-step upload: reserve slot, upload binary chunks, commit with MD5 checksum.

### Required Dimensions

| Display Type | API Value | Portrait | Landscape |
|---|---|---|---|
| iPhone 6.7" (14/15 Pro Max) | `APP_IPHONE_67` | 1290 x 2796 | 2796 x 1290 |
| iPhone 6.5" (11 Pro Max) | `APP_IPHONE_65` | 1242 x 2688 | 2688 x 1242 |
| iPhone 5.5" (6s/7/8 Plus) | `APP_IPHONE_55` | 1242 x 2208 | 2208 x 1242 |
| iPad Pro 12.9" (3rd gen+) | `APP_IPAD_PRO_129` | 2048 x 2732 | 2732 x 2048 |

**Minimum required:** `APP_IPHONE_67` + `APP_IPAD_PRO_129`. Apple auto-scales to smaller devices.

**6.5" and 6.7" have DIFFERENT dimensions -- do NOT reuse images between them.**

iPad screenshots are required even for iPhone-focused apps unless `TARGETED_DEVICE_FAMILY: "1"`.

### Generate from Simulator

```bash
xcrun simctl boot "iPhone 15 Pro Max"
xcrun simctl boot "iPad Pro (12.9-inch) (6th generation)"

# List available simulators if names don't match:
xcrun simctl list devices available | grep -E "iPhone|iPad"

# For native apps:
xcrun simctl install booted build/YourApp.app
xcrun simctl launch booted com.yourcompany.yourapp

# Capture:
xcrun simctl io booted screenshot ~/Desktop/screenshot_iphone67_1.png

# Verify dimensions:
sips -g pixelWidth -g pixelHeight ~/Desktop/screenshot_iphone67_1.png
```

### Upload via API

```python
def upload_screenshot(localization_id, display_type, file_path, position=1):
    """Upload a screenshot. 3-step: reserve -> upload chunks -> commit."""
    file_path = Path(file_path)
    file_size = file_path.stat().st_size
    file_name = file_path.name

    # Get or create screenshot set
    sets_data = asc_get(
        f"/v1/appStoreVersionLocalizations/{localization_id}/appScreenshotSets"
    )
    screenshot_set_id = next(
        (s["id"] for s in sets_data["data"]
         if s["attributes"]["screenshotDisplayType"] == display_type),
        None
    )

    if not screenshot_set_id:
        set_data = asc_post("/v1/appScreenshotSets", {
            "data": {
                "type": "appScreenshotSets",
                "attributes": {"screenshotDisplayType": display_type},
                "relationships": {
                    "appStoreVersionLocalization": {
                        "data": {"type": "appStoreVersionLocalizations", "id": localization_id}
                    }
                }
            }
        })
        screenshot_set_id = set_data["data"]["id"]

    # Step 1: Reserve
    reserve_data = asc_post("/v1/appScreenshots", {
        "data": {
            "type": "appScreenshots",
            "attributes": {"fileName": file_name, "fileSize": file_size},
            "relationships": {
                "appScreenshotSet": {
                    "data": {"type": "appScreenshotSets", "id": screenshot_set_id}
                }
            }
        }
    })
    screenshot_id = reserve_data["data"]["id"]
    upload_ops = reserve_data["data"]["attributes"]["uploadOperations"]

    # Step 2: Upload binary chunks
    file_bytes = file_path.read_bytes()
    for op in upload_ops:
        chunk = file_bytes[op["offset"]:op["offset"] + op["length"]]
        headers_list = {h["name"]: h["value"] for h in op["requestHeaders"]}
        requests.put(op["url"], headers=headers_list, data=chunk).raise_for_status()

    # Step 3: Commit with MD5
    md5_hash = hashlib.md5(file_bytes).hexdigest()
    asc_patch(f"/v1/appScreenshots/{screenshot_id}", {
        "data": {
            "type": "appScreenshots",
            "id": screenshot_id,
            "attributes": {"uploaded": True, "sourceFileChecksum": md5_hash}
        }
    })
    print(f"Uploaded {file_name} to {display_type}")
    return screenshot_id


def upload_all_screenshots(localization_id, screenshots):
    """Upload multiple screenshots. screenshots: list of {display_type, file_path}."""
    by_type = {}
    for s in screenshots:
        by_type.setdefault(s["display_type"], []).append(s["file_path"])
    for display_type, paths in by_type.items():
        for i, path in enumerate(paths, 1):
            upload_screenshot(localization_id, display_type, path, position=i)
```

**Validation:** Each screenshot shows `assetDeliveryState.state: "COMPLETE"`.

### Screenshot Errors

| Error | Fix |
|-------|-----|
| `ENTITY_ERROR.ATTRIBUTE.INVALID: fileSize` | File too large (max 8MB) or invalid PNG |
| `409 Conflict` on set creation | Screenshot set already exists -- use existing one |
| Wrong dimensions after upload | Dimensions must exactly match display type specs |
| "Processing" indefinitely | MD5 checksum mismatch -- recompute on exact uploaded bytes |

---

## Review Contact

```python
def set_review_info(version_id, contact_info):
    """
    Set App Review contact. Required for first submission.

    contact_info keys: contactFirstName, contactLastName, contactEmail,
    contactPhone (with country code, e.g., "+15551234567"),
    demoAccountName (optional), demoAccountPassword (optional), notes (optional)
    """
    data = asc_get(f"/v1/appStoreVersions/{version_id}/appStoreReviewDetail")

    if data.get("data"):
        detail_id = data["data"]["id"]
        asc_patch(f"/v1/appStoreReviewDetails/{detail_id}", {
            "data": {
                "type": "appStoreReviewDetails",
                "id": detail_id,
                "attributes": contact_info
            }
        })
    else:
        asc_post("/v1/appStoreReviewDetails", {
            "data": {
                "type": "appStoreReviewDetails",
                "attributes": contact_info,
                "relationships": {
                    "appStoreVersion": {
                        "data": {"type": "appStoreVersions", "id": version_id}
                    }
                }
            }
        })
    print("Review contact info set")
```

**Use a real phone number** -- Apple reviewers may call during review. No 555 placeholders.

**Validation:** `GET /v1/appStoreVersions/{id}/appStoreReviewDetail` returns contact data.
