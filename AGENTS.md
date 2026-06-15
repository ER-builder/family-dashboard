# Family Dashboard — AGENTS.md

Single-file HTML/CSS/JS dashboard that runs full-screen on a hacked Meta Portal device named **"Terry"** in the Rifman family kitchen (London). Family of 4: Elul + wife + Eitan + Tamar (kids).

URL: https://er-builder.github.io/family-dashboard/ (GitHub Pages, public). Auto-deploys from `main` in ~30–60s.

## Repo trio

| Repo | Path (Mac) | What it does |
| --- | --- | --- |
| `ER-builder/family-dashboard` (this) | `/Users/elul/Projects/family-dashboard/` | The dashboard — single `index.html` |
| `ER-builder/family-dashboard-proxy` | `/Users/elul/Projects/apps/family-dashboard-proxy/` | Vercel serverless function fetching the secret iCal URL → JSON |
| `kiosk-webview` (not a repo) | `/Users/elul/Projects/apps/kiosk-webview/` | Kotlin Android WebView kiosk on Terry. Built via `build.sh` from Mac Terminal.app (sandbox blocks Gradle) |

## Architecture

`index.html` is the entire dashboard: Tailwind-free vanilla CSS, vanilla JS. Per-section JS polls keep data fresh (calendar 5 min, Spotify 15 s, TfL bus 30 s, weather 15 min, stars 30 min); the `<meta http-equiv="refresh">` is set to 6 h purely as a memory-hygiene reset (was 5 min — dropped 2026-05-12 because the synchronized fetch burst dominated Vercel CPU and JS polls already cover freshness). State in `localStorage` (routine checks, celebration flags). No writeable lists — family uses Google Keep on phones for shared notes.

**Layout (B′ redesign, 2026-05-10):**
- Two-column grid: calendar 27.5% / right 72.5% (`0.55fr / 1.45fr`).
- Right column has TWO regions only:
  1. `.context-strip` (top, ~120px, always visible, horizontal): `cs-weather` (incl. 5-cell hourly forecast) + `cs-divider` + `cs-transit` + `cs-divider` + `cs-stars` (3-column grid per kid: name | 🎁 prizes | ⭐ cycle, with subtle vertical hairlines).
  2. `.primary-panel` (bottom, fills remaining ~500px, content swaps via `body[data-mode]`): one of `#morning-card` / `#evening-card` / `.spotify-card`.
- Mode picker: `pickMode()` returns `"morning" | "evening" | "spotify"` based on time + weekday. Updates every 60s.
- **Result:** Spotify card is the DEFAULT primary panel content — routine cards take over only during their windows. The Spotify card has two states via `data-state="idle"|"playing"`: idle shows a "no music playing — tap to launch" placeholder; playing shows full art + controls. The corner `#spotify-corner` launch button only appears during routine windows (CSS-gated by `body[data-mode]`).

**Calendar:** block-agenda view (B′ redesign) — vertical stack of chunky event cards. Today's events split into "Now" (currently happening, terracotta accent + bigger title) and "Up Next" sections. Tomorrow has its own dashed-rule day-head and recessed blocks (dimmer text, smaller title, indented). All-day chips strip stays on top. The prior 06–21 timeline view is gone.

**Previous designs (archived):** D+E.1 (2026-04-26): To-Do was the default primary panel + Google-Cal-style 06–21 timeline calendar. To-Do dropped 2026-05-10; the timeline lost out to the agenda blocks for kitchen-distance glanceability.

**Routines:** time-windowed (morning 06–09 weekdays only; evening 15:30–20:30 every day, London tz). **8 morning + 8 evening items per kid** (TV / Dessert / Reading dropped — celebration overlay handles the "all done" moment). MUST fit without scrolling. Items at 40px min-height; checkbox 28px. Celebration triggers once per kid per day per slot when all 8 are checked.

