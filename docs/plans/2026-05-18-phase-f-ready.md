# Phase F — ready to apply (don't apply until Gate 2 + Phase D pass)

Pre-drafted edits to execute the moment the SDK player is verified
working on Terry. Each edit is exact text replacement; ~5 min total
to apply + commit + push + kiosk rebuild.

## Order of operations when Gate 2 passes

1. Apply dashboard edits (5 min) — rename, push
2. Apply kiosk edits (5 min) — strip dead code, add volume control
3. User rebuilds + reinstalls APK from Mac Terminal (5 min)
4. Live verify on Terry (10 min)

## Edit 1 — Dashboard swap

`/Users/elul/Projects/family-dashboard/`:

```
git mv index.html index-legacy.html
git mv index-sdk.html index.html
git commit -m "swap: index-sdk → index; old index → index-legacy

After Gate 2 + Phase D verified the SDK player works on Terry:
swap the default dashboard to the SDK build. index-legacy.html
kept as a one-line rollback path.

Rollback: git mv index.html index-sdk.html && git mv index-legacy.html index.html"
git push
```

GitHub Pages auto-deploys in ~30-60s. Terry's kiosk URL is the repo
root — no kiosk-side change needed for the dashboard swap.

## Edit 2 — Kiosk MainActivity.kt strip

`/Users/elul/Projects/apps/kiosk-webview/app/src/main/java/xyz/erapps/kiosk/MainActivity.kt`:

### 2a. Remove unused imports

REMOVE these lines from the imports block:

```kotlin
import android.media.session.MediaController
import android.media.session.MediaSessionManager
import android.media.session.PlaybackState
import android.text.TextUtils
```

### 2b. Add volume-rocker → STREAM_MUSIC in onCreate

After `requestWindowFeature(Window.FEATURE_NO_TITLE)` (very early in
onCreate, before any other setup), ADD:

```kotlin
// Phase F (2026-05-18): route Portal's hardware volume rocker to
// WebView audio (STREAM_MUSIC). Without this, the rocker might
// default to ringer volume when the kiosk is in foreground, which
// would mean the family couldn't control SDK playback volume with
// the physical buttons.
volumeControlStream = AudioManager.STREAM_MUSIC
```

### 2c. Remove deferred notification-access prompt

REMOVE this block from onCreate (currently around line 108-116):

```kotlin
// Defer the Notification-access prompt so it doesn't collide with the
// overlay-permission prompt above. Fires once, ~3s after launch,
// ONLY if the user has overlay grant (i.e. core kiosk works) AND
// doesn't already have notification access AND we haven't asked
// recently. Non-blocking — the kiosk fully works without it; the
// Spotify control path just degrades to dispatchMediaKey-only.
handler.postDelayed({
    maybePromptNotificationAccess()
}, 3_000L)
```

### 2d. Simplify the onCreate log line

REPLACE:

```kotlin
Log.i(TAG, "onCreate done; overlay=${if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.M) Settings.canDrawOverlays(this) else true} notif=${hasNotificationListenerAccess()}")
```

WITH:

```kotlin
Log.i(TAG, "onCreate done; overlay=${if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.M) Settings.canDrawOverlays(this) else true}")
```

### 2e. Strip the media-command branches

REPLACE the entire `handleSpotifyCommand` function with:

```kotlin
private fun handleSpotifyCommand(cmd: String) {
    Log.i(TAG, "spotify cmd=$cmd")
    when (cmd) {
        "launch" -> launchSpotifyApk()
        // 2026-05-18 Phase F: play/pause/prev/next now go directly
        // through the Web Playback SDK in the dashboard. We keep this
        // method only for the launcher and any future kiosk-side
        // commands.
    }
}
```

### 2f. Delete now-unused helper methods

REMOVE these entire methods:

