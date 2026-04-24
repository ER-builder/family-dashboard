# Family Dashboard — Roadmap / Future Ideas

A backlog of ideas for the Portal ("Terry") dashboard. Pick & build whenever.

---

## 🧠 Re-evaluate / Refine (queued 2026-04-24)

- ~~**Hide morning routine on Sat & Sun.**~~ **Done 2026-04-24.** Added `isWeekendLondon()` helper using `toLocaleDateString("en-GB", { weekday })` with a `getDay()` fallback for old WebViews. Morning gated by `!weekend && hour-window`; evening unchanged.
- **Bin reminder banner — is it pulling its weight?** Tuesday bin day already shows up in the family Google Calendar (recurring event), so the dedicated banner from Mon 18:00 → Tue 09:00 is duplicative. Two options: (a) **delete the banner entirely** (calendar handles it, less UI clutter), or (b) **keep but make it smarter** — only show during a tighter window (e.g. Mon 19:00 → Mon 23:00 only, as a true "act now" reminder; or Tue 06:00–08:00 as a final wake-up nudge). Decide which after living with both for a week.
- **Layout outside routine windows.** When neither morning (06–09) nor evening (15:30–20:30) is showing — i.e. ~09:00–15:29 weekdays + most of weekends — the right column has weather + transit + todos. Currently todos uses `flex: 1` so it grows, but visually the column may look unbalanced (lots of vertical todos). Consider: (a) different cards rotate into the routine slot during off-windows (e.g. Drive folder shortcuts, family photo of the day, "today in history" card), or (b) the calendar/timeline card grows wider during off-windows, or (c) a single "ambient" card (rotating quote, photo, or family fact) fills the slot. **Recommendation:** start with (c) — one card, low friction, lots of design upside.

## 🐞 Open Bugs

- **Morning + Evening routine visibility — Morning confirmed working 2026-04-24.** 22:58 BST and morning windows both verified ✓. Fix uses `setAttribute("hidden","")` + `style.display="none"` together so old WebViews can't ignore either. A `<span id="dbg">` in the bottom-left corner shows `mins=NNN` live. **Still to verify: evening window (15:30–20:30).** Once evening is also confirmed, remove the `dbg` indicator (search `id="dbg"` in `index.html`).

## 🔧 Pending Infra (needs Mac Terminal.app + Terry connected via USB)

0. **Rebuild + reinstall kiosk APK.** Two source changes are already made:
   - `LOAD_NO_CACHE` cache mode (no more stale dashboard) — already in MainActivity.kt
   - `shouldOverrideUrlLoading` for `kiosk://exit` URL — **still needs adding** to MainActivity.kt's WebViewClient:
     ```kotlin
     override fun shouldOverrideUrlLoading(view: WebView, request: WebResourceRequest): Boolean {
         if (request.url.scheme == "kiosk" && request.url.host == "exit") {
             finishAndRemoveTask()
             return true
         }
         return super.shouldOverrideUrlLoading(view, request)
     }
     ```
   Then build + install from Terminal.app:
   ```
   /Users/elul/Projects/apps/kiosk-webview/build.sh && \
   adb install -r /Users/elul/Projects/apps/kiosk-webview/app/build/outputs/apk/debug/app-debug.apk && \
   adb shell am force-stop xyz.erapps.kiosk && \
   adb shell am start -n xyz.erapps.kiosk/.MainActivity
   ```
   After this: every dashboard push propagates on the 5-min meta-refresh (no `pm clear`), and long-pressing the bottom-right corner for 2s exits the kiosk.

---

## 🧭 Open Question — Multi-Screen / Swipe Gesture

Should the dashboard add a swipe-left gesture to flip to a second (or third) "page"? Could host things that don't need to be glanceable at the kitchen at 7am: photo slideshow, recipe browser, message-board archive, kid-mode (just routines fullscreen), or a dedicated weather/transit screen.

**Trade-off:** A glance device works best with one rich screen — adding more dilutes "look up and see what matters." Probably overkill *unless* a specific feature genuinely doesn't fit on the main screen.

**Recommendation:** Defer until we have something that demands it. If we do add it, the natural triggers would be:
- A long route-specific transit detail (next 5 buses + walk time)
- A photo-frame mode kids could swipe to in idle moments
- A "Shabbat candle-lighting time + parsha" screen for Friday afternoons