**Testing:** append `?mode=morning|evening|spotify` to the dashboard URL to force a mode for that page load (laptop/phone preview without waiting for the time window). Normal reload reverts to time-based logic. (`?mode=todo` is also accepted as a back-compat alias for `spotify`.)

## Hard constraints

- **No scroll on the routine card.** Kids tap items in order; scrolling breaks the flow. Layout decisions ladder up to this.
- **Touch targets ≥40px** for routine items (was ≥44px goal; 40px is the practical floor).
- **Width 1280, height 800** (Portal screen). Test against this — narrower dev browsers will lie about routine fit.
- **Hebrew + RTL must keep working** in calendar event titles. Heebo loaded as Hebrew fallback; Fraunces has no Hebrew so the cascade falls through automatically.
- **Never push secrets** — push protection blocks. The OpenWeather API key is a public key (intentional exception).
- **Apps Script TODO endpoint:** previously hard-coded; B′ redesign removed all reads/writes. The Sheet still exists if we ever want a read-only callout; nothing in the dashboard touches it now.

## Kiosk app (kiosk-webview) — Android-side facts

- **HOME registration is on a `<activity-alias>`**, not MainActivity directly. **Stays enabled at all times** — Aloha is permanent default Home (user picked "Always") so no snap-back risk. The `setComponentEnabledSetting(DISABLED)` step earlier iterations used is **gone** — do not reintroduce.
- **`SYSTEM_ALERT_WINDOW` permission is REQUIRED** (Android 10+ / SDK 29+, which Portal is). Without it, AutoReturnReceiver's `startActivity` is silently dropped — `appops get xyz.erapps.kiosk SYSTEM_ALERT_WINDOW` shows `default; rejectTime=…` against our background launches. Launcher status / HomeAlias enabled is NOT enough on its own (we tested). User must grant once via "Display over other apps"; MainActivity.onCreate auto-opens that settings page if `Settings.canDrawOverlays()` is false. **Same permission also powers the OverlayService floating button.**
- **Auto-return = `onStop` schedules an AlarmManager exact alarm; `onStart` cancels it.** Lifecycle hooks fire for ALL backgrounding (manual exit, incoming call, screen-off), so we get one consistent path. `AUTO_RETURN_MS` = **2 min** (was 5; shortened 2026-05-10 alongside the OverlayService rollout) — this is the flat timer for **non-Spotify** backgrounding. Spotify has its own music-gated rule (next bullet).
- **Receiver is call-aware:** before launching, checks `AudioManager.getMode()` for `MODE_IN_COMMUNICATION` (VoIP / Portal video) or `MODE_IN_CALL` (cellular). If active, reschedules for `RESCHEDULE_MS` (2 min) later instead of yanking out of the call.
- **Spotify is music-gated, not time-gated (changed 2026-06-12).** Earlier guidance said "Music is intentionally NOT a deferral signal — kids would leave Spotify on indefinitely." **That decision was reversed by Elul after a few weeks of real use — the flat 2-min yank out of Spotify was annoying.** New behavior: when the user is on Spotify (the `kiosk://spotify/launch` handler sets an `on_spotify` flag in `kiosk_prefs`), the receiver does NOT auto-return while music is playing — it polls `AudioManager.isMusicActive` every `MUSIC_POLL_MS` (30s) and stays out. Only after `MUSIC_GRACE_MS` (**2 min of continuous silence**) does it return to the dashboard. Brief pauses / track gaps under 2 min don't trigger it. State (`on_spotify`, `music_stopped_since`) lives in `SharedPreferences("kiosk_prefs")`, cleared in `onStart`. **Accepted tradeoff:** kids CAN now leave Spotify open indefinitely *as long as music keeps playing* — the manual OverlayService "← Dashboard" pill is the deliberate escape hatch. Non-Spotify apps still use the flat `AUTO_RETURN_MS`. **Confirmed by Elul 2026-06-14:** the silence countdown deliberately also counts "on Spotify but nothing playing" (e.g. browsing a playlist >2 min returns to the dashboard) — this is the intended literal "2 min of no music" behavior, NOT a bug. Do not add a `music_started` gate. *(This documents intended behavior; the change lives in the kiosk-webview `MainActivity.kt` + `AutoReturnReceiver` on the Mac — apply + rebuild the APK to make it live on Terry.)*
- **Screen-on while music plays (2026-06-15, verified on Terry):** Spotify only holds a PARTIAL/audio wake-lock, so once the music-gated change kept the kiosk on Spotify, Portal's screensaver took over after ~1 min and the screen blanked at the 5-min display timeout (the dashboard avoids this via its own `FLAG_KEEP_SCREEN_ON`). Fix lives entirely in `OverlayService`: it self-polls every `MUSIC_CHECK_MS` (15s) and toggles `FLAG_KEEP_SCREEN_ON` on its always-present overlay window — ON while `on_spotify && AudioManager.isMusicActive`, cleared the instant music stops so the screensaver + silence-return behave normally. Window flag, NOT a deprecated `PowerManager` wake-lock. Verified via `dumpsys power`: a `SCREEN_BRIGHT_WAKE_LOCK` with `WorkSource{<kiosk uid>}` is held while backgrounded-on-Spotify-with-music, released within ~15s of pause; `dumpsys dreams` shows no dream while playing.
- **OverlayService (added 2026-05-10):** floating "← Dashboard" pill drawn via `WindowManager` + `TYPE_APPLICATION_OVERLAY`. `MainActivity.onStop` starts it, `onStart` stops it. Sits top-right at `y=dp(68)` — `TYPE_APPLICATION_OVERLAY` (Android 8+) renders BELOW system UI, so anything <~50dp from the top gets covered by Portal's status bar (battery/wifi). Pill text 15sp, padding 18×11dp, semi-transparent aubergine background. Tap → launches MainActivity → onStart cleans up the service.
- **Spotify URI handlers (added 2026-05-10, control path rewritten 2026-05-13):** `MainActivity.shouldOverrideUrlLoading` intercepts `kiosk://spotify/{launch,play,pause,prev,next}`. `launch` opens the Spotify APK via `getLaunchIntentForPackage("com.spotify.music")`. Controls now use `AudioManager.dispatchMediaKeyEvent(KeyEvent(KEYCODE_MEDIA_PLAY|PAUSE|NEXT|PREVIOUS))` — the same media-key framework Bluetooth headsets use. **Do NOT revert to the old `com.spotify.mobile.android.ui.widget.{PLAY,PAUSE,PREVIOUS,NEXT}` broadcast intents** — that API was removed when Spotify migrated to MediaSession (works on no current Spotify build, including the pinned 8.6.98.900). Media keys route to whichever app holds the audio focus, so Spotify always responds as long as it has an active media session (i.e. has played something this session). Won't start playback from a cold app — use `launch` for that.
- **WiFi ADB endpoint:** `192.168.1.116:5555` (MAC `a4:0e:2b:74:d4:85`). `~/bin/terry` wrapper auto-discovers via ARP if cached IP shifts.
- **Portal launcher (Aloha):** `com.facebook.alohaapps.launcher` / activity `com.facebook.aloha.app.home.touch.HomeActivity`. Set as default Home; doesn't surface third-party LAUNCHER apps in its UI (kiosk only relaunchable via terry CLI, BootReceiver, or as registered HOME alternate).
- **Portal is degoogled:** no Google Play Services (`com.google.android.gms`), no Chrome stable. Only `org.chromium.chrome` for browsers. Has bearing on any 3rd-party app login that uses Custom Tabs / browser deep-linking — see Spotify section below.

