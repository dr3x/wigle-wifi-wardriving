# Custom Upload Server — Implementation Guide

## Overview

This fork of WiGLE WiFi Wardriving adds two settings that redirect capture uploads to a home server instead of (or in addition to) WiGLE's API. The changes are minimal and isolated: the WiGLE identity/authentication path is untouched. When the custom URL field is blank the app behaves identically to stock WiGLE.

---

## Decisions Made

### File format: WiGLE CSV v1.6

The app already serializes every scan to WiGLE CSV v1.6 before uploading. No alternative format was introduced because:

- The serialization code (`ObservationUploader.writeFileWithCursor`) is battle-tested and produces a well-defined schema.
- The file is already on-device before the upload call is made — changing format would require a parallel serializer.
- CSV is human-readable, easily imported into SQLite/Postgres/InfluxDB, and the column layout is fully documented below.

The file is sent as a **multipart/form-data POST**, field name `file`, identical to what WiGLE's API accepts.

### Authentication: Bearer token

WiGLE uses HTTP Basic Auth (`Authorization: Basic base64(authname:apiToken)`). For a home server behind a Cloudflare Tunnel, Bearer token is the right choice:

- Cloudflare Access supports `Authorization: Bearer <token>` via service tokens or a simple origin rule.
- Avoids sending a username in the credential, which has no meaning on a private server.
- Simple to rotate: update the pref on the phone and the secret on Cloudflare.

When a custom token is set, the app adds `Authorization: Bearer <token>` directly to the request and skips the WiGLE Basic Auth interceptor entirely. The two auth paths never mix.

### Where the decision happens

`WiGLEApiManager.upload()` reads both prefs at call time (not at construction time), so changing the URL or token in settings takes effect on the next upload without restarting the app.

---

## App Changes

### Files modified

| File | Change |
|---|---|
| `wiglewifiwardriving/src/main/java/net/wigle/wigleandroid/util/PreferenceKeys.java` | Added `PREF_CUSTOM_UPLOAD_URL` and `PREF_CUSTOM_UPLOAD_TOKEN` constants |
| `wiglewifiwardriving/src/main/res/layout/settings.xml` | New "Custom Upload Server" section with two text fields |
| `wiglewifiwardriving/src/main/res/values/strings.xml` | Three new string resources for the section |
| `wiglewifiwardriving/src/main/java/net/wigle/wigleandroid/SettingsFragment.java` | Wires the two new fields to SharedPreferences |
| `wiglewifiwardriving/src/main/java/net/wigle/wigleandroid/net/WiGLEApiManager.java` | `upload()` reads custom URL/token and overrides request URL and auth header |

### Preference keys

```
customUploadUrl    — full HTTPS URL, e.g. https://scans.home.example.com/upload
customUploadToken  — raw Bearer token string (stored in SharedPreferences, not encrypted)
```

Both are stored in the app's standard `WiglePrefs` SharedPreferences file.

### Behaviour matrix

| customUploadUrl | customUploadToken | Result |
|---|---|---|
| blank | blank | Posts to `https://api.wigle.net/api/v2/file/upload` with WiGLE Basic Auth (stock behaviour) |
| set | blank | Posts to custom URL with no auth header |
| set | set | Posts to custom URL with `Authorization: Bearer <token>` |
| blank | set | Posts to WiGLE URL with `Authorization: Bearer <token>` (unusual — probably not useful) |

---

## Upload Request Specification

The app sends an HTTP POST to the configured URL with:

```
Content-Type: multipart/form-data; boundary=<generated>
Authorization: Bearer <token>          (when token is set)
User-Agent: WigleWifi (<java version info>)
```

### Multipart fields

| Field name | Value |
|---|---|
| `file` | The WiGLE CSV v1.6 file, `Content-Type: application/octet-stream` |
| `donate` | `"on"` — only present if the user has the "donate" checkbox checked in WiGLE settings |

### WiGLE CSV v1.6 format

**Line 1 — file header (comma-separated):**

```
WigleWifi-1.6,appRelease=<versionName>,model=<device model>,release=<android version>,
device=<device codename>,display=<build display>,board=<board>,brand=<brand>,
star=Sol,body=3,subBody=0
```

**Line 2 — column header:**

```
MAC,SSID,AuthMode,FirstSeen,Channel,Frequency,RSSI,CurrentLatitude,CurrentLongitude,AltitudeMeters,AccuracyMeters,RCOIs,MfgrId,Type
```

**Lines 3+ — one observation per line:**