Implementation note: Portal Android WebView supports `touchstart`/`touchend` — swipe could be detected with ~30 lines of JS. Each screen would be a `<section>` in DOM with CSS `transform: translateX()` to swap. Same auto-refresh mechanism still works.

---

## 🏠 Smart Home Hub

1. ~~**🚇 Live Transport (status)**~~ **Done 2026-04-24.** Northern Line + Bus 102 (toward E. Finchley / Golders Green) live status from `api.tfl.gov.uk/Line/{id}/Status` (no key needed). Color-coded: sage = good, ochre = minor, terracotta = severe. Polled every 5 min.
1a. **🚌 Live 102 bus arrivals — framework shipped, awaiting stop IDs.** `index.html` now has a `TFL_STOPS` array (just below the line-status loader). Add entries like `{ id: "490004733E", route: "102", dest: "Edgware" }` — IDs come from clicking a stop on https://tfl.gov.uk/bus/route/102/ (URL contains the StopID). Add one entry per direction. Once filled in, arrivals show under the 102 row, polled every 30s, with "Due" highlighted in ochre.
2. **📦 Delivery Tracker** — Show expected deliveries today (parse from Gmail via Google API).
3. **🗓️ Family Countdown** — Big visual countdowns: *"Holiday in 23 days 🏖️"*, *"Mum's birthday in 5 days 🎂"*.
4. **🏫 School / Activities** — Daily schedule for kids: *"Swimming 4pm, Piano 5:30pm"*.
5. **💬 Family Message Board** — Shared note area anyone can update from their phone (Google Sheet backend, no server needed).
6. **🌅 Daily Photo Memory** — "On this day"-style memory surface from Google Photos.

## 📊 Useful Glanceable Info

7. ~~**☀️ Hourly Weather**~~ **Done.** Strip under the weather card shows next ~18h (6 × 3-hourly forecast points). Rain hours (codes 09/10/11) coloured cobalt. Same OpenWeather key, free `/forecast` endpoint.
8. ~~**🗑️ Bin Day Reminder**~~ **Done 2026-04-24.** Banner above masthead from Mon 18:00 → Tue 09:00 (London tz). "Bins out tomorrow morning" Mon evening, "Don't forget the bins!" Tue early. *Future: extend to multiple bin types (e.g. recycling Wed, garden Thu) — change `updateBinBanner()` to a schedule lookup.*
9. **🛒 Shared Shopping List** — Google Sheet backend, anyone adds from their phone. *Needs: new Apps Script endpoint or extra tab in existing "Family Dashboard" sheet.*
10. **📰 News Headlines** — BBC RSS, scrolling ticker at the bottom. *Needs: CORS proxy (could extend the Vercel proxy).*
11. **💷 Budget Tracker** — Monthly family spending from a Google Sheet (or Rifman Family Budget Firestore).

## 📅 Calendar — Multi-Source Merge

- **Let the proxy merge multiple iCal feeds.** Today the Vercel proxy reads a single `ICAL_URL` env var (the family-shared Google Calendar). Manually copying personal events to the family cal isn't sustainable.
- Change to: `ICAL_URLS` (comma-separated list of secret iCal URLs). The serverless function fetches each, concatenates the parsed VEVENTs, dedupes by UID, and returns one merged sorted JSON list. Each event optionally tagged with a `source` label (e.g. "Family", "Elul", "School") so the dashboard could color-code in future.
- Repo: `ER-builder/family-dashboard-proxy` at `~/Projects/apps/family-dashboard-proxy/`. Single file `api/cal.js`. Update env var via Vercel dashboard or CLI (`vercel env rm ICAL_URL && printf '%s' "$URLS" | vercel env add ICAL_URLS production --sensitive`). Note: each source is a *secret iCal URL* (Settings → Integrate calendar → Secret address in iCal format) — not a public URL. Calendars stay private.

## 🚪 Exit-to-Portal

- **HTML side: done.** Bottom-right corner has a transparent 72×72px button. Long-pressing it for 2 seconds fires `window.location.href = "kiosk://exit"`.
- **Kotlin side: pending** (part of APK rebuild above). Once `shouldOverrideUrlLoading` intercepts `kiosk://exit` and calls `finishAndRemoveTask()`, the gesture will return to Portal's launcher.
- Caveat: `BootReceiver` re-launches the kiosk on next boot — this is "exit for now," not permanent.