## Spotify Web Playback SDK — attempted 2026-05-18, rolled back

The 10/10 path via SDK rebuild was attempted and rolled back same day. SDK
works in Portal's standalone Chromium browser (Widevine L1 + HW_SECURE_ALL
confirmed via Axinom). SDK does **NOT** work in the system WebView
(`android.webkit.WebView`) the kiosk uses — `INIT_ERR: Failed to initialize
player` on `player.connect()`, EME init silently fails despite the API
surface being present. UA shows `wv` token. Don't re-attempt the SDK
without first replacing the WebView component (launch standalone Chromium
or sideload a newer System WebView APK). Full retro at
`docs/plans/2026-05-18-sdk-rebuild-learnings.md`. Test files kept in repo
(`index-sdk.html`, `sdk-test-v2.html`) as reference; `/api/spotify-token`
proxy endpoint was deleted (security exposure — recreate from git history
if revisiting).

**Spotify Web Playback SDK scope set (for future re-attempts):**
`streaming user-read-email user-read-private user-read-playback-state user-modify-playback-state user-read-currently-playing user-read-recently-played`.
`user-read-private` is the easy-to-miss one — without it the SDK throws
`AUTH_ERR: Invalid token scopes`.

## Reliability patterns (2026-05-17 hardening pass)