| Column | Type | Example | Notes |
|---|---|---|---|
| MAC | string | `AA:BB:CC:DD:EE:FF` | BSSID for WiFi; cell/BT identifiers for other types |
| SSID | string | `MyNetwork` | May contain Unicode; quoted if it contains commas |
| AuthMode | string | `[WPA2-PSK-CCMP][ESS]` | Android capabilities string |
| FirstSeen | datetime | `2024-06-25 14:23:01` | UTC, `yyyy-MM-dd HH:mm:ss` |
| Channel | int | `6` | WiFi channel number; may be blank |
| Frequency | int | `2437` | MHz; may be blank |
| RSSI | int | `-67` | dBm |
| CurrentLatitude | decimal | `47.606209` | WGS84 |
| CurrentLongitude | decimal | `-122.332071` | WGS84 |
| AltitudeMeters | decimal | `56.3` | meters above WGS84 ellipsoid |
| AccuracyMeters | decimal | `4.0` | horizontal accuracy from GPS |
| RCOIs | string | `` | Roaming Consortium OIs; usually blank |
| MfgrId | int | `0` | BLE manufacturer ID; 0 or blank for WiFi |
| Type | string | `WIFI` | One of: `WIFI`, `CELL`, `BT`, `BLE` |

Line endings are `\n` (LF only, not CRLF).

---

## Required Server Response

The app parses the response body as JSON using the `UploadReseponse` model. A minimal success response is:

```json
{
  "success": true,
  "results": {
    "timeTaken": "0.123s",
    "filesize": 4096,
    "filename": "upload.csv",
    "transids": [
      {
        "file": "upload.csv",
        "size": 4096,
        "transId": "20240625-abc123"
      }
    ]
  }
}
```

**The app only checks `success` and `results.transids`:**

- If `success` is `false` or missing, the upload is marked failed and the DB marker is not advanced (the same records will be re-uploaded next time).
- If `success` is `true` and `transids` is non-empty, the transaction IDs are logged and broadcast via `UPLOAD_COMPLETE_INTENT`. The DB marker advances so those records are not re-uploaded.
- If `transids` is empty but `success` is `true`, the upload is still considered successful (DB marker advances) but a warning is logged.

A failure response:

```json
{
  "success": false,
  "message": "reason string shown in logs"
}
```

HTTP status codes: `200` is success. Any non-2xx triggers `onTaskFailed`; a `429` shows a "too many within timeframe" message to the user.

---

## Minimal Python Server (Raspberry Pi / FastAPI)

```python
# pip install fastapi uvicorn python-multipart
import time, uuid
from fastapi import FastAPI, File, Form, UploadFile, HTTPException, Depends
from fastapi.security import HTTPBearer, HTTPAuthorizationCredentials

SECRET_TOKEN = "your-secret-token-here"
UPLOAD_DIR = "/home/pi/wigle-captures"

app = FastAPI()
bearer = HTTPBearer()

def verify_token(creds: HTTPAuthorizationCredentials = Depends(bearer)):
    if creds.credentials != SECRET_TOKEN:
        raise HTTPException(status_code=401, detail="Invalid token")

@app.post("/upload")
async def upload(
    file: UploadFile = File(...),
    donate: str = Form(default=None),
    _=Depends(verify_token),
):
    import os
    os.makedirs(UPLOAD_DIR, exist_ok=True)
    trans_id = f"{time.strftime('%Y%m%d%H%M%S')}-{uuid.uuid4().hex[:8]}"
    dest = os.path.join(UPLOAD_DIR, f"{trans_id}.csv")
    content = await file.read()
    with open(dest, "wb") as f:
        f.write(content)
    return {
        "success": True,
        "results": {
            "timeTaken": "0s",
            "filesize": len(content),
            "filename": file.filename,
            "transids": [{"file": file.filename, "size": len(content), "transId": trans_id}],
        },
    }

# Run: uvicorn server:app --host 0.0.0.0 --port 8080
```

### Cloudflare Tunnel wiring

In your Cloudflare Tunnel config, forward `scans.home.example.com` → `localhost:8080`. Then in Cloudflare Access, create a service token and use its Client Secret as the Bearer token in the app. Alternatively, use a Cloudflare WAF rule to require a specific `Authorization` header value and skip Access entirely.

---

## Build Instructions

Requirements: Android Studio Ladybug or later, or the command line with a valid `local.properties` pointing to your Android SDK.

```bash
# debug APK
./gradlew assembleDebug

# release APK (requires signing config in build.gradle)
./gradlew assembleRelease
```

The app targets SDK 36, min SDK 24 (Android 7.0+). No new permissions are required by this change.