## 🎨 Design Refinement

- ~~**To-Do notebook lines don't align with task rows.**~~ **Done.** Replaced repeating-gradient with `border-bottom` on each `<li>`. Removed static terracotta margin line.
- ~~**Hebrew font in calendar events looks bad.**~~ **Done.** Added Heebo (wght 300–700) to Google Fonts; inserted before Frank Ruhl Libre in `--serif` and `--sans` stacks. Fraunces doesn't cover Hebrew so Hebrew chars fall through to Heebo automatically.
- ~~**"The Rifman Almanac · Kitchen Edition" brand line is too small and unnecessary.**~~ **Done.** Removed. Greeting is now slightly larger (clamp 30–44px).
- ~~**Calendar card huge and empty; routine card too narrow.**~~ **Done 2026-04-24.** Grid changed `1.65fr / 1fr` → `0.95fr / 1.1fr` (kid columns ~290px, was ~210). Calendar rebuilt as Google-Cal-style day view: 06:00–22:00 timeline with hour markers, all-day chips strip, terracotta "now" line, future days as compact list below. Eyebrow meta = TODAY.
- **Follow-up if routines still feel cramped during active windows:** swap left/right placements via a body class set by JS (e.g. `body.mode-morning` / `body.mode-evening` → routine takes the wider left slot, calendar moves right). Not built — see how the current resize feels first.
- **Weather card is wide — open slot for a sibling.** After the right column got wider, the weather card spans the full width and feels stretched. We can split it into a 2-column row inside the right column: weather on the left (~60%), and a small companion card on the right (~40%). Good candidates for the companion: TfL transit (always-visible instead of routine-conditional), Family Countdowns, Bin schedule (multi-bin), Spotify Now Playing, Daily Photo, or a "next event in N min" callout. Easy implementation: wrap weather + companion in a `<div class="wx-row">` that's `display: grid; grid-template-columns: 1.5fr 1fr` inside `.right`.

## ✨ UX Polish

- ~~**Celebration animation when a kid completes their morning routine.**~~ **Done.** Toast "🎉 You did it!" + kid's name in their accent colour, 44 confetti emoji falling, kid's name pulses for 1.3s. Once-per-day per kid via `localStorage` key `celebrated:YYYY-MM-DD:{kid}`. Morning only.
- ~~**To-Dos fade out 15s after being checked.**~~ **Done.** Tick → 15s grace → CSS `.fading` (opacity + slide-right 350ms) → API delete. Un-tick within window cancels timer. Items already-done from a phone (caught on the 60s poll) get the same 15s grace.

## 🎮 Fun / Vibe

12. **🎵 Spotify "Now Playing"** — What's playing on the home speaker (free Spotify API). *Needs: OAuth flow.*
13. ~~**🌍 World Clock**~~ **Done.** Tel Aviv time line under the masthead date (🇮🇱 + city + time, mono). Add more cities by extending the `tick()` worldclock block.
14. **📸 Photo Slideshow Background** — Rotate family photos behind the dashboard cards. *Needs: photo source (Drive folder? Google Photos OAuth?).*
15. **🐾 Chore Wheel** — Random daily chore assignments per family member, resets each morning. *Needs: chore list.*
16. ~~**💡 Daily Quote / Mantra**~~ **Done.** 36 rotating quotes, picked by day-of-year, fixed at bottom-center of screen in subtle italic. Edit the `QUOTES` array in `index.html` to add more.

---

## 📚 Guides for Elul (work that needs your hands, not Pi's)

### Spotify "Now Playing" — full setup
**Goal:** show what's playing on the home Spotify on the dashboard.