These came out of the 5.5→9.5 plan and are now load-bearing. Don't regress:

- **Spotify-card buttons use single `click` handlers.** The old pattern (`click` + `touchend` with `e.preventDefault()`) silently suppresses the synthetic click on Portal's old Chromium → intermittent dead taps. CSS already has `touch-action: manipulation` on `.sp-ctrl-btn` / `.sp-launch-btn` / `#spotify-corner` / `#routine-back`, so there's no 350ms tap-delay reason to add `touchend`. Routine items have always used plain `click` and work fine — do not migrate them to PointerEvent without explicit on-device verification on Portal Chromium.
- **`spCommand()` cache-busts its post-tap polls** with `loadSpotify({ fresh: true })` → `?t=<ms>`. Without this, the 15s proxy edge cache returns stale `isPlaying:true` after a pause → the icon doesn't flip → the kid taps again → fires `pause→play`. Don't remove the `fresh` flag without also dropping the proxy `s-maxage`.
- **Explicit pause clears the music override immediately.** The 30s grace timer is *only* for proxy-observed stops (track gaps, brief pauses). User-initiated pauses skip grace via `if (cmd === "pause") setMusicOverride(false)` at the top of `spCommand`.
- **`#spotify-corner` is always visible.** A previous version hid it outside morning/evening modes, which left zero paths to launch the Spotify APK during default daytime when music was playing. Three independent launchers now exist while playing: the corner pill, the in-card "Open Spotify" pill, and long-press on the album art.
- **No `<meta http-equiv="refresh">`.** It wiped Spotify JS state mid-track every 6h → card flashed idle for one 15s poll cycle → suspected contributor to "Spotify randomly stops". Replaced with a `setInterval` soft-reset (nudges GC + forces a fresh poll) that doesn't touch the DOM.
- **Spotify resurrection logic:** if `isPlaying:false` for 3 consecutive polls AND `lastPlayed` unchanged AND we were previously playing AND we're in `spotify` mode (not a routine window), fire `kiosk://spotify/play` to auto-resume. Capped to 1 attempt per 5 min. Lives at the bottom of `index.html`.
- **Reliable media commands (kiosk side):** `MainActivity.reliableMediaCmd()` dispatches the media key, then 600ms later verifies playback state via `MediaSessionManager.getActiveSessions()`. If unchanged, falls back to `MediaController.transportControls` directly (bypasses key-event routing). If that fails too, re-launches Spotify (covers the "Spotify lost its media session entirely" case). **Requires Notification Listener grant** — but degrades gracefully if denied (just sends the bare media key, same as before). Grant flow surfaces on first launch ~3s after the overlay prompt; re-prompts at most once per 7 days. Diagnostic: `adb -s 192.168.1.116:5555 logcat -s TerryKiosk`.
- **NotificationListenerService stub** (`NotificationListener.kt`) is empty — it exists *only* so the user's grant unlocks `MediaSessionManager.getActiveSessions()`. Don't add notification reading; it's pure permission-gating.

