# Submission Workflow: Pricing, Privacy, Content Rights, and Review

## Contents
- Pricing setup (API and browser)
- Privacy questionnaire (browser required)
- Content rights (browser required)
- Pre-submission verification script
- Submit for review (3-step API)

---

## Pricing

### Via API (Free apps)

```python
def set_free_pricing(app_id):
    """Set app to Free pricing in all territories."""
    payload = {
        "data": {
            "type": "appPriceSchedules",
            "relationships": {
                "app": {"data": {"type": "apps", "id": app_id}},
                "manualPrices": {"data": [{"type": "appPrices", "id": "${price1}"}]},
                "baseTerritory": {"data": {"type": "territories", "id": "USA"}}
            }
        },
        "included": [{
            "type": "appPrices",
            "id": "${price1}",
            "relationships": {
                "priceTier": {"data": None}  # None = Free
            }
        }]
    }
    try:
        asc_post("/v1/appPriceSchedules", payload)
        print("Pricing set to Free")
    except requests.exceptions.HTTPError as e:
        if "409" in str(e):
            print("Price schedule already exists")
        else:
            raise
```

### Via Browser (simpler alternative)

Navigate to App > Pricing and Availability > set price to Free. Takes 30 seconds.

**Note:** The "Set Up Availability" button is optional. Apps default to all 175 territories.

For paid apps with regional pricing, refer to [Apple's pricing documentation](https://developer.apple.com/documentation/appstoreconnectapi/app_prices_and_availability).

---

## Privacy Questionnaire (Browser Required)

No reliable API exists for this. Use browser automation (Claude-in-Chrome).

### Navigation

Navigate to: `https://appstoreconnect.apple.com/apps/YOUR_APP_ID/distribution/privacy`

**WARNING:** The old URL `/apps/{id}/app-store/privacy` returns 404. Use `/distribution/privacy`.

### Step-by-step for each data type

For each data type your app collects:

1. Select the data type category (Contact Info, Identifiers, Usage Data, etc.)
2. Select specific types within that category
3. For EACH selected type, complete this wizard:
   a. Select purpose (App Functionality, Analytics, Advertising, etc.)
   b. Click "Next" through 2 informational pages about tracking definitions
   c. Answer "Is this data used to track users?" (typically No, unless cross-app tracking)
   d. Answer "Is this data linked to user's identity?" (typically Yes for logged-in apps)
   e. Save

4. After ALL types are configured, click **Publish**

### Common data types for apps with user accounts

- **Email Address** (Contact Info)
- **User ID** (Identifiers)
- **Product Interaction** (Usage Data)
- **Other User Content** (User Content)

### Time estimate

4 data types = ~20 individual clicks, approximately 5-10 minutes.

**Validation:** Privacy page shows "Published" status with all data types listed.

---

## Content Rights (Browser Required)

Content Rights lives under **App Information** (left sidebar), NOT on the version's "Prepare for Submission" page.

### Steps

1. Navigate to App Information page (left sidebar in App Store Connect)
2. Scroll to "Content Rights" section
3. Click "Set Up Content Rights Information"
4. Select "Yes, this app contains, shows, or accesses third-party content"
5. Check the confirmation that your app has the necessary rights
6. Save

**Validation:** Content Rights section shows "Yes" with rights confirmed.

**If you skip this:** Apple blocks submission with: "You must set up Content Rights Information in App Information."

---

## Pre-Submission Verification Script

Run this before submitting to catch missing fields programmatically:

```python
def verify_submission_readiness(app_id, version_id):
    """Check all required fields before submission."""
    issues = []

    # Build assigned?
    v_data = asc_get(f"/v1/appStoreVersions/{version_id}", params={"include": "build"})
    if not v_data.get("included"):
        issues.append("No build assigned to version")

    # Description and supportUrl?
    loc_data = asc_get(f"/v1/appStoreVersions/{version_id}/appStoreVersionLocalizations")
    for loc in loc_data["data"]:
        attrs = loc["attributes"]
        if not attrs.get("description"):
            issues.append(f"Missing description for {attrs['locale']}")
        if not attrs.get("supportUrl"):
            issues.append(f"Missing supportUrl for {attrs['locale']}")

    # Screenshots?
    for loc in loc_data["data"]:
        sets_data = asc_get(
            f"/v1/appStoreVersionLocalizations/{loc['id']}/appScreenshotSets"
        )
        types = [s["attributes"]["screenshotDisplayType"] for s in sets_data["data"]]
        if "APP_IPHONE_67" not in types and "APP_IPHONE_65" not in types:
            issues.append("Missing iPhone screenshots")
        if "APP_IPAD_PRO_129" not in types and "APP_IPAD_PRO_3GEN_129" not in types:
            issues.append("Missing iPad Pro 12.9\" screenshots")

    # Age rating?
    info_data = asc_get(f"/v1/apps/{app_id}/appInfos")
    app_info_id = info_data["data"][0]["id"]
    try:
        asc_get(f"/v1/appInfos/{app_info_id}/ageRatingDeclaration")
    except Exception:
        issues.append("Age rating not set")

    # Review contact?
    try:
        review_data = asc_get(f"/v1/appStoreVersions/{version_id}/appStoreReviewDetail")
        if not review_data.get("data"):
            issues.append("Review contact not set")
    except Exception:
        issues.append("Review contact not set")

    if issues:
        print("SUBMISSION BLOCKERS:")
        for i, issue in enumerate(issues, 1):
            print(f"  {i}. {issue}")
        return False
    print("All checks passed -- ready to submit!")
    return True
```

---

## Submit for Review (3-Step API)

**CRITICAL: Use `reviewSubmissions`, NOT `appStoreVersionSubmissions` (deprecated, returns 403).**

```python
def submit_for_review(app_id, version_id):
    """Submit app for review. 3-step process."""
    # Step 1: Create submission
    submission = asc_post("/v1/reviewSubmissions", {
        "data": {
            "type": "reviewSubmissions",
            "relationships": {
                "app": {"data": {"type": "apps", "id": app_id}}
            }
        }
    })
    submission_id = submission["data"]["id"]

    # Step 2: Add version as item
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
    print(f"App submitted for review! Submission ID: {submission_id}")
    return submission_id
```

### Browser Fallback for Submission

If the API submission fails (409 due to stale drafts):
1. Navigate to the version page in App Store Connect
2. Click "Add for Review"
3. Click "Create New Submission" (bypasses stale drafts from failed API attempts)
4. Click "Submit for Review"

### Post-Submission

- **Status:** Changes to "Waiting for Review"
- **Review time:** 24-48 hours for first submission, sometimes same-day for updates
- **Auto-release:** If enabled, app goes live immediately on approval
- **Expedited review:** `https://developer.apple.com/contact/app-store/?topic=expedite` (web form only, for emergencies)