- `reliableMediaCmd()` (the verify-and-fallback orchestrator)
- `dispatchMediaKey()` (the AudioManager keypress dispatcher)
- `hasNotificationListenerAccess()` (the grant check)
- `spotifyPlaybackState()` (the MediaSession state reader)
- `spotifyMediaController()` (the MediaController fetcher)
- `maybePromptNotificationAccess()` (the deferred prompt)

### 2g. Remove the MEDIA_VERIFY_DELAY_MS constant

REMOVE from the companion object:

```kotlin
// Verify-and-fallback timing for reliableMediaCmd. Long enough that
// Spotify's MediaSession has reflected the new state, short enough
// that the user doesn't perceive a stuck button.
const val MEDIA_VERIFY_DELAY_MS = 600L
```

(SPOTIFY_PKG and TAG stay — still used by launchSpotifyApk + logging.)

## Edit 3 — Delete NotificationListener.kt

```
rm /Users/elul/Projects/apps/kiosk-webview/app/src/main/java/xyz/erapps/kiosk/NotificationListener.kt
```

## Edit 4 — AndroidManifest.xml: remove NotificationListener service

`/Users/elul/Projects/apps/kiosk-webview/app/src/main/AndroidManifest.xml`:

REMOVE this entire block:

```xml
<!-- NotificationListenerService stub (2026-05-17).
     We don't actually read notifications. The service exists so the
     user can grant "Notification access" once, which unlocks
     MediaSessionManager.getActiveSessions() — required for the
     verify-and-fallback Spotify control path in MainActivity
     (reliableMediaCmd). If the grant is not given, MainActivity
     falls back to the original dispatchMediaKey-only behavior. -->
<service
    android:name=".NotificationListener"
    android:exported="false"
    android:permission="android.permission.BIND_NOTIFICATION_LISTENER_SERVICE">
    <intent-filter>
        <action android:name="android.service.notification.NotificationListenerService" />
    </intent-filter>
</service>
```

## Edit 5 — User rebuilds + reinstalls

From Mac Terminal (sandbox blocks Gradle):

```bash
cd /Users/elul/Projects/apps/kiosk-webview
./build.sh && \
  adb -s 192.168.1.114:5555 install -r app/build/outputs/apk/debug/app-debug.apk && \
  adb -s 192.168.1.114:5555 shell am force-stop xyz.erapps.kiosk && \
  adb -s 192.168.1.114:5555 shell am start -n xyz.erapps.kiosk/.MainActivity
```

After install:
- Kiosk relaunches → loads `https://er-builder.github.io/family-dashboard/`
  (now serving the SDK build at index.html)
- The notification-access prompt no longer fires (code removed)
- Volume rocker now routes through STREAM_MUSIC for SDK audio
- All `kiosk://spotify/{play,pause,prev,next}` URIs from the dashboard
  are no-ops (handleSpotifyCommand now ignores them). The new dashboard
  doesn't fire those URIs at all (spCommand goes direct to SDK), so this
  is harmless.

## Edit 6 — Update AGENTS.md

After the swap is verified, update AGENTS.md to reflect:
- New architecture: dashboard = Spotify Connect device
- Spotify 8.6.x APK pin no longer load-bearing (still installed as
  music browser, but audio comes from SDK)
- Removed: reliableMediaCmd patterns, NotificationListener requirement,
  resurrection logic, soft-reset
- New: token refresh loop, sdkStateToProxyShape adapter, maybeTransferToTerry

## Rollback plan

If Phase F breaks something:

1. Dashboard rollback (instant, no APK rebuild needed):
   ```
   cd /Users/elul/Projects/family-dashboard
   git mv index.html index-sdk.html
   git mv index-legacy.html index.html
   git commit -m "rollback: revert to 9.5 dashboard"
   git push
   ```

2. Kiosk rollback (only if media-key fallback for the legacy dashboard
   matters): revert MainActivity.kt + restore NotificationListener.kt
   + restore manifest block, then rebuild APK. The Phase F changes are
   tracked in this doc.