## Old WebView gotchas (Portal runs an aging Chromium)

- **`element.hidden = bool` is unreliable.** Use `setAttribute("hidden","") + style.display="none"` together. Helper `setCardVisible(id, visible)` exists but is now unused — primary-panel mode switching uses CSS rules keyed off `body[data-mode]` instead.
- **`Intl.DateTimeFormat#formatToParts` may not exist.** Use `toLocaleTimeString({timeZone})` and regex-parse, with a `getHours()` fallback. See `nowMinutesLondon()` and `londonMins()`.
- **`:has()` selector** is risky — prefer body classes set by JS (e.g. `body[data-mode]`) for conditional layouts.
- **Abs-positioned children of `display: grid`** get auto-placed into a track on Portal Chromium (modern Chrome treats them as out-of-flow per spec). Symptom: a leading `position:absolute` decorator div pushes every real grid cell one column right and clips the last one off-card. Fix: render decorators via `::before`/`::after` pseudo-elements, never as a child div. Bit us 2026-05-11 on `.agenda .block .bar`.

## Design language

"The Rifman Almanac, Kitchen Edition" — editorial print on midnight-aubergine. **Each font has ONE job:** DM Serif Display = big glanceable numbers (clock + weather temp); Fraunces = prose; Public Sans = labels; JetBrains Mono = tabular alignment; Heebo + Frank Ruhl Libre = Hebrew fallback. Muted print accents: ochre, terracotta, sage, cobalt, lavender. **Eitan = cobalt, Tamar = terracotta** — load-bearing in kid card CSS, don't flip.

## Family facts that show up in code

- Location: London. Northern Line + Bus 102 (toward East Finchley / Golders Green).
- Kids' routines hard-coded in `ROUTINES` const.

## External APIs

- **OpenWeather** (free, key in source — public). `/weather` (current) + `/forecast` (3-hourly).
- **TfL** at `api.tfl.gov.uk/Line/{id}/Status` — **no auth needed** for line status. Severity scale: `≥10` Good, `7-9` Minor, `≤6` Severe. Bus 102 has no AVL feed → use `/Line/102/Timetable/{stopId}` for scheduled times rendered as countdown fallback.
- **Calendar** via `family-dashboard-proxy` Vercel function (`ICAL_URLS` comma-separated + `ICAL_LABELS` parallel labels).
- **Google Apps Script** for shared to-dos (endpoint hard-coded in `index.html`).
- **Table Stars** at `tablestars.erapps.xyz/api/public/stats?key=…` — keyed (`STATS_READ_KEY` in Vercel), wildcard CORS, returns `{kids: [{name, emoji, prize_count, cycle_progress}]}`. Powers the Stars slot in the context strip — per kid: `🎁 NAME N ⭐ X/10` (slow milestone count + fast daily-effort ratio). Both pulse on increment. Polled every 30 min.
- **Spotify** via `family-dashboard-proxy` Vercel function (`/api/spotify-now`). Refresh-token auth (`SPOTIFY_CLIENT_ID` + `SPOTIFY_CLIENT_SECRET` + `SPOTIFY_REFRESH_TOKEN` env vars, all sensitive). Returns `{isPlaying, track, artist, album, albumArt, progress, duration, spotifyUrl}` when playing, or `{isPlaying:false, lastPlayed: {track, artist, album, albumArt, spotifyUrl} | null}` when not. `lastPlayed` requires `user-read-recently-played` on the refresh token; degrades to `null` if scope is missing. Polled every 15 s; proxy uses adaptive edge cache — `s-maxage=15` while playing, `s-maxage=60` when idle (gives ~70-90% Vercel HIT rate, was 10s+8s = ~0% hit). See Spotify section below for the full integration.

