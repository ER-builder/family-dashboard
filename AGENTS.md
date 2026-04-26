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

`index.html` is the entire dashboard: Tailwind-free vanilla CSS, vanilla JS. Auto-refresh every 5 min via `<meta http-equiv="refresh">`. All state in `localStorage` (routine checks, celebration flags) except shared to-dos which live in a Google Sheet via Apps Script endpoint.

**Layout (UX redesign D + E.1, 2026-04-26):**
- Two-column grid: calendar 27.5% / right 72.5% (`0.55fr / 1.45fr`).
- Right column has TWO regions only:
  1. `.context-strip` (top, ~120px, always visible, horizontal): `cs-weather` + `cs-divider` + `cs-transit` + `cs-divider` + `cs-stars`. Compact form of three former cards.
  2. `.primary-panel` (bottom, fills remaining ~500px, content swaps via `body[data-mode]`): one of `#morning-card` / `#evening-card` / `.todos`.
- Mode picker: `pickMode()` returns `"morning" | "evening" | "todo"` based on time + weekday. Updates every 60s.
- **Result:** To-Do is the DEFAULT primary panel content (visible most of the day) — routines take it over only during their windows. To-Do reachable in every state — fixed the v0 bug where it was crushed to invisibility.

**Calendar:** Google-Cal-style day view — 06:00–21:00 timeline (16 hours × 22px = 352px), all-day chips strip on top, terracotta "now" line, tomorrow as a compact list below.

**Routines:** time-windowed (morning 06–09, evening 15:30–20:30 London tz). 9 morning items per kid, 10 evening. **MUST fit without scrolling** — driving constraint for the whole layout. Items at 40px min-height (compromise from 44px for fit; checkbox is 28px).

## Hard constraints

- **No scroll on the routine card.** Kids tap items in order; scrolling breaks the flow. Layout decisions ladder up to this.
- **Touch targets ≥40px** (was ≥44px target; routine items dropped to 40px to fit all 10 evening items).
- **Width 1280, height 800** (Portal screen). Test against this — narrower dev browsers will lie about routine fit.
- **Hebrew + RTL must keep working** in calendar event titles. Heebo loaded as Hebrew fallback; Fraunces has no Hebrew so the cascade falls through automatically.
- **Never push secrets** — push protection blocks. The OpenWeather API key is a public key (intentional exception).
- **Apps Script TODO endpoint** is hard-coded in `index.html`. Don't change.

## Kiosk app (kiosk-webview) — Android-side facts

- **HOME registration is a runtime-toggleable `<activity-alias>`**, not on MainActivity directly. Disabled via `PackageManager.setComponentEnabledSetting` *before* exit; otherwise Android's home-resolution snaps right back. Re-enabled in `onCreate`. Only iteration that broke the snap-back loop after 4 prior attempts.
- **Auto-return = `onStop` schedules an AlarmManager exact alarm; `onStart` cancels it.** Lifecycle hooks fire for ALL backgrounding (manual exit, incoming call, screen-off), so we get one consistent path. `AUTO_RETURN_MS` = 5 min.
- **Receiver is call-aware:** before launching, checks `AudioManager.getMode()` for `MODE_IN_COMMUNICATION` (VoIP / Portal video) or `MODE_IN_CALL` (cellular). If active, reschedules itself for 5 min later instead of yanking out of the call. No permissions needed.
- **WiFi ADB endpoint:** `192.168.1.116:5555` (MAC `a4:0e:2b:74:d4:85`). `~/bin/terry` wrapper auto-discovers via ARP if cached IP shifts.
- **Portal launcher (Aloha):** `com.facebook.alohaapps.launcher` / activity `com.facebook.aloha.app.home.touch.HomeActivity`. Set as default Home; doesn't surface third-party LAUNCHER apps in its UI (kiosk only relaunchable via terry CLI, BootReceiver, or as registered HOME alternate).

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
- **Calendar** via `family-dashboard-proxy` Vercel function (single `ICAL_URL` env var; multi-source merge is on roadmap).
- **Google Apps Script** for shared to-dos (endpoint hard-coded in `index.html`).
- **Table Stars** at `tablestars.erapps.xyz/api/public/stats?key=…` — keyed (`STATS_READ_KEY` in Vercel), wildcard CORS, returns `{kids: [{name, emoji, prize_count, cycle_progress}]}`. Powers the Stars slot in the context strip (compact: `🎁 Name N` per kid, no pips in v1). Polled every 30 min.

## Roadmap is authoritative

`ROADMAP.md` is the source of truth for next steps, open bugs, and "📚 Guides for Elul" (Spotify setup walkthrough, multi-source iCal merge instructions). Update it whenever you ship/change something.