1. **Register the app.** Go to https://developer.spotify.com/dashboard, log in with your personal Spotify account, "Create app". Note the **Client ID** and **Client Secret**.
2. **Add a redirect URI** in the app settings — use your Vercel proxy URL: `https://family-dashboard-proxy.vercel.app/api/spotify-callback`.
3. **Do the OAuth dance once** (manual, browser-based) to capture a **refresh token**:
   - Open this URL in a browser (substitute `<CLIENT_ID>`):
     `https://accounts.spotify.com/authorize?client_id=<CLIENT_ID>&response_type=code&redirect_uri=https%3A%2F%2Ffamily-dashboard-proxy.vercel.app%2Fapi%2Fspotify-callback&scope=user-read-currently-playing+user-read-playback-state`
   - Authorize → callback URL contains `?code=...`. Capture that `code`.
   - In a terminal, exchange `code` for tokens:
     ```bash
     curl -X POST https://accounts.spotify.com/api/token \
       -d grant_type=authorization_code \
       -d code=<CODE> \
       -d redirect_uri=https://family-dashboard-proxy.vercel.app/api/spotify-callback \
       -u <CLIENT_ID>:<CLIENT_SECRET>
     ```
   - Response includes `refresh_token`. Store it in Vercel env: `vercel env add SPOTIFY_REFRESH_TOKEN production --sensitive`. Also add `SPOTIFY_CLIENT_ID` + `SPOTIFY_CLIENT_SECRET`.
4. **Add `/api/spotify-now`** to the proxy repo (`family-dashboard-proxy/api/spotify-now.js`):
   - Use refresh token to fetch a fresh access token (cache it in module-scope memory for ~50 min)
   - Call `https://api.spotify.com/v1/me/player/currently-playing` with `Authorization: Bearer <accessToken>`
   - Return `{ track, artist, isPlaying, albumArt }` JSON; return `{ isPlaying: false }` if 204.
5. **Dashboard side** — add a small Spotify card that polls `/api/spotify-now` every 30s. Hide card when `!isPlaying`.

### Multi-source iCal merge — full setup
**Goal:** dashboard calendar shows events from multiple Google Calendars (family-shared + your personal + maybe school's published cal).

In the proxy repo (`ER-builder/family-dashboard-proxy`, `api/cal.js`):
1. Replace `process.env.ICAL_URL` lookup with:
   ```js
   const urls = (process.env.ICAL_URLS || process.env.ICAL_URL || "")
     .split(",")
     .map(s => s.trim())
     .filter(Boolean);
   ```
2. Fetch all in parallel:
   ```js
   const responses = await Promise.all(urls.map(u => fetch(u).then(r => r.text())));
   const events = responses.flatMap((ics, i) => parseIcs(ics).map(ev => ({ ...ev, source: i })));
   ```
   (`source` is just a numeric index for now; could become a label later.)
3. Dedupe by `uid` (last write wins is fine):
   ```js
   const byUid = new Map();
   for (const ev of events) byUid.set(ev.uid, ev);
   const merged = [...byUid.values()].sort((a, b) => new Date(a.start) - new Date(b.start));
   ```
4. Return `{ events: merged }` as before.
5. **Rotate Vercel env** (from Pi or Mac):
   ```bash
   cd ~/Projects/apps/family-dashboard-proxy
   vercel env rm ICAL_URL production -y
   printf '%s' "<URL1>,<URL2>,<URL3>" | vercel env add ICAL_URLS production --sensitive
   vercel --prod
   ```
6. **Get each secret iCal URL:** Google Calendar → click the calendar → Settings and sharing → scroll to **Integrate calendar** → **Secret address in iCal format**. Copy that URL. Per-calendar, not per-event. Treat it like a password (anyone with that URL can read all events).

The dashboard side (the day view we just built) needs no changes — it already reads `events` from the proxy.

---

## 🧠 Power Move: Google Sheets as Backend

For everything that needs cross-device editing (lists, countdowns, messages, budget), use **Google Sheets as the database**:

- ✅ Free
- ✅ Editable from any phone/laptop
- ✅ Public JSON API (no server needed — `gviz/tq?tqx=out:json`)
- ✅ Perfect for shopping lists, to-dos, countdowns, message board, budget

Dashboard polls the sheet every few minutes. Family edits on phone → shows on Terry automatically.

---

## Top 3 Picks (Initial Recommendation)

| Priority | Feature | Why |
| --- | --- | --- |
| 1️⃣ | 🚇 TfL Live Departures | Glance on the way out the door |
| 2️⃣ | 🗓️ Family Countdowns | Kids love it, builds excitement |
| 3️⃣ | 💬 Sheets message board + shopping list | Whole family contributes from their phones |
