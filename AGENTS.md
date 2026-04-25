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

**Layout (right column priority during routine windows):** weather → routine → todos → transit (transit hides during routine windows via `body.routine-active` so kids' card has full height).

**Calendar:** Google-Cal-style day view — 06:00–21:00 timeline (16 hours × 22px = 352px), all-day chips strip on top, terracotta "now" line, tomorrow as a compact list below.

**Routines:** time-windowed (morning 06–09, evening 15:30–20:30 London tz). 9 morning items per kid, 10 evening. **MUST fit without scrolling** — driving constraint for the whole layout. Items at 40px min-height (compromise from 44px for fit; checkbox is 28px).

## Hard constraints

- **No scroll on the routine card.** Kids tap items in order; scrolling breaks the flow. Layout decisions ladder up to this.
- **Touch targets ≥40px** (was ≥44px target; routine items dropped to 40px to fit all 10 evening items).
- **Width 1280, height 800** (Portal screen). Test against this — narrower dev browsers will lie about routine fit.
- **Hebrew + RTL must keep working** in calendar event titles. Heebo loaded as Hebrew fallback; Fraunces has no Hebrew so the cascade falls through automatically.
- **Never push secrets** — push protection blocks. The OpenWeather API key is a public key (intentional exception).
- **Apps Script TODO endpoint** is hard-coded in `index.html`. Don't change.

## Old WebView gotchas (Portal runs an aging Chromium)

- **`element.hidden = bool` is unreliable.** Use `setAttribute("hidden","") + style.display="none"` together (and `removeAttribute + style.display=""` to show). The dashboard uses a `setCardVisible(id, visible)` helper.
- **`Intl.DateTimeFormat#formatToParts` may not exist.** Use `toLocaleTimeString({timeZone})` and regex-parse, with a `getHours()` fallback. See `nowMinutesLondon()` and `londonMins()`.
- **`:has()` selector** is risky — prefer body classes set by JS (e.g. `body.routine-active`) for conditional layouts.
- **Cache:** APK currently uses `LOAD_DEFAULT` — until rebuilt, dashboard pushes need `pm clear` to be visible. Source already patched to `LOAD_NO_CACHE` in `MainActivity.kt`; pending rebuild.

## Design language

"The Rifman Almanac, Kitchen Edition" — editorial print on midnight-aubergine. Fraunces serif + Public Sans + JetBrains Mono + Heebo (Hebrew) + Frank Ruhl Libre (Hebrew fallback) via Google Fonts. Muted print accents: ochre, terracotta, sage, cobalt, lavender. **Eitan = cobalt, Tamar = terracotta** (these are load-bearing in the kid card CSS — don't flip).

## Family facts that show up in code

- Location: London. Northern Line + Bus 102 (toward East Finchley / Golders Green) shown in transit card.
- **Bins go out Tuesday.** Banner appears Mon 18:00 → Tue 09:00.
- World clock shows **Tel Aviv** (family abroad).
- Kids' routines hard-coded in `ROUTINES` const — edit there to add/change items.

## External APIs

- **OpenWeather** (free, key in source — public). `/weather` (current) + `/forecast` (3-hourly).
- **TfL** at `api.tfl.gov.uk/Line/{id}/Status` — **no auth needed** for line status. Severity scale: `≥10` Good, `7-9` Minor, `≤6` Severe. Bus 102 has no AVL feed → use `/Line/102/Timetable/{stopId}` for scheduled times rendered as countdown fallback.
- **Calendar** via `family-dashboard-proxy` Vercel function (single `ICAL_URL` env var; multi-source merge is on roadmap).
- **Google Apps Script** for shared to-dos (endpoint hard-coded in `index.html`).
- **Table Stars** at `tablestars.erapps.xyz/api/public/stats?key=…` — keyed (`STATS_READ_KEY` in Vercel), wildcard CORS, returns `{kids: [{name, emoji, prize_count, cycle_progress}]}`. Powers the Stars card in the right column (off-routine windows only, hidden via `body.routine-active`). Polled every 30 min.

## Roadmap is authoritative

`ROADMAP.md` is the source of truth for next steps, open bugs, and "📚 Guides for Elul" (Spotify setup walkthrough, multi-source iCal merge instructions). Update it whenever you ship/change something.
