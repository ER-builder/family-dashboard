# Family Dashboard — Roadmap / Future Ideas

A backlog of ideas for the Portal ("Terry") dashboard. Pick & build whenever.

---

## 🐞 Open Bugs

- **Morning + Evening routine visibility (deployed fix — verify on Terry).** 22:58 BST screenshot showed neither card visible ✓ (correct). New fix: uses `setAttribute("hidden","")` + `style.display="none"` together so old WebViews can't ignore either. Try/catch defaults to both hidden on any error. A `<span id="dbg">` in the bottom-left corner now shows `mins=NNN` live — glance at it on Terry to confirm `nowMinutesLondon()` is reading the right value. If mins look wrong, the WebView may not support `timeZone` in `toLocaleTimeString` — next step: derive London offset from UTC manually. **Verify over the full day cycle (morning window 06–09, evening 15:30–20:30).**

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

## 🏠 Smart Home Hub

1. **🚇 Live Transport** — TfL API (free) showing next trains/tubes from your nearest station. *"Northern Line: 2 min, 5 min, 8 min"*
2. **📦 Delivery Tracker** — Show expected deliveries today (parse from Gmail via Google API).
3. **🗓️ Family Countdown** — Big visual countdowns: *"Holiday in 23 days 🏖️"*, *"Mum's birthday in 5 days 🎂"*.
4. **🏫 School / Activities** — Daily schedule for kids: *"Swimming 4pm, Piano 5:30pm"*.
5. **💬 Family Message Board** — Shared note area anyone can update from their phone (Google Sheet backend, no server needed).
6. **🌅 Daily Photo Memory** — "On this day"-style memory surface from Google Photos.

## 📊 Useful Glanceable Info

7. ~~**☀️ Hourly Weather**~~ **Done.** Strip under the weather card shows next ~18h (6 × 3-hourly forecast points). Rain hours (codes 09/10/11) coloured cobalt. Same OpenWeather key, free `/forecast` endpoint.
8. **🗑️ Bin Day Reminder** — Color/alert changes the night before collection (big UK win). *Needs: which bin day(s) + which bins (recycling/general/garden).*
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
