# Family Dashboard — Web Playback SDK rebuild (9.5 → 10)

**Date:** 2026-05-18
**Trigger:** SDK feasibility test today confirmed `READY` on Terry. Widevine L1 + HW_SECURE_ALL works inside Portal's WebView. Manus's NO-GO was wrong on both blockers. Path to 10/10 is open.
**Goal:** Replace the polling + media-key control architecture with the dashboard *as the player*. Eliminates 3 of 4 reliability bugs at the architectural level, removes the Spotify 8.6.x APK pin, makes Terry a real Spotify Connect target anyone on the account can stream to from any phone.
**Time:** ~10–12h active + 1 day soak. Split across 2–3 sessions.

---

## What this kills vs. what it preserves

| Today (post-9.5 ship) | After SDK rebuild |
|---|---|
| Proxy polls Spotify every 60s + edge cache 120s/600s | Direct SDK events (`player_state_changed`) — zero polling, zero proxy lag |
| Dashboard fires `kiosk://spotify/*` → kiosk Kotlin → media keys → Spotify APK | Dashboard calls `player.pause()` / `player.nextTrack()` directly — zero hops |
| Spotify-on-Terry APK is the audio source (pinned to 8.6.x because 9.x breaks login on degoogled Portal) | Dashboard WebView is the audio source. APK becomes optional / can be uninstalled |
| Terry not visible as Connect target from family phones (account-scoped) | Terry is a real Connect device named "Terry" — anyone signed into the household account targets it from any phone |
| Bug 1a (cache lag), 1b (key dispatch fails), 1c (grace race), 4 (random stops via LMK) | All gone at the architectural level |
| Bug 2 (button click handlers) | Unchanged — DOM event problem, already fixed in Phase 1b |
| Bug 3 (launcher reachable while playing) | Mostly gone — for daily playback, family uses phone Spotify → targets Terry. The "open APK to browse on the Portal screen" use case becomes optional |

What we keep from the 9.5 ship:
- Single-click handlers (Phase 1b)
- Always-on launcher (Phase 0 + 1c) — still useful for the rare "browse on Terry's screen" path
- All Phase 2 polish (typography, marquee, tap feedback, stale badge)
- The Phase 0–2 CPU savings on Vercel (Spotify proxy still exists for `lastPlayed` fallback when SDK isn't initialized)

What we delete:
- Kiosk's `reliableMediaCmd()` + MediaSessionManager fallback (no longer load-bearing — we don't need to control the APK)
- `NotificationListener.kt` stub (only existed for the MediaSessionManager grant)
- `kiosk://spotify/{play,pause,prev,next}` URI handlers in `MainActivity.kt` (no callers)
- The Phase 1d resurrection loop (no longer needed — SDK fires its own state events)
- The `<setInterval>` polling of `/api/spotify-now` (replaced by SDK events)

---

## Architecture

```
┌──────────────────────────────────────────────────────────────┐
│  Terry kiosk WebView                                         │
│  ┌────────────────────────────────────────────────────────┐ │
│  │  dashboard index.html                                  │ │
│  │  • Spotify.Player SDK instance ("Terry")               │ │
│  │  • player.addListener('player_state_changed', render)  │ │
│  │  • play/pause/prev/next call player.* directly         │ │
│  │  • Token refresh every 50min                           │ │
│  │  • Audio plays via WebView's media element             │ │
│  └────────────────────────────────────────────────────────┘ │
│                          │                                   │
│                          │ STREAM_MUSIC                      │
│                          ▼                                   │
│              Portal speakers + volume rocker                 │
└──────────────────────────────────────────────────────────────┘
                           ▲
                           │ Spotify Connect (push)
                           │
┌──────────────────────────────────────────────────────────────┐
│  Phone Spotify (Elul / household)                            │
│  Pick playlist → tap "Devices" → tap "Terry" → music plays   │
└──────────────────────────────────────────────────────────────┘

           Proxy: minimal, almost dead
           ┌─────────────────────────────────────┐
           │ /api/spotify-token (Edge)           │
           │   returns short-lived access_token  │
           │   guarded by per-user shared secret │
           │ /api/cal (unchanged)                │
           └─────────────────────────────────────┘
```

The dashboard becomes a real Spotify Connect device. Wife's question of "can I play to Terry from my phone" becomes a yes — *if* she logs into the household account. Cross-account still doesn't work, same as today.

---

## Phases (in order)

### Phase A — OAuth scope re-issue (~30 min)

The feasibility test showed `authentication_error: Invalid token scopes`. Current refresh token has `user-read-currently-playing user-read-recently-played`. We need to add `streaming user-modify-playback-state user-read-playback-state user-read-email`.