## Spotify integration (shipped 2026-05-10)

**Three surfaces, all in `index.html` + `MainActivity.kt`:**

1. **Polled now-playing display** — drives the auto-opening full player card (no ambient strip; that was removed).
2. **Full player card** — `<section id="spotify-card">` inside `.primary-panel`. Visible when `body[data-mode="spotify"]` OR when `body[data-music-override="true"]` (music override). Two states via `data-state="idle|playing"`. **Idle state shows the last-played track** (album art with overlay play glyph + title + artist + green "Tap to play" button with Spotify glyph SVG). Album art is itself tappable in idle. Both fire `kiosk://spotify/launch`. No "Spotify on Terry/Portal" labelling — the green button + glyph carries the affordance. **Music override (added 2026-05-11):** false→true playback transition during morning/evening modes sets `body[data-music-override="true"]`, which CSS-overrides the routine card and shows the spotify card instead. A "← Back to Routine" pill appears bottom-left (replaces the Spotify-corner pill). Tap pill → clear override → routine returns. true→false transition starts a 30s grace timer before auto-clearing the override (avoids flapping on brief pauses / track gaps). `updateMode()` clears the override when the routine window ends (mode→spotify) since the override is meaningless outside routines. ⏮ ⏸ ⏭ buttons fire `kiosk://spotify/{cmd}` URIs that the kiosk WebView turns into Android broadcast intents to Spotify on Terry.
3. **Spotify corner button (`#spotify-corner`)** — bottom-left mirror of the exit button. Visible only during routine windows (CSS-gated by `body[data-mode]`). Smart click: music already playing → sets music override (swaps routine for player card); nothing playing → fires `kiosk://spotify/launch`.

**Spotify-on-Terry: currently 9.1.52.234 (May 2026 stable, arm64-v8a + 320-480dpi from APKMirror).** Upgraded from 8.6.98.900 via `adb install-multiple -r` on 2026-05-18 — existing logged-in session persisted through the upgrade, no re-auth required, materially better browsing UX. **Do NOT fresh-install any 9.x version** — Spotify 9.x forces browser-based OAuth via Custom Tabs, and Portal is degoogled (no Play Services, only bare Chromium), so the post-captcha `spotify://` deep-link callback never fires → infinite login loop. The upgrade-in-place path works because the existing session token is preserved by the install; a fresh install has no token to inherit.

**If you ever need to re-auth on Terry:** downgrade to 8.6.98.900 first (backup at `~/Downloads/spotify-8.6.98.900-backup.apk`, command: `adb -s 192.168.1.114:5555 install -r --downgrade <path>`), log in via 8.6.x's in-app email/password form (which bypasses the browser entirely), then upgrade back in-place to 9.x.

**Cross-account Connect:** Spotify Connect only lets devices on the same account target each other. To play to Terry from a non-Terry-account phone, options are: (a) share login (cleanest for shared family use); (b) Premium-only Spotify Jam (host targets Terry, shares ephemeral link/QR, guest joins); (c) Bluetooth-pair the phone to Terry (bypasses Spotify-on-Terry entirely; dashboard strip won't show the music since it's tied to the refresh-token's account).

**`updateMode()` quirk:** the time-of-day mode picker runs on a 60s interval and stomps `body[data-mode]` to `pickMode()`. Without a guard it would force-close the spotify card. Guard added 2026-05-10: `updateMode()` skips the override if current mode is already `"spotify"`. Don't remove the guard.

## Roadmap is authoritative

`ROADMAP.md` is the source of truth for next steps, open bugs, and shipped-feature notes. The "📚 Guides for Elul" section was for unbuilt features (multi-source iCal, Spotify) — both shipped, refer to the as-built notes. Update `ROADMAP.md` whenever you ship/change something. Open tests/follow-ups live in the **"✅ Open Tests / Follow-ups"** section there.
