# Agnes AI Backend Migration — Implementation Plan

## Goal
Rebuild `Agnes 3.0.47.apk` so all AI calls use the Agnes AI gateway (`https://apihub.agnes-ai.com/v1`) and `agnes-*` models, while preserving every other aspect of the app.

## Deliverable
`/workspaces/OpenAgnesAIAndroidApp/modified/Agnes-3.0.47-AgnesAI.apk`

## Target Endpoints & Models

| Capability | Model | Endpoint |
|---|---|---|
| Chat / text / agent | `agnes-2.5-flash` | `POST /v1/chat/completions` |
| Image generation | `agnes-image-2.1-flash` | `POST /v1/images/generations` |
| Video creation | `agnes-video-v2.0` | `POST /v1/videos` |
| Video polling | — | `GET https://apihub.agnes-ai.com/agnesapi?video_id=<ID>` |

Auth: `Authorization: Bearer <API_KEY>` on every request.

## Test Key (verification only)
`sk-I4LDRg6Juj4x0WhAU7l0FM7lBgDXgUqhtg5hebP3Nf6ys5IM`
Use during Phase 5 to confirm network calls work. **Do not embed in the final APK.**

---

## Phase 1 — Decompile & Inventory

### 1.1 Pre-flight
```bash
APK_SRC="/workspaces/OpenAgnesAIAndroidApp/Original-App/Agnes 3.0.47.apk+"
APK_WORK="/tmp/agnes-original.apk"
cp "$APK_SRC" "$APK_WORK"
python3 -c "import zipfile; print(zipfile.is_zipfile('$APK_WORK'))"
unzip -l "$APK_WORK" | grep -E 'config\.|split_' && echo "SPLIT_APK_DETECTED" || echo "MONOLITHIC"
```

**Gate G1**: If `SPLIT_APK_DETECTED` or zip invalid → abort; this plan covers monolithic APKs only.

### 1.2 Decode
```bash
mkdir -p /tmp/agnes-apktool /tmp/agnes-jadx /tmp/agnes-unzipped
unzip "$APK_WORK" -d /tmp/agnes-unzipped
apktool d "$APK_WORK" -o /tmp/agnes-apktool
jadx "$APK_WORK" -d /tmp/agnes-jadx
```

Record:
- Package name from `AndroidManifest.xml`
- Whether `classes2.dex` exists
- Any `.so` files under `lib/`
- Any `assets/` config files

### 1.3 Locate AI touch points
Run these searches and save results to `/tmp/agnes-audit.txt`:

```bash
# URLs / hosts
grep -ri "apihub\|agnes-ai\|openai\|api\.openai\|gpt-\|claude-\|/v1/chat\|/v1/images\|/v1/videos" \
  /tmp/agnes-jadx --include="*.java" -l 2>/dev/null >> /tmp/agnes-audit.txt

# HTTP clients
grep -ri "okhttp3\|retrofit2\|com.squareup\|com.android.volley\|ktor\|HttpURLConnection" \
  /tmp/agnes-jadx --include="*.java" -l 2>/dev/null >> /tmp/agnes-audit.txt

# Auth / keys
grep -ri "api_key\|apiKey\|API_KEY\|Authorization\|Bearer\|SharedPreferences" \
  /tmp/agnes-jadx --include="*.java" -l 2>/dev/null >> /tmp/agnes-audit.txt

# Native scan
for so in $(find /tmp/agnes-unzipped/lib -name '*.so' 2>/dev/null); do
  strings "$so" | grep -i "apihub\|agnes-ai\|openai\|gpt-\|/v1/" && echo "FOUND_IN:$so"
done >> /tmp/agnes-audit.txt
```

**Gate G2**: If `/tmp/agnes-audit.txt` is empty or contains only `.so` hits → escalate; no patchable Java/smali entry points found.

---

## Phase 2 — Analysis & Mapping

Produce a short mapping table. For each hit file, record:

| File | Class | Method | Current URL | Endpoint | Auth Source | Capability |
|---|---|---|---|---|---|---|

Also note:
- Which HTTP client is used (Retrofit, OkHttp, Volley, HttpURLConnection)
- Whether strings are obfuscated (single-letter class names = high obfuscation)
- Whether a `mapping.txt` exists in the APK

**Gate G3**: If >70% of AI classes are single-letter obfuscated → escalate.

---

## Phase 3 — Modification

