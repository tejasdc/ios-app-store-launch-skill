# API Setup: JWT Auth, Helpers, App Lookup, and Version Management

## Contents
- JWT authentication
- Helper functions (asc_get, asc_post, asc_patch, asc_delete)
- Find your app by bundle ID
- Create or find App Store version
- Wait for build processing and assign to version

---

## JWT Authentication

Every ASC API call requires a JWT signed with your .p8 key. Tokens are valid for max 20 minutes.

```python
#!/usr/bin/env python3
"""
App Store Connect API helper.
Requirements: pip3 install PyJWT cryptography requests
"""

import jwt
import time
import json
import hashlib
import requests
from pathlib import Path

# ============================================================
# CONFIGURATION -- Fill in for your app
# ============================================================
ISSUER_ID = "YOUR_ISSUER_ID"       # From ASC > Users and Access > Integrations > Keys
KEY_ID = "YOUR_KEY_ID"             # Key ID shown next to your .p8 key
KEY_FILE = Path.home() / ".appstoreconnect/private_keys" / f"AuthKey_{KEY_ID}.p8"

ASC_BASE = "https://api.appstoreconnect.apple.com"

def make_token():
    """Generate a fresh JWT for App Store Connect API."""
    private_key = KEY_FILE.read_text()
    now = int(time.time())
    payload = {
        "iss": ISSUER_ID,
        "iat": now,
        "exp": now + 1200,  # 20 minutes max
        "aud": "appstoreconnect-v1"
    }
    return jwt.encode(payload, private_key, algorithm="ES256", headers={"kid": KEY_ID})

def asc_headers():
    """Return authorization headers for ASC API requests."""
    return {
        "Authorization": f"Bearer {make_token()}",
        "Content-Type": "application/json"
    }

def asc_get(path, params=None):
    """GET request to ASC API."""
    r = requests.get(f"{ASC_BASE}{path}", headers=asc_headers(), params=params)
    r.raise_for_status()
    return r.json()

def asc_post(path, data):
    """POST request to ASC API."""
    r = requests.post(f"{ASC_BASE}{path}", headers=asc_headers(), json=data)
    if not r.ok:
        print(f"POST {path} failed: {r.status_code}")
        print(r.text)
    r.raise_for_status()
    return r.json()

def asc_patch(path, data):
    """PATCH request to ASC API."""
    r = requests.patch(f"{ASC_BASE}{path}", headers=asc_headers(), json=data)
    if not r.ok:
        print(f"PATCH {path} failed: {r.status_code}")
        print(r.text)
    r.raise_for_status()
    return r.json()

def asc_delete(path):
    """DELETE request to ASC API."""
    r = requests.delete(f"{ASC_BASE}{path}", headers=asc_headers())
    r.raise_for_status()
```

**Validation:** `asc_get("/v1/apps")` returns 200 with your app list. If 401, check key file path and token expiration.

---

## Find Your App

```python
def find_app(bundle_id):
    """Find app by bundle ID. Returns app ID."""
    data = asc_get("/v1/apps", params={"filter[bundleId]": bundle_id})
    apps = data["data"]
    if not apps:
        raise ValueError(f"No app found with bundle ID: {bundle_id}")
    app = apps[0]
    print(f"Found app: {app['attributes']['name']} (ID: {app['id']})")
    return app["id"]
```

If no app exists yet, create it in App Store Connect first (API: `POST /v1/apps`, requires existing bundle ID registration).

---

## Create or Find App Store Version

```python
def get_or_create_version(app_id, version_string="1.0"):
    """Get existing editable version or create a new one."""
    data = asc_get(f"/v1/apps/{app_id}/appStoreVersions",
                   params={"filter[appStoreState]": "PREPARE_FOR_SUBMISSION,DEVELOPER_REJECTED,REJECTED"})
    versions = data["data"]

    if versions:
        v = versions[0]
        print(f"Found existing version: {v['attributes']['versionString']} (state: {v['attributes']['appStoreState']})")
        return v["id"]

    payload = {
        "data": {
            "type": "appStoreVersions",
            "attributes": {
                "versionString": version_string,
                "platform": "IOS"
            },
            "relationships": {
                "app": {
                    "data": {"type": "apps", "id": app_id}
                }
            }
        }
    }
    data = asc_post("/v1/appStoreVersions", payload)
    version_id = data["data"]["id"]
    print(f"Created version {version_string} (ID: {version_id})")
    return version_id
```

**Validation:** `GET /v1/apps/{id}/appStoreVersions` returns version with state `PREPARE_FOR_SUBMISSION`.

---

## Wait for Build and Assign

After upload, builds take 5-15 minutes to process. Wait for `VALID` state, then assign to version.

```python
def wait_for_build(app_id, build_number, timeout_minutes=30):
    """Wait for a build to finish processing. Returns build ID."""
    import time
    deadline = time.time() + timeout_minutes * 60
    while time.time() < deadline:
        data = asc_get(f"/v1/builds", params={
            "filter[app]": app_id,
            "filter[version]": str(build_number),
            "filter[processingState]": "PROCESSING,VALID,INVALID"
        })
        for build in data["data"]:
            state = build["attributes"]["processingState"]
            print(f"Build {build_number}: {state}")
            if state == "VALID":
                return build["id"]
            elif state == "INVALID":
                raise ValueError(f"Build {build_number} is INVALID -- check ASC for errors")
        time.sleep(30)
    raise TimeoutError(f"Build {build_number} did not finish in {timeout_minutes} minutes")

def assign_build(version_id, build_id):
    """Assign a processed build to a version."""
    payload = {
        "data": {
            "type": "appStoreVersions",
            "id": version_id,
            "relationships": {
                "build": {
                    "data": {"type": "builds", "id": build_id}
                }
            }
        }
    }
    asc_patch(f"/v1/appStoreVersions/{version_id}", payload)
    print(f"Build {build_id} assigned to version {version_id}")
```

**Validation:** `GET /v1/appStoreVersions/{id}?include=build` includes the build in the response.
