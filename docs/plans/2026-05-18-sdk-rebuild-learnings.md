# SDK rebuild — what we learned (2026-05-18)

**Status:** Attempted, rolled back same day. The Web Playback SDK is viable on
Portal hardware in principle but blocked by the system WebView our kiosk uses.
9.5 architecture (proxy-poll + kiosk media-key dispatch) is the right shape
for this device until we either replace the WebView or replace the hardware.

## TL;DR — when to revisit

Trigger conditions for trying again:
- Hardware change (Portal replaced with a less degoogled / less locked device)
- We're willing to spend a day replacing `WebView` with a `startActivity` into
  Portal's standalone `org.chromium.chrome` (which DOES work with the SDK,
  proven today)
- We're willing to sideload a newer Android System WebView APK on Portal
  (risky on a degoogled device — could brick other apps)

If none of those, the polling + media-key control architecture is fine. Today's
9.5 work (CPU pass, button-handler fix, always-on launcher, polish) already
moved the device from a self-reported 5.5 to 9.5. The remaining gap to 10 is
latency (polling lag) and voice — neither solvable without one of the above.

## What we proved (positive findings)

1. **Widevine L1 + HW_SECURE_ALL works on Portal hardware.** The Axinom EME
   capability tool reported it in Portal's standalone Chromium browser
   (`org.chromium.chrome`). Manus's NO-GO assumption that degoogled Portal
   would only have L3 was wrong.

2. **The Web Playback SDK initializes and plays audio in Portal's standalone
   Chromium browser.** sdk-test.html → green "READY" banner, audio of
   `Never Gonna Give You Up` played through Portal speakers on tap, pause/
   resume worked.

3. **The dashboard's index-sdk.html architecture is sound.** Bootstrap,
   token refresh, sdkStateToProxyShape adapter, maybeTransferToTerry, all
   work as designed (in Chromium browser).

## What we hit (the real blocker)

**The kiosk uses `android.webkit.WebView` (system WebView), NOT Portal's
standalone Chromium app.** They share the same Chromium codebase but have
DIFFERENT EME/Widevine capabilities depending on the device manufacturer's
WebView build.

On Terry:
- Standalone Chromium browser (`org.chromium.chrome`): EME init works, SDK plays
- System WebView (used by our `android.webkit.WebView` in `MainActivity`):
  EME init **fails silently** → SDK throws `INIT_ERR: Failed to initialize player`

The SDK's `navigator.requestMediaKeySystemAccess` *presence check* passes in
both contexts (UA reports it's available), but actually *calling* it
successfully only works in the standalone Chromium. The system WebView is too
locked-down or too old.

UA from the kiosk: `Mozilla/5.0 (Linux; Android 10; PortalGo Build/QKQ1.210213.001; wv)`
The `wv` token indicates WebView mode. No way to bypass without replacing the
component.

## Other findings worth remembering

### Spotify Web Playback SDK scope requirements (often-overlooked)
The SDK throws `AUTH_ERR: Invalid token scopes` if any of these are missing:
- `streaming`
- `user-read-email`
- **`user-read-private`** — easy to miss; the docs bury it
- Plus the usual `user-read-playback-state`, `user-modify-playback-state`,
  `user-read-currently-playing`, `user-read-recently-played` if you want
  to drive Spotify Connect transfers + read state

### Browser autoplay policy
Even when the SDK initializes successfully, Chromium blocks audio that starts
from a network event (Spotify Connect transfer) without a user gesture. Fix:
`player.activateElement()` from inside a click/touch handler. Bind a one-time
document-level listener so any tap unlocks audio.

Doesn't apply in the kiosk WebView because MainActivity sets
`mediaPlaybackRequiresUserGesture = false`.

### System WebView ≠ standalone Chromium for DRM
This was the deepest learning. On the same device:
- Same Chromium version reported
- Same Widevine binaries on disk
- Same OS, same hardware
- **But different DRM capabilities** because the system WebView is provided
  by a separate Android component that the OEM may have stripped down.

Trust standalone Chromium results only for standalone Chromium deployments.
Always test in the actual production WebView context before committing to an
architecture that depends on advanced browser features.

### Manus.ai feasibility report assessment
Manus said NO-GO based on degoogled Widevine assumptions. The verdict was
ultimately correct for our deployment (kiosk WebView), but the reasoning was
wrong. Widevine L1 IS present on Portal. The real blocker was the system
WebView vs standalone Chromium distinction Manus didn't make. Lesson: for
SDK feasibility questions, the cheap on-device test (load it and watch) is
worth more than research, because forums collapse "Android WebView" into one
thing when in practice there are at least two materially different runtimes.

## What we kept from today

The non-rolled-back wins:
- **−80% Vercel CPU** (proxy migrated to Edge runtime + cache TTL bump,
  dashboard poll cadence dropped). Holds.
- **Phase 0 + 1 + 2 reliability ship** (cache-bust on tap, skip-grace on
  explicit pause, single-click handlers, always-on launcher, in-card "Open
  Spotify" pill, long-press art, marquee, stale badge, type scale-up).
  Holds.
- **Phase 1a kiosk media-command reliability** (verify-and-fallback via
  MediaSessionManager). Holds — still the load-bearing path for play/pause
  reliability since we're back on media-key dispatch.

## What we rolled back

- `index-sdk.html` — **kept in repo** as reference for future revisit.
  Not the live `index.html` anymore.
- `sdk-test-v2.html` — kept in repo. Useful as a one-off harness if we
  want to retest in a different WebView later.
- `/api/spotify-token` proxy endpoint — **deleted** (security exposure).
  Recreate from git history if revisiting.
- `MainActivity.URL` — reverted from `/index-sdk.html` to `/`. APK rebuild
  required.
- Spotify refresh token in Vercel still has the expanded scopes — harmless,
  the polling proxy uses a subset. No reason to revert.

## If we revisit, in order of decreasing risk

1. **Replace WebView with a startActivity launch of Portal's standalone
   Chromium** (`org.chromium.chrome`) at the dashboard URL, fullscreen.
   The kiosk Activity becomes a thin launcher + auto-return scheduler.
   - Cost: lose `kiosk://exit`, `kiosk://spotify/launch`, overlay pill —
     would need replacements via Chrome flags or window manager tricks
   - Cost: lose the long-press exit pill UX
   - Gain: SDK works, native Chromium audio
   - Estimated: 4-8h exploratory

2. **Sideload Android System WebView APK from a non-degoogled source**
   (XDA, APKMirror). Replace `com.google.android.webview` with a newer
   build that has full EME.
   - Risk: bricks the kiosk if signatures don't match Portal's expectations
   - Risk: breaks other Portal apps that use WebView
   - Gain: drop-in fix if successful
   - Estimated: 2-3h research + 1h install + recovery if failed

3. **Replace Portal hardware.** A non-degoogled tablet with stock Android
   would have a normal WebView and the SDK would just work.
   - Cost: new device + reconfiguration
   - Cost: lose Portal's audio quality and form factor
   - Estimated: 1-2 day project including hardware purchase, kiosk setup,
     mounting, family migration

## Total time spent today

~5h end-to-end. ~3h would have been saved if we'd loaded sdk-test-v2.html in
the actual kiosk WebView first (instead of in standalone Chromium). Lesson:
**always test the SDK in the exact runtime that production will use.**
