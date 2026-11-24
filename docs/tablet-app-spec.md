# Offline-First Digital Signage Player: Tablet App Spec (Demo / MVP)

## 1. Goal

Build an **Android tablet app (Flutter)** that plays short videos (reels and ads) in a fullscreen loop inside public transport vehicles in Madagascar, where internet is unreliable.

The app must:
1. **Play with no internet.** All playback comes from local storage.
2. **Update itself** when a connection is available (new videos downloaded, old ones removed).
3. **Log every play** locally, to prove ad diffusion to advertisers.
4. **Show different content per cooperative.** Each tablet belongs to one cooperative (transport group) and plays only the playlist assigned to that cooperative, plus any global content.
5. **Accept only authorized content.** Nobody (passenger, driver, hacker, or a compromised server/CDN) can make the tablet play a video that was not published by the owner.

This is a **demo to show prospects**, so reliability and a clean look matter more than extra features.

## 2. Out of scope (do NOT build now)

- Admin dashboard (the manifest is edited by hand or by script)
- User accounts, login, payments
- Live streaming
- Targeting by GPS or route (targeting per cooperative IS in scope)
- iOS support

## 3. Tech stack

- Flutter (stable), Dart, **Android only**
- Packages (suggested):
  - `video_player`: playback
  - `dio`: downloads with progress, retry, and resume
  - `path_provider`: local storage paths
  - `connectivity_plus`: online/offline detection
  - `wakelock_plus`: keep the screen on
  - `sqflite` or `shared_preferences`: cache index and play logs
  - `crypto`: SHA-256 hash check of downloaded files
  - `cryptography` (or `ed25519_edwards`): Ed25519 signature verification of the manifest
  - `flutter_secure_storage`: device token stored in the Android Keystore
  - `workmanager` or a simple periodic `Timer` (foreground is fine for the demo)

## 4. Architecture

```
lib/
  main.dart
  core/
    config.dart              // MANIFEST_URL, sync interval, log endpoint
  data/
    manifest_service.dart    // fetch + parse manifest.json
    download_service.dart    // download, verify hash, resume
    cache_repository.dart    // local index of cached videos
    playlist_repository.dart // builds playlist from cache + manifest
    play_log_repository.dart // local play logs + batch upload
  sync/
    sync_engine.dart         // orchestrates a sync cycle
  security/
    manifest_verifier.dart   // Ed25519 signature + version/cooperative checks
    device_auth.dart         // device token in secure storage
    pin_guard.dart           // PIN protection for status screen and admin actions
  ui/
    player_screen.dart       // fullscreen loop player
    status_screen.dart       // hidden status/diagnostics screen
```

## 5. Manifest format (per cooperative, signed)

Each cooperative has **its own manifest**. The server builds it by combining:
- **global content** (shown in every cooperative), and
- **cooperative-specific content** (ads and reels assigned only to that cooperative).

The tablet is provisioned with a `cooperative_id` and only ever requests and accepts the manifest for that ID.

### 5.1 Where it is fetched
- Demo: a static file per cooperative, e.g. `https://YOUR_HOST/manifests/coop-tana-01.json`.
- Production: `GET /v1/cooperatives/{cooperative_id}/manifest` with the device token in the `Authorization` header.

### 5.2 Envelope (what the file contains)

The manifest is wrapped in a **signed envelope**. The signature covers the exact bytes of the payload, so there is no JSON canonicalization problem.

```json
{
  "payload_b64": "<base64 of the payload JSON bytes>",
  "signature_b64": "<base64 Ed25519 signature of those exact bytes>",
  "key_id": "k1"
}
```

### 5.3 Payload (after base64 decoding)

```json
{
  "manifest_version": 12,
  "cooperative_id": "coop-tana-01",
  "cooperative_name": "Cooperative Example",
  "generated_at": "2026-11-23T08:00:00Z",
  "videos": [
    {
      "id": "ad-001",
      "type": "ad",
      "scope": "cooperative",
      "title": "Brand X Promo",
      "url": "https://cdn.example.com/videos/ad-001.mp4",
      "sha256": "abc123...",
      "size_bytes": 4200000,
      "duration_sec": 15,
      "start_date": "2026-11-20",
      "end_date": "2026-12-20",
      "weight": 2
    },
    {
      "id": "reel-007",
      "type": "reel",
      "scope": "global",
      "title": "Funny clip",
      "url": "https://cdn.example.com/videos/reel-007.mp4",
      "sha256": "def456...",
      "size_bytes": 3100000,
      "duration_sec": 20,
      "start_date": "2026-11-01",
      "end_date": "2027-01-01",
      "weight": 1
    }
  ]
}
```

