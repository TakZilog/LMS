# AMAES Moodle Solver (Android Release Distribution)

- **Download Latest Android APK:** [AMAES-Moodle-Solver.apk](https://github.com/TakZilog/LMS/raw/main/AMAES-Moodle-Solver.apk)
- **Live OTA Version Manifest:** [`version.json`](https://raw.githubusercontent.com/TakZilog/LMS/main/version.json)

## SECURITY: rotate the old Gemini key

APK builds up to and including v1.3.47 (build 147) shipped a Gemini API key inside the app,
XOR-masked in `main.dart`. Anyone with the APK can recover it, so **treat that key as compromised**:

1. Open Google AI Studio → API keys and **delete / revoke the old key**.
2. Create a new key and put it **only** on the backend (see below). Never put it in the Flutter app.

The app no longer calls Gemini directly and has no fallback to direct Gemini. All solving goes
through the backend.

## Backend setup

```
cd backend
python -m venv .venv
.venv\Scripts\pip install -r requirements.txt
```

`GEMINI_API_KEY` must be set in the environment, or in `backend\.env` (`GEMINI_API_KEY=...`), which is
loaded into the environment at startup. The server **refuses to start** if it is missing. Optional:
`GEMINI_MODEL` (default `gemini-3.5-flash-lite`).

### Run with exactly one worker

```
start_backend.bat
# equivalent to:
uvicorn main:app --host 127.0.0.1 --port 8000 --workers 1
```

The 4-slot concurrency cap and the per-device rate limit are stored in process memory. With more
than one worker, each process would get its own 4 slots and its own counters, so the limits stop
working. Always use `--workers 1`. `start_backend.bat` and `deploy_backend.bat` both enforce this.

### Limits

- 4 global solve slots. Each request leases one slot for up to 90 s. The Gemini call times out at 60 s.
  Slots are released on success, error, timeout and cancel. Expired leases are reclaimed.
  If no slot is free the server returns `429 {"error":"BUSY"}`.
- 60 solves per device (SSAID) per 60 s → `429 {"error":"RATE_LIMITED"}`.

### API

| Method | Path | Responses |
|---|---|---|
| POST | `/solve` `{"ssaid","question","subject"}` | 200 `{"answer","slot"}`, 400 `INVALID_REQUEST`, 429 `BUSY` / `RATE_LIMITED`, 502 `UPSTREAM_ERROR` |
| POST | `/solve-batch` `{"ssaid","subject","notes","questions":[{"id","question"}]}` (≤10) | 200 `{"answers":[{"id","answer"}],"slot","cached"}`, same errors as `/solve`. One Gemini call and one slot per page. Answers are cached in memory and shared across devices, keyed on the exact subject + notes + question, so cache hits use no slot and don't count toward the rate limit. |
| GET | `/status` | `{"in_use","capacity":4,"slots":[{"index","in_use","expires_in_ms"}]}` |
| GET | `/health` | `{"status":"ok"}` |

### Tests

```
cd backend
.venv\Scripts\python -m pytest
```

## Public URL (Cloudflare Quick Tunnel)

`deploy_backend.bat` starts the backend (one worker) and opens a Quick Tunnel. Each time the tunnel
restarts it prints a **new** `https://<random>.trycloudflare.com` URL. Then do this:

1. Copy the new URL from the tunnel window.
2. In the app, open **Options → Backend Server → Backend URL**, paste it and tap **Save**.

**Automatic URL delivery (secret gist):** if `backend\.env` has `GIST_TOKEN=` (a GitHub token with
only the `gist` scope), `deploy_backend.bat` writes each new tunnel URL to a secret gist through
`tools\publish_tunnel_url.ps1`. The app reads that gist when it starts, and again (at most once a
minute) whenever the backend is unreachable. On the first run the script creates the gist, saves
`GIST_ID` to `.env` and writes it into `_backendUrlGistId` in `main.dart`, so you then rebuild the APK
once. Anyone who extracts the gist ID from the APK can read the URL.

The URL is saved on the device. `/health`, `/status` and `/solve` all use this one base URL.
No tunnel URL is hardcoded in the app. The default is `http://10.0.2.2:8000`, which is the emulator's
address for the host PC.

## App behaviour

- While the app is in the foreground, it polls `GET /status` every 2.5 s and shows `X/4 slots in use`.
  When all 4 slots are busy, it shows "All slots busy — try again shortly" and disables Solve.
  When the backend is unreachable, it shows "Service unavailable" and disables Solve.
- The device ID sent as `ssaid` is Android `Settings.Secure.ANDROID_ID`.
- Release builds check the APK signing certificate. If its SHA-256 does not match
  `_expectedSigningSha256` in `flutter_app/lib/main.dart`, Solve is disabled.
  Release builds are signed with the keystore named in `flutter_app/android/key.properties`. If that
  file is missing, the build falls back to the debug key, and the check then disables Solve. The constant
  still holds the debug-key hash until you generate your release keystore (see the next section).
- Release builds use R8 (`minifyEnabled`, `shrinkResources`, `debuggable false`).

## How to generate a release keystore and update the signature hash

Run these in PowerShell from `C:\xampp\htdocs\LMS\flutter_app\android`.

1. Generate the keystore. You will be asked for a password and your name or organisation:
   ```powershell
   & "C:\Program Files\Android\Android Studio\jbr\bin\keytool.exe" -genkeypair -v -keystore upload-keystore.jks -storetype JKS -keyalg RSA -keysize 2048 -validity 10000 -alias upload
   ```
   Back up `upload-keystore.jks` and its password somewhere safe. If you lose them, you cannot ship
   updates that install over the existing app.
2. Copy `key.properties.example` to `key.properties` and fill in the passwords. Keep `keyAlias=upload`
   and `storeFile=../upload-keystore.jks`. Both files are gitignored.
3. Build a release APK:
   ```powershell
   cd ..; flutter build apk --release --target-platform=android-arm64 --obfuscate --split-debug-info=build/debug-info
   ```
4. Extract the signing certificate SHA-256 with `apksigner`:
   ```powershell
   $env:JAVA_HOME = "C:\Program Files\Android\Android Studio\jbr"
   & "$env:LOCALAPPDATA\Android\Sdk\build-tools\36.0.0\apksigner.bat" verify --print-certs build\app\outputs\flutter-apk\app-release.apk
   ```
   Copy the hex from the `Signer #1 certificate SHA-256 digest:` line.
5. Paste that value into `_expectedSigningSha256` in `flutter_app/lib/main.dart` (lowercase, no colons),
   then rebuild. Use `build_apk.bat` for a published release.

Phones that have the old debug-signed APK must uninstall it once. Android refuses to install an update
signed with a different key.

## Known limitations

- **Quick Tunnel URL changes** on every restart and depends on the host PC staying on. Use a named
  tunnel or a hosted server for a stable URL.
- **No Play Integrity / Firebase App Check.** The backend cannot prove that a request comes from the
  genuine app. The signature check runs on the device and a modified client can bypass it.
- **SSAID is advisory.** The client supplies it, so a modified client can rotate it to avoid the
  per-device rate limit. The global 4-slot cap still protects the Gemini quota.
- If `key.properties` is missing, releases fall back to the **debug keystore**, which is not a secret.
- The backend has no auth. Anyone who has the tunnel URL can use the 4 slots.
- State is in memory, so a backend restart clears the slots and rate counters.
