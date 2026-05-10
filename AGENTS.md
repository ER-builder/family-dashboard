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

`index.html` is the entire dashboard: Tailwind-free vanilla CSS, vanilla JS. Auto-refresh every 5 min via `<meta http-equiv="refresh">`. State in `localStorage` (routine checks, celebration flags). No writeable lists — family uses Google Keep on phones for shared notes.

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
- **Auto-return = `onStop` schedules an AlarmManager exact alarm; `onStart` cancels it.** Lifecycle hooks fire for ALL backgrounding (manual exit, incoming call, screen-off), so we get one consistent path. `AUTO_RETURN_MS` = **2 min** (was 5; shortened 2026-05-10 alongside the OverlayService rollout — kids can't leave Spotify open forever).
- **Receiver is call-aware:** before launching, checks `AudioManager.getMode()` for `MODE_IN_COMMUNICATION` (VoIP / Portal video) or `MODE_IN_CALL` (cellular). If active, reschedules for `RESCHEDULE_MS` (2 min) later instead of yanking out of the call. **Music is intentionally NOT a deferral signal** — defer-while-music-plays was tried and rejected because kids would leave Spotify on indefinitely.
- **OverlayService (added 2026-05-10):** floating "← Dashboard" pill drawn via `WindowManager` + `TYPE_APPLICATION_OVERLAY`. `MainActivity.onStop` starts it, `onStart` stops it. Sits top-right at `y=dp(68)` — `TYPE_APPLICATION_OVERLAY` (Android 8+) renders BELOW system UI, so anything <~50dp from the top gets covered by Portal's status bar (battery/wifi). Pill text 15sp, padding 18×11dp, semi-transparent aubergine background. Tap → launches MainActivity → onStart cleans up the service.
- **Spotify URI handlers (added 2026-05-10):** `MainActivity.shouldOverrideUrlLoading` intercepts `kiosk://spotify/{launch,play,pause,prev,next}`. `launch` opens the Spotify APK via `getLaunchIntentForPackage("com.spotify.music")`; controls send broadcast intents (`com.spotify.mobile.android.ui.widget.{PLAY,PAUSE,PREVIOUS,NEXT}`) targeted at the Spotify package. Broadcasts only affect the local Spotify instance — Spotify-on-phone playback can't be controlled this way.
- **WiFi ADB endpoint:** `192.168.1.116:5555` (MAC `a4:0e:2b:74:d4:85`). `~/bin/terry` wrapper auto-discovers via ARP if cached IP shifts.
- **Portal launcher (Aloha):** `com.facebook.alohaapps.launcher` / activity `com.facebook.aloha.app.home.touch.HomeActivity`. Set as default Home; doesn't surface third-party LAUNCHER apps in its UI (kiosk only relaunchable via terry CLI, BootReceiver, or as registered HOME alternate).
- **Portal is degoogled:** no Google Play Services (`com.google.android.gms`), no Chrome stable. Only `org.chromium.chrome` for browsers. Has bearing on any 3rd-party app login that uses Custom Tabs / browser deep-linking — see Spotify section below.

## Old WebView gotchas (Portal runs an aging Chromium)

- **`element.hidden = bool` is unreliable.** Use `setAttribute("hidden","") + style.display="none"` together. Helper `setCardVisible(id, visible)` exists but is now unused — primary-panel mode switching uses CSS rules keyed off `body[data-mode]` instead.
- **`Intl.DateTimeFormat#formatToParts` may not exist.** Use `toLocaleTimeString({timeZone})` and regex-parse, with a `getHours()` fallback. See `nowMinutesLondon()` and `londonMins()`.
- **`:has()` selector** is risky — prefer body classes set by JS (e.g. `body[data-mode]`) for conditional layouts.

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
- **Spotify** via `family-dashboard-proxy` Vercel function (`/api/spotify-now`). Refresh-token auth (`SPOTIFY_CLIENT_ID` + `SPOTIFY_CLIENT_SECRET` + `SPOTIFY_REFRESH_TOKEN` env vars, all sensitive). Returns `{isPlaying, track, artist, album, albumArt, progress, duration, spotifyUrl}` or `{isPlaying:false}`. Polled every 10s; proxy caches 8s. See Spotify section below for the full integration.

## Spotify integration (shipped 2026-05-10)

**Three surfaces, all in `index.html` + `MainActivity.kt`:**

1. **Polled now-playing display** — drives the auto-opening full player card (no ambient strip; that was removed).
2. **Full player card** — `<section id="spotify-card">` inside `.primary-panel`. Visible only when `body[data-mode="spotify"]`. Auto-opens on the false→true playback transition **only when current mode is `todo`** (kids' morning/evening checklists are load-bearing — music doesn't yank them). Auto-closes after 90s back into the time-of-day mode (calls `updateMode()`, not a hard-coded "todo" — important for sessions that span a routine boundary). ⏮ ⏸ ⏭ buttons fire `kiosk://spotify/{cmd}` URIs that the kiosk WebView turns into Android broadcast intents to Spotify on Terry.
3. **Green ♫ corner button** — bottom-left mirror of the exit button. Smart click: music playing → opens the dashboard's full player card; nothing playing → fires `kiosk://spotify/launch`.

**Critical Spotify-on-Terry version pin: 8.6.98.900 universal nodpi from APKMirror.** Spotify 9.x forces browser-based auth via Custom Tabs; Portal is degoogled (no Play Services, only bare Chromium), so the post-captcha `spotify://` deep-link callback never fires → infinite login loop. **Do not "upgrade" Spotify on Terry.** 8.6.x has the in-app email/password form that bypasses the browser entirely. Tolerate the "outdated app" nags from Spotify.

**Cross-account Connect:** Spotify Connect only lets devices on the same account target each other. To play to Terry from a non-Terry-account phone, options are: (a) share login (cleanest for shared family use); (b) Premium-only Spotify Jam (host targets Terry, shares ephemeral link/QR, guest joins); (c) Bluetooth-pair the phone to Terry (bypasses Spotify-on-Terry entirely; dashboard strip won't show the music since it's tied to the refresh-token's account).

**`updateMode()` quirk:** the time-of-day mode picker runs on a 60s interval and stomps `body[data-mode]` to `pickMode()`. Without a guard it would force-close the spotify card. Guard added 2026-05-10: `updateMode()` skips the override if current mode is already `"spotify"`. Don't remove the guard.

## Roadmap is authoritative

`ROADMAP.md` is the source of truth for next steps, open bugs, and shipped-feature notes. The "📚 Guides for Elul" section was for unbuilt features (multi-source iCal, Spotify) — both shipped, refer to the as-built notes. Update `ROADMAP.md` whenever you ship/change something. Open tests/follow-ups live in the **"✅ Open Tests / Follow-ups"** section there.