Field notes:
- `cooperative_id`: must equal the tablet's configured cooperative ID, otherwise the manifest is rejected.
- `manifest_version`: strictly increasing number per cooperative. The tablet rejects any version lower than the one it already accepted (prevents replaying an old manifest).
- `scope`: `global` (all cooperatives) or `cooperative` (this one only). Informational for the app; the server decides what goes into each manifest.
- `weight`: how many times the video appears per loop (weight 2 plays twice as often).
- `start_date` / `end_date`: the video only plays within this window (checked locally against the device date).
- `sha256`: the only way a file is trusted. It is inside the signed payload, so it cannot be changed without the private key.

## 6. Functional requirements

### 6.1 Player (`player_screen.dart`)
- Fullscreen, landscape-friendly, no system bars (immersive mode).
- Keep screen awake (`wakelock_plus`).
- Play videos from **local files only**, in a continuous loop based on the current playlist.
- Play **only files that are in the verified cache index** (see Security). Never scan folders for videos, never accept a file path or URL from outside the app.
- Seamless transition between videos (preload the next video if possible, no black flash).
- If the playlist is empty (first launch, nothing downloaded yet), show a simple branded placeholder screen.
- If a video file is missing or corrupted, **skip it** and continue; never crash or freeze.
- Respect `start_date` / `end_date` and `weight` when building the playlist.
- Mute option via config (default: muted with a visible subtitle area reserved for the future; sound can be enabled by a config flag).

### 6.2 Local cache (`cache_repository.dart`)
- Store videos in the app's **private internal storage** (not external storage, not the SD card, not a shared folder), under `/videos/`, so other apps and users cannot drop files into it.
- Keep an index (id, local path, sha256, size, downloaded_at).
- Downloads write to a temp file (`.part`) and are renamed only after the hash is verified, so a half-downloaded file is never played.
- Delete videos that are no longer in the manifest or have expired (cleanup after a successful sync).
- Enforce a max cache size (configurable, e.g. 2 GB); if exceeded, remove expired first, then oldest.

### 6.3 Sync engine (`sync_engine.dart`)
- Trigger a sync: on app start, every N minutes (config, default 5 min), and when connectivity changes from offline to online.
- Cycle:
  1. Fetch the manifest for **this tablet's cooperative** (timeout 10 s). If it fails, stop silently and keep playing the current playlist.
  1b. **Verify the manifest** (signature, cooperative ID, version; see Security). If any check fails, reject it, keep the current playlist, and record the event in a local security log.
  2. Compare with the cache index (by `id` + `sha256`).
  3. Download missing or changed videos sequentially, with retry and resume.
  4. Verify the hash; discard and retry on mismatch.
  5. **Only after all required downloads succeed**, swap in the new playlist. Never leave the device with a partial playlist.
  6. Delete removed or expired videos.
- Must never interrupt playback. Downloads run in the background.
- Handle connection drops mid-download gracefully (resume on next cycle).
- Use a lock so two sync cycles never run at the same time.

### 6.4 Play logging (`play_log_repository.dart`)
- Each time a video **finishes playing (or plays at least 80%)**, store a local log row: `video_id`, `started_at`, `duration_played`, `device_id`.
- Logs are saved locally and survive restarts.
- When online, upload logs in batches to `LOG_ENDPOINT` (POST JSON). Delete logs only after a successful upload.
- If `LOG_ENDPOINT` is not configured (demo mode), logs stay local and are shown on the status screen.

Log upload payload example:
```json
{
  "device_id": "tablet-demo-01",
  "logs": [
    {"video_id": "ad-001", "started_at": "2026-11-23T08:01:10Z", "played_sec": 15},
    {"video_id": "reel-007", "started_at": "2026-11-23T08:01:25Z", "played_sec": 20}
  ]
}
```