Re-run the OAuth dance (steps already documented in AGENTS.md under "Refresh-token recovery"):

1. Open `https://accounts.spotify.com/authorize?client_id=<CLIENT_ID>&response_type=code&redirect_uri=https%3A%2F%2Ffamily-dashboard-proxy.vercel.app%2Fapi%2Fspotify-callback&scope=streaming+user-read-email+user-read-playback-state+user-modify-playback-state+user-read-currently-playing+user-read-recently-played`
2. Approve, copy the `code=` value from the resulting 404 URL
3. `curl -X POST ...` → grab the new `refresh_token`
4. Update `SPOTIFY_REFRESH_TOKEN` in Vercel proxy env
5. Redeploy proxy (one push or trigger redeploy)

**Exit criteria:** Token endpoint returns a token with the new scopes (verify with `https://api.spotify.com/v1/me` → should return without 403). SDK test page (we'll redeploy briefly) goes green with no auth error.

### Phase B — Audio playback spike (~1.5h)

Build a one-off test page that:
1. Inits the SDK (now with correct scopes)
2. Transfers playback to "Terry" via `PUT /me/player`
3. Plays a hardcoded track for 30s
4. Verifies: audio comes out of Portal speakers, volume rocker controls it, pausing via `player.pause()` is instant

If audio plays cleanly through Portal speakers → green-light the rebuild.

**Specific things to check:**
- Audio latency on play tap (target < 200ms)
- Pause via SDK → silence is instant (target < 100ms)
- Volume rocker behavior — does it control the WebView audio without changes? If not, we need `setVolumeControlStream(AudioManager.STREAM_MUSIC)` in `MainActivity.onCreate()` (5 lines of Kotlin)
- Gapless playback (no clicks/pops between tracks)
- Behavior on incoming Portal video call — does audio duck or stop?

### Phase C — Build the SDK player layer in the dashboard (~3-4h)

Add a `spotify-player.js` script block to `index.html`. New responsibilities:

1. **Init:** load SDK, instantiate `Spotify.Player({ name: "Terry", getOAuthToken: ... })`, connect.
2. **Token refresh:** every 50 min (tokens are 60min, refresh with 10min headroom), fetch a fresh access_token from the proxy and pass to the SDK's `getOAuthToken` callback.
3. **State render:** listener for `player_state_changed` → render the same card the polling code renders today (track, artist, album art, progress, isPlaying). No polling. No cache-bust. No `?t=<ms>`. All synchronous, event-driven.
4. **Controls:** replace every `spCommand(cmd)` call with the direct SDK equivalent:
   - `spCommand("pause")` → `player.pause()`
   - `spCommand("play")` → `player.resume()`
   - `spCommand("next")` → `player.nextTrack()`
   - `spCommand("prev")` → `player.previousTrack()`
5. **No `lastPlayed`:** the SDK has a current state or it doesn't. When idle, the proxy's `lastPlayed` from `/api/spotify-now` still serves the "idle card" — but optionally, we can fetch it once per session via `/me/player/recently-played` directly and cache in `localStorage`.
6. **Connect device transfer:** on dashboard load, optionally call `PUT /me/player` with `device_id=Terry's SDK device_id` to make Terry the active playback target. Skip if there's already an active device — that means someone else's phone is playing, don't yank it.
7. **Music-override behavior:** keep current logic (auto-show during routine windows is suppressed). Routines win.

### Phase D — Connect device flow (~1h)

Verify on the actual Portal:
1. Open Spotify on phone (your account) → Devices → "Terry" should be listed
2. Tap "Terry" → music transfers, audio comes out of Portal
3. From phone: skip, pause, change playlist — verify dashboard reflects state instantly
4. Add wife's account to the household Spotify Premium plan if not already, log her into Spotify on her phone with that account, repeat step 2

### Phase E — Picker UX decision (~30 min, ship in 2 hours)

The remaining gap: "I'm standing at Terry, I want to start music, no phone in hand."

Two options:
1. **Keep the Spotify APK launcher** (zero new code — it already works). Family launches APK on Terry, browses, plays → APK targets Terry's SDK as the Connect device. APK no longer plays audio itself; it's a remote control.
2. **Build a minimal in-dashboard picker** (a few hours): list of recent contexts via `/me/player/recently-played?limit=10`, tap any → `PUT /me/player/play` with that context_uri. No search, no browse — just "play one of the last 10 things you played." Maybe add a "Resume" button (last paused context).

Default to (1) for ship — the APK launcher is already there, no risk. Add (2) as a follow-on if the family ever complains about needing to grab a phone.

### Phase F — Strip the now-dead code (~1.5h)

After the new code is verified, delete:

- **Kiosk `MainActivity.kt`:**
  - `reliableMediaCmd()`, `dispatchMediaKey()`, `spotifyMediaController()`, `spotifyPlaybackState()`, `hasNotificationListenerAccess()`, `maybePromptNotificationAccess()` — all unused now
  - The `kiosk://spotify/{play,pause,prev,next}` branches in `handleSpotifyCommand()` — keep only `launch`
  - The deferred Notification-access prompt in `onCreate()`
- **Kiosk `NotificationListener.kt`:** delete entire file
- **`AndroidManifest.xml`:** remove the NotificationListener `<service>` block and `BIND_NOTIFICATION_LISTENER_SERVICE` if not used elsewhere
- **`index.html`:**
  - `spCommand()` polling re-issue logic
  - The `setInterval(loadSpotify, 60 * 1000)` background poll (SDK events replace it)
  - The Phase 1d resurrection loop (no longer needed)
  - The `<meta refresh>`-replacement soft-reset (would now wipe SDK state mid-track)
  - The `?t=<ms>` cache-bust in `spCommand` (no proxy roundtrip anymore)
- **Proxy `/api/spotify-now`:** keep for now as fallback/idle-card source. Can delete in a future cleanup if `lastPlayed` moves to direct SDK calls.

APK rebuild + reinstall on Terry (Mac Terminal):
```
/Users/elul/Projects/apps/kiosk-webview/build.sh && \
adb -s 192.168.1.114:5555 install -r app/build/outputs/apk/debug/app-debug.apk && \
terry restart
```

### Phase G — Soak + verify (1 day in real use)

12-item Phase 3 checklist from the 9.5 plan still applies. Plus new ones:
1. Audio plays for ≥1h continuously without glitches
2. Token refresh at the 50min mark is invisible (no audible pause)
3. Phone → Terry Connect target works from all family phones on the household account
4. Pause from phone reflects on dashboard within 1s
5. Skip from dashboard reflects on phone within 1s
6. Volume rocker on Portal controls WebView audio
7. Incoming Portal video call ducks/stops music (figure out which is right and make it consistent)
8. Dashboard reload (manual or automatic) re-initializes SDK within 5s
9. WebView crash recovery — kiosk auto-restart kicks in, SDK re-inits

---

## Risks

| Risk | Probability | Mitigation |
|---|---|---|
| Audio glitches / clicks during long playback | Medium | Phase B spike catches it; if present, fall back to keeping the APK as audio source and use SDK only for state events |
| Token refresh fails silently → audio stops | Medium | Visible error state already shipped in Phase 2 (stale badge); add explicit "Spotify token expired" line that prompts a click-to-refresh |
| Portal video calls fight with WebView audio focus | Medium-high | The current architecture has the same problem (APK loses audio focus); test specifically in Phase B |
| Premium subscription lapses → SDK rejects | Low | Family pays for Premium reliably; add a graceful degradation that falls back to the APK launcher if `account_error` fires |
| Dashboard reload kills the SDK player → audio stops | High if we don't think about it | Phase F bullet — remove or guard the soft-reset; if SDK is connected, skip the GC nudge entirely |
| Wife's account on a separate Premium plan can't target Terry | Inherent | Document. Add her phone to the household account, OR she uses Bluetooth pairing (today's fallback) |
| WebView SDK uses more memory than the APK approach → LMK reaps the kiosk | Medium | Phase G soak watches for this; if real, add the foreground-service trick that was cut from the 9.5 plan |

---

## Suggested ship order

Each phase is independently testable. Stop after C and you have a working SDK player on Terry (with APK still installed as fallback). Stop after E and you're at ~9.8. Phases F + G push to genuine 10.

1. **Phase A** (30 min) — OAuth scope re-issue
2. **Phase B** (1.5h) — audio playback spike, GO / NO-GO decision point #2 (real audio, not just SDK init)
3. **Phase C** (3-4h) — build the player layer in the dashboard
4. **Phase D** (1h) — Connect device flow verification
5. **Phase E** (30 min decision, 0-2h build) — picker UX
6. **Phase F** (1.5h) — strip dead code, rebuild APK
7. **Phase G** (1 day soak) — verify in real use

**Total active work:** 8-10h. **Calendar:** 2-3 days including soak.

---

## What's still NOT solved at 10/10

Even after this rebuild, two things remain:
- **Cross-Premium-account Connect** (wife on her own Premium subscription targeting Terry without joining household account) — Spotify's design constraint, no SDK workaround
- **Voice control** (hands wet from dishes use case) — separate project, Picovoice Porcupine + small Android service

If we want either, they're separate projects after this rebuild ships.