### 3.1 URL & endpoint replacement
Replace all AI API hostnames with `https://apihub.agnes-ai.com/v1`.
Replace all AI paths with the exact Agnes paths from the table above.
Do not touch non-AI URLs.

### 3.2 Model name substitution
Replace every model identifier with the Agnes equivalent:
- General chat → `agnes-2.5-flash`
- Fast chat (if explicitly distinguished) → `agnes-1.5-flash`
- Image → `agnes-image-2.1-flash`
- Video → `agnes-video-v2.0`

### 3.3 Auth handling
- If a hardcoded key exists: remove it.
- If no key input exists: add a minimal first-launch `EditText` dialog; store in `SharedPreferences`.
- Every AI request must include `Authorization: Bearer <stored_key>`.
- If the app currently sends the key as a query param or body field, patch it to use the header instead.

### 3.4 Response parser patches (only if needed)
| Issue | Patch |
|---|---|
| Video response parsed `id` or `task_id` | Patch to read `video_id` |
| Video status checked exact string | Accept `succeeded`, `success`, `completed`, `done` |
| Chat request body missing `model` field | Add it |
| Image response parsed differently | Map to `url` or `b64_json` |

### 3.5 Do not touch
UI layouts, non-AI business logic, permissions, non-AI assets, native `.so` files, ProGuard config.

### 3.6 Execution order
1. URL strings
2. Model constants
3. Auth header
4. Response parsers

---

## Phase 4 — Rebuild

```bash
apktool b /tmp/agnes-apktool -o /tmp/agnes-modified.apk

keytool -genkeypair -v -keystore /tmp/debug.keystore -keyalg RSA -keysize 2048 -validity 10000 \
  -alias debug -storepass android -keypass android -dname "CN=Debug, OU=Dev, O=OpenAgnes, L=, S=, C=US"

apksigner sign --ks /tmp/debug.keystore --ks-pass pass:android --key-pass pass:android \
  --out /tmp/agnes-signed.apk /tmp/agnes-modified.apk

apksigner verify -v /tmp/agnes-signed.apk
mkdir -p /workspaces/OpenAgnesAIAndroidApp/modified
zipalign -v 4 /tmp/agnes-signed.apk /workspaces/OpenAgnesAIAndroidApp/modified/Agnes-3.0.47-AgnesAI.apk
zipalign -c -v 4 /workspaces/OpenAgnesAIAndroidApp/modified/Agnes-3.0.47-AgnesAI.apk
```

Validate:
- `apksigner verify` passes
- `zipalign -c` passes
- Output size is within 10% of original

---

## Phase 5 — Verification

### 5.1 Install on device / emulator
```bash
adb uninstall <PACKAGE_NAME> 2>/dev/null || true
adb install /workspaces/OpenAgnesAIAndroidApp/modified/Agnes-3.0.47-AgnesAI.apk
adb shell monkey -p <PACKAGE_NAME> -c android.intent.category.LAUNCHER 1
```

**Gate G4**: Install must succeed; app must launch without `FATAL EXCEPTION`.

### 5.2 Network verification
Use a proxy or logcat to confirm:
- Chat → `https://apihub.agnes-ai.com/v1/chat/completions` with model `agnes-2.5-flash`
- Image → `https://apihub.agnes-ai.com/v1/images/generations` with model `agnes-image-2.1-flash`
- Video create → `https://apihub.agnes-ai.com/v1/videos` with model `agnes-video-v2.0`
- Video poll → `https://apihub.agnes-ai.com/agnesapi?video_id=...`
- All requests include `Authorization: Bearer` header

Inject the test key via the app's settings UI for this check. Do not leave it in the APK.

### 5.3 Functional smoke tests
| Feature | Pass Criteria |
|---|---|
| Text chat | Non-empty response, no 401 |
| Image generation | Returns image URL or base64 |
| Video generation | Task created and poll succeeds |
| Error handling | Invalid key shows error UI, no crash |
| Non-AI features | All non-AI screens present and functional |

### 5.4 Logcat check
```bash
adb logcat -d | grep -E "FATAL|AndroidRuntime" | head -20
```
Must be empty.

---

## Rollback Criteria
Stop and revise if:
- UI elements are missing or broken
- App crashes on startup
- AI requests hit the old host
- Video polling uses `task_id` instead of `video_id`

---

## Out of Scope
- Rebuilding with original developer signature (impossible without original keystore)
- Supporting split APKs or app bundles
- Patching native `.so` AI logic via disassembly