### 6.5 Status screen (`status_screen.dart`), the sales tool
- Opened by a **hidden gesture** (e.g. long-press on the top-right corner for 3 seconds) and then a **PIN** (configurable, stored hashed). Wrong PIN attempts are limited (e.g. 5 tries, then a 5-minute lockout). Return with a back gesture or button.
- Also shows the **cooperative name/ID** and the result of the last manifest verification.
- Shows:
  - Online / offline indicator
  - Last successful sync time and manifest version
  - Number of cached videos and total cache size
  - Current playlist (titles)
  - **Play count per video** (today and total)
  - Pending logs not yet uploaded
  - Device ID and app version
- Buttons: **"Sync now"** and **"Clear cache"** (with confirmation).

## 7. Non-functional requirements

- **Resilience**: survive app kill, tablet reboot, and sudden power loss without corrupting the cache or logs.
- **Auto-start**: the app should launch on boot (Android boot receiver) and open directly on the player.
- **Kiosk behavior**: use Android screen pinning / lock task mode so passengers cannot leave the app (for the demo, documenting the manual setup steps is enough).
- **Performance**: smooth playback on a mid-range Android tablet; videos are H.264 MP4, 480p to 720p.
- **Low data usage**: sync only downloads what changed; manifest requests are tiny.
- **Logging/debugging**: simple console logs for sync steps and errors.
- **No crashes on bad data**: invalid manifest or missing fields must be ignored safely.

## 8. Configuration (`config.dart`)

```dart
// Identity (set once per tablet during provisioning)
const cooperativeId = 'coop-tana-01';
const deviceId = 'tablet-demo-01';

// Endpoints
const manifestBaseUrl = 'https://YOUR_HOST/manifests'; // final: /{cooperativeId}.json
const logEndpoint = ''; // optional, empty = local logs only

// Security
const manifestPublicKeys = {
  'k1': 'BASE64_ED25519_PUBLIC_KEY', // public key only, NEVER the private key
};
const allowedVideoHosts = ['cdn.example.com']; // downloads from any other host are refused

// Behavior
const syncIntervalMinutes = 5;
const maxCacheBytes = 2 * 1024 * 1024 * 1024; // 2 GB
const muted = true;
```

The device token is **not** in this file. It is stored in the Android Keystore through `flutter_secure_storage` during provisioning.

## 9. Security: only authorized content can play

**Principle: the tablet trusts nothing except what is signed by the owner's private key.** Even if the network, the storage bucket, or the CDN is compromised, the tablet will not play an unauthorized video.

### 9.1 Signed manifest (the core protection)
- The owner generates an **Ed25519 key pair**. The **private key stays on the backend / the owner's machine only** and is never put in the app, in Git, or on a tablet. The app embeds only the **public key**.
- Every manifest is signed with the private key. The app verifies the signature with the public key **before** using the manifest. Invalid signature means reject.
- The app also checks that `cooperative_id` matches its own and that `manifest_version` is greater than or equal to the last accepted one (rollback protection).
- Support a `key_id` so keys can be rotated later (the app can hold several public keys).

### 9.2 Hash-verified files
- Every video's `sha256` is inside the signed payload. A downloaded file is accepted **only if its SHA-256 matches**. A swapped or modified file, even on a hacked CDN, is deleted.
- The **verified cache index** (id, path, sha256, size) is the only list the player can use. A file that is not in the index is never played, even if it sits in the folder.
- At app start and after each sync, re-check that indexed files still exist and match their size. Do a full hash re-check periodically (e.g. once a day, or at startup if CPU allows). If a file fails, delete it and re-download.

### 9.3 Network security
- **HTTPS only.** No cleartext HTTP in release builds.
- Downloads are only allowed from hosts in `allowedVideoHosts`.
- Optional: **certificate pinning** for the API/CDN host.

### 9.4 Device identity and revocation
- Each tablet has a unique `deviceId` and a **secret device token** (stored in the Android Keystore), provisioned once.
- The token is sent on manifest requests and log uploads. The server can **revoke** a token (stolen or lost tablet), after which the server refuses it.
- Provisioning also fixes the tablet's `cooperative_id`. Changing it requires re-provisioning with the PIN.

### 9.5 No way to inject content locally
- **No import feature** of any kind: no file picker, no "play from URL", no USB/SD scanning in the app.
- Videos live only in app-private internal storage.
- Set `android:exported="false"` on every component except the launcher activity, and register **no deep links or intents** that accept a video path or URL.
- Status screen and actions (Sync now, Clear cache, change config) are behind the **PIN**.

### 9.6 Physical lock-down of the tablet
- **Kiosk / lock task mode** (Android screen pinning, or device-owner mode for full lock-down), so passengers cannot leave the app.
- Disable: USB file transfer (MTP), developer options and USB debugging, the status bar and notifications, installing unknown apps, and SD card access if possible.
- Set a screen lock/PIN on the tablet. Consider an MDM tool later to manage the fleet.
- Mount the tablet in a **locked case or bracket** inside the vehicle.

### 9.7 Build hardening
- Release build with `flutter build apk --release --obfuscate --split-debug-info=./symbols`.
- No debug logging of tokens, keys or URLs in release.
- Optional: basic root/emulator detection that stops playback of unsynced content and logs the event.

### 9.8 Logs integrity (for advertisers)
- Play logs include `deviceId` and `cooperative_id`, and are uploaded with the device token. The server should reject logs from unknown or revoked devices.
- Optional: sign each log batch with an HMAC using a per-device secret, so logs can't be forged.

### 9.9 Security log
- Store local security events (rejected manifest, hash mismatch, wrong PIN attempts, unknown file found) and upload them with the play logs. Show the last few on the status screen.

### 9.10 Honest limits
- This setup makes it impossible to push unauthorized videos **through the network, the storage, or the app itself**.
- Someone with **full physical access and a rooted or re-flashed tablet** could still tamper with it. The kiosk lock, locked mounting, and token revocation reduce that risk and let you cut off a compromised device. The **private signing key** is the crown jewel: protect it and back it up securely.

## 10. Android setup notes

- `AndroidManifest.xml`: `INTERNET`, `ACCESS_NETWORK_STATE`, `WAKE_LOCK`, `RECEIVE_BOOT_COMPLETED`.
- Set the activity to landscape (or allow both orientations).
- Use HTTPS even for tests (cleartext HTTP is allowed in debug builds only, if really needed).
- Immersive fullscreen via `SystemChrome.setEnabledSystemUIMode(SystemUiMode.immersiveSticky)`.
- `android:allowBackup="false"` and `android:usesCleartextTraffic="false"` in release.
- `android:exported="false"` on every component except the launcher activity.

## 11. Demo scenario (acceptance test)

The app is "done" for the demo when this sequence works end to end:

1. Launch the app online. It downloads the playlist and starts looping.
2. Switch the tablet to **airplane mode**. Playback continues with no interruption.
3. Upload a new video and update `manifest.json`.
4. Turn the connection back on. Within about one sync interval, the new video downloads and appears in the loop, **without restarting the app**.
5. Remove a video from the manifest. After the next sync, it disappears from the loop and from storage.
6. Reboot the tablet. The app auto-starts and plays from the cache even with no connection.
7. Open the status screen (after the PIN). Play counts and the last sync time are correct.
8. **Per cooperative**: provision two tablets (or two configs) with different `cooperative_id` values. Each plays only its own cooperative content plus the global content.
9. **Tampered manifest**: edit the manifest JSON without re-signing. The tablet rejects it, keeps the old playlist, and logs a security event.
10. **Wrong cooperative / old version**: serve manifest B to cooperative A, then an older `manifest_version`. Both are rejected.
11. **Tampered video**: replace a video on the server (or in storage) with a different file without updating the signed hash. The download is rejected and deleted.
12. **Injected file**: copy a random `.mp4` into the app's video folder (on a debug device). It is never played because it is not in the verified index.
13. **Wrong PIN**: after 5 wrong attempts the status screen locks for a few minutes.

## 12. Helper script (optional, for the demo)

A small Python script, run only on the owner's machine or the server, that:
1. reads a simple config of cooperatives and which videos belong to each (plus global ones),
2. scans the video folder and computes the SHA-256 and size of each file,
3. generates one payload per cooperative with an incremented `manifest_version`,
4. **signs each payload with the Ed25519 private key** and writes the signed envelope `manifests/{cooperative_id}.json`.

Updating content for any cooperative is then one command. The private key never leaves the owner's machine.

## 13. Later (not now)

- Admin web dashboard (FastAPI): upload, campaign dates, assign to vehicles, reports
- Per-vehicle / per-route targeting (cooperative-level targeting is already in the MVP)
- Remote monitoring (heartbeat), remote config
- Fleet sync over SIM data, depot Wi-Fi, or USB/SD fallback
- Subtitles and optional sound
