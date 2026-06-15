# Family Dashboard — Roadmap / Future Ideas

A backlog of ideas for the Portal ("Terry") dashboard. Pick & build whenever.

---

## 🧠 Re-evaluate / Refine (queued 2026-04-24)

- ~~**Morning card needs to fit ALL 9 items at a glance.**~~ **Done 2026-04-24.** Removed Hi/Lo/Feels stats from weather card to reclaim ~24px of vertical space. JS guarded with null checks so the missing #w-hi/#w-lo/#w-feels elements don't throw. **Verify on Terry tomorrow morning** — if still ≤7/9 visible, drop routine item font 17px → 15px (touch-target floor: keep min-height ≥36px).
- ~~**Hide morning routine on Sat & Sun.**~~ **Done 2026-04-24.** Added `isWeekendLondon()` helper using `toLocaleDateString("en-GB", { weekday })` with a `getDay()` fallback for old WebViews. Morning gated by `!weekend && hour-window`; evening unchanged.
- ~~**Bin reminder banner.**~~ **Removed 2026-04-24.** Two reasons: (1) it duplicated the calendar event (calendar is now source of truth via merged iCal), (2) the schedule was wrong anyway — bins are picked up Wed early morning, banner was set up for Tue. Calendar entry handles the Tue evening "put bins on sidewalk" reminder cleanly. All `bin-banner` markup, CSS, and `updateBinBanner()` removed from `index.html`.
- ~~**Layout outside routine windows — first ambient card shipped.**~~ **Done 2026-04-24.** Stars card filled the slot. **Superseded 2026-04-26 by the D+E.1 redesign** — there's no "ambient slot" anymore; To-Do is the default primary panel.
- ~~**To-Do unreachable during routine windows + crushed off-routine.**~~ **Fixed 2026-04-26 by the D+E.1 redesign.** Right column restructured into context strip (top, weather+transit+stars compacted) + primary panel (bottom, swaps morning/evening/todo via `body[data-mode]`). To-Do is now the DEFAULT view, visible all day except during routine windows. See `UX_REDESIGN_PLAN.md` for the full rationale and decisions.

## 🐞 Open Bugs

- **Auto-return real fix (2026-04-27, awaiting user to grant overlay permission once).** Two-iter saga: iter 1 (HomeAlias kept enabled for launcher exemption) didn't work — `appops get xyz.erapps.kiosk SYSTEM_ALERT_WINDOW` showed `default; rejectTime=...` against our background activity launches on Android 10 (Portal SDK 29). Launcher status alone isn't enough; Android 10+'s background-activity-start rule requires SYSTEM_ALERT_WINDOW. **Iter 2:** added `<uses-permission SYSTEM_ALERT_WINDOW>` to manifest + onCreate auto-launches `Settings.ACTION_MANAGE_OVERLAY_PERMISSION` on first run if not granted. User taps "Allow" once in Portal Settings → "Display over other apps" → `Settings.canDrawOverlays(this)` returns true → background launches succeed. Diagnostic command: `adb shell appops get xyz.erapps.kiosk SYSTEM_ALERT_WINDOW`.

- **Morning + Evening routine visibility — Morning confirmed working 2026-04-24.** 22:58 BST and morning windows both verified ✓. Fix uses `setAttribute("hidden","")` + `style.display="none"` together so old WebViews can't ignore either. A `<span id="dbg">` in the bottom-left corner shows `mins=NNN` live. **Still to verify: evening window (15:30–20:30).** Once evening is also confirmed, remove the `dbg` indicator (search `id="dbg"` in `index.html`).
- **Bus 102 countdown — likely root cause found 2026-04-24 (escapeHtml was undefined), awaiting Terry verification.** The sequential `for` loop refactor + diagnostic strip shipped earlier in the day. On top of that, **a `function escapeHtml()` helper was missing from the script entirely** — two call sites in `loadTflArrivals()` reference it (`escapeHtml(s.dest)` and `escapeHtml(upcoming[i].t)`), which would throw `ReferenceError` and abort the DOM update. That's a strong candidate for why the arrivals row has been silent in the kitchen despite correct logic. Helper is now defined at the top of the `<script>` block. **Walk past Terry:** if the 102 row now shows `Due · 4 min · 10 min · SCHEDULED`, root cause confirmed — then remove the `dbg` line (search `arr-dbg` in `index.html`). If still blank, fall back to the DevTools plan from the earlier entry.

## 🔧 Kiosk APK Rebuild Recipe (when MainActivity.kt or AndroidManifest.xml change)

Needs Mac Terminal.app (sandbox-blocked from Claude Code) — but **no cable required** thanks to WiFi ADB:
```
/Users/elul/Projects/apps/kiosk-webview/build.sh && \
adb -s 192.168.1.116:5555 install -r /Users/elul/Projects/apps/kiosk-webview/app/build/outputs/apk/debug/app-debug.apk && \
terry restart
```
Cable only required if WiFi adb port 5555 has closed (Terry was rebooted) — re-arm with `adb tcpip 5555` then unplug.

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
1a. **🚌 Live 102 bus arrivals — on probation.** Stops wired (Brookland Rise E + W: `490004463E` / `490004463W`). TfL `/StopPoint/{id}/Arrivals` returns `[]` for every 102 stop tested, and the legacy Countdown API says `"No Information"` for 102 at SMS code 47013 — so TfL's iBus prediction feed isn't covering this route right now (outage, permanent gap, or something in between). Schedule fallback shipped: next 3 scheduled departures with a `SCHEDULED` chip, rendered as **`4 min · 16 min · scheduled`** (<60 min) or `HH:MM` (later-evening) so it still has the glance-value of a live countdown. If live predictions ever come back, the existing `loadTflArrivals()` live path takes over automatically. **If they don't come back within a week of normal operation, remove the entire bus card** — line-status alone ("Good Service") isn't useful enough to justify the real estate. Keep the Northern Line status row regardless.
2. **📦 Delivery Tracker** — Show expected deliveries today (parse from Gmail via Google API).
3. ~~**🗓️ Family Countdown**~~ — **Dropped 2026-04-24.** Built then removed: maintaining a hardcoded date array isn't worth the friction when the calendar already has the same events. The merged iCal feed (Family + Elul) is the single source of truth. If we ever want a "next X days" countdown card, derive it from the calendar feed instead of a separate config.
4. **🏫 School / Activities** — Daily schedule for kids: *"Swimming 4pm, Piano 5:30pm"*.
5. **💬 Family Message Board** — Shared note area anyone can update from their phone (Google Sheet backend, no server needed).
6. **🌅 Daily Photo Memory** — "On this day"-style memory surface from Google Photos.
7. ~~**⭐ Stars Card (Table Stars mirror)**~~ **Done 2026-04-24.** Pulls `/api/public/stats` from `tablestars.erapps.xyz` (new keyed public endpoint: `STATS_READ_KEY` env var in the table-stars Vercel project, gated separately from NextAuth). Shows per-kid **prize_count** as `🎁 × N` in DM Serif Display (matching the clock + temp editorial face — slow-moving celebratory number) plus **current-cycle progress** as 10 pip stars + `X/10` ratio in JetBrains Mono. Kid accents match the rest of the dashboard: Eitan cobalt, Tamar terracotta. Prize count pulses on increment between polls. Hides during routine windows via `body.routine-active .stars { display: none }`. Polled every 30 min (stars change 1–2× per day). Explicit non-goals: no leaderboard, no raw lifetime star totals, no trend graphs, no editing from Terry — Table Stars stays the source of truth and the only write surface.

## 📊 Useful Glanceable Info

7. ~~**☀️ Hourly Weather**~~ **Done.** Strip under the weather card shows next ~18h (6 × 3-hourly forecast points). Rain hours (codes 09/10/11) coloured cobalt. Same OpenWeather key, free `/forecast` endpoint.
8. ~~**🗑️ Bin Day Reminder**~~ **Built then removed 2026-04-24.** Calendar already has the recurring "put bins out" event on Tue evening (pickup is Wed early morning). Banner was duplicative + had the wrong day baked in. Calendar wins.
9. **🛒 Shared Shopping List** — Google Sheet backend, anyone adds from their phone. *Needs: new Apps Script endpoint or extra tab in existing "Family Dashboard" sheet.*
11. **💷 Budget Tracker** — Monthly family spending from a Google Sheet (or Rifman Family Budget Firestore).

## 📅 Calendar — Multi-Source Merge

- ~~**Let the proxy merge multiple iCal feeds.**~~ **Done 2026-04-24.** Proxy now reads `ICAL_URLS` (comma-separated) + `ICAL_LABELS` (parallel labels: "Family,Elul"). Fetches in parallel, dedupes by `uid+start`, returns merged sorted list. Each event tagged with its `source` label. Dashboard side needed no changes — already reads `events` array.
- ~~**Color-code events by source.**~~ **Done 2026-04-24.** `sourceClass(ev)` slugs the source label into a CSS class (`source-family`, `source-elul`, etc.) applied to chips, timeline blocks, and future-day rows. Color map: Family=ochre, Elul=cobalt, School=sage, Lior=terracotta. Add more in the CSS by following the same 4-line pattern.
- **CLI gotcha (preview env):** `vercel env add NAME preview --sensitive --yes --value "..."` returns `git_branch_required` error. Fix: pass empty string as 3rd positional arg → `vercel env add NAME preview "" --sensitive --yes --value "..."`. The CLI's `next[]` hint suggests omitting the arg, but that doesn't work non-interactively when the var also exists in production.

## 📡 WiFi ADB to Terry (no cable needed)

**Terry's local IP:** `192.168.1.116` (Apr 2026 — recommend setting a DHCP reservation in your router so it stays this).

**Re-arm WiFi ADB after a Mac reboot or losing the connection:**
```
adb connect 192.168.1.116:5555
adb devices                                                # confirm "device" status
adb shell am start -n xyz.erapps.kiosk/.MainActivity       # launch dashboard
```

**One-time WiFi ADB setup (already done 2026-04-24):** with cable connected, ran `adb tcpip 5555` to put adb into TCP mode on Terry. Once done, port 5555 stays open until next Terry reboot — at which point you need the cable + `adb tcpip 5555` again.

**If `adb connect` fails** (e.g. after Terry reboot — port 5555 closes): plug USB cable in, run `adb tcpip 5555`, unplug, then `adb connect 192.168.1.116:5555` works again. To make this fully cable-free permanently, look into Android 11+ wireless ADB pairing (Developer Options → Wireless debugging → Pair device with pairing code) — Portal's Android version may or may not support it.

**Use the `terry` CLI (preferred — wraps all the adb commands):** Located at `~/bin/terry`. **Auto-discovers Terry by MAC address** (`a4:0e:2b:74:d4:85`) if the cached IP stops responding — no DHCP reservation needed (Deco's Smart DHCP locks reservations out anyway). On a cached-IP miss the script pings the local subnet to populate the ARP cache, then matches MAC → IP and saves the new endpoint to `~/.terry-ip`. Subcommands:
```
terry              # launch dashboard (default)
terry on           # launch dashboard
terry off          # stop dashboard, return to Portal
terry restart      # force-stop + relaunch
terry home         # switch to Portal (kiosk keeps running in background)
terry status       # show connection, default home, foreground app
terry reboot       # reboot Terry (~30s)
terry connect      # re-arm WiFi adb after Mac reboot
terry rediscover   # force MAC-based IP rediscovery (e.g. after IP change)
terry ip           # print cached IP
terry help         # show usage
```

**If you ever need raw adb directly** (with WiFi connected via `terry connect`):
```
adb -s 192.168.1.116:5555 shell am start -n xyz.erapps.kiosk/.MainActivity   # launch
adb -s 192.168.1.116:5555 shell am force-stop xyz.erapps.kiosk               # stop
adb -s 192.168.1.116:5555 reboot                                             # reboot Terry
```

## 🚪 Exit-to-Portal

- ~~**Iter 1–3 all failed** (HOME removed → kiosk invisible; explicit Aloha launch → snap-back; Settings.ACTION_HOME_SETTINGS → no-op or invisible).~~ Root cause: kiosk is the registered default Home (`xyz.erapps.kiosk` per `cmd package resolve-activity`), so any in-app exit triggers Android's home-resolution → routes back to us.
- ~~**Iter 4 (shipped, verified working 2026-04-24): activity-alias toggle.**~~ Manifest split: MainActivity is LAUNCHER-only; new `<activity-alias name=".HomeAlias">` carries the HOME category and is toggleable via `PackageManager.setComponentEnabledSetting`. `exitToPortalLauncher()` disables HomeAlias *before* launching Aloha and finishing — Android can't snap back because we're no longer a registered Home. `MainActivity.onCreate()` re-enables HomeAlias on every fresh launch (incl. BootReceiver path), so kiosk is restored as a Home choice automatically on next reboot. After install, user picked **Portal Launcher → Always** in the system home picker — Aloha is now permanent default home, exit transitions cleanly with no chooser, BootReceiver still auto-launches kiosk on reboot.
- **Visible button:** bold terracotta pill labeled "HOLD TO EXIT" at bottom-right, with circular ochre progress ring (2s).
- ~~**Iter 5 (shipped 2026-04-24): auto-return after N min.**~~ `exitToPortalLauncher()` now schedules an `AlarmManager.setExactAndAllowWhileIdle` `AUTO_RETURN_MS` ahead BEFORE disabling HomeAlias. New `AutoReturnReceiver.kt` BroadcastReceiver fires the alarm and relaunches MainActivity. **2026-05-10:** `AUTO_RETURN_MS` shortened from 5 min → **2 min**, paired with the new floating "← Dashboard" overlay (see Spotify Integration section). Receiver still call-aware (`AudioManager.MODE_IN_COMMUNICATION`/`MODE_IN_CALL` → reschedule). **Music is intentionally NOT a deferral signal** — kids can leave Spotify playing forever otherwise.
- **WiFi ADB available as a manual override:** `terry on` from any laptop, no waiting for the auto-return.
- **Floating overlay button (shipped 2026-05-10):** `OverlayService.kt` paints a "← Dashboard" pill via `WindowManager` + `TYPE_APPLICATION_OVERLAY` whenever the kiosk is in background. Started in `MainActivity.onStop`, stopped in `onStart`. Sits top-right at `y=68dp` to clear Portal's status bar (the bar draws above `TYPE_APPLICATION_OVERLAY`, so smaller offsets get covered).

## 🎨 Design Refinement

- ~~**Numbers display redesign (clock, weather temp).**~~ **Done 2026-04-24 (Pass 3).** Switched from JetBrains Mono Bold → **DM Serif Display** for big glanceable numbers. Editorial Bodoni-style face (think NYT magazine, New Yorker covers): tall narrow tabular figures with distinctive silhouettes — kids parse digits as shapes (flat-bottomed 2, tailed 5, sharp 7) instead of letterforms. New CSS variable `--display` defined; applied only to `.clock` (cream, 64–116px) and `.weather .temp` (ochre, 44–60px). Color hierarchy now meaningful: cream clock = time anchor, ochre temp = warm world-outside info. Each font now has ONE job — display=big numbers, Fraunces=prose, Public Sans=labels, Mono=tabular alignment.

- ~~**To-Do notebook lines don't align with task rows.**~~ **Done.** Replaced repeating-gradient with `border-bottom` on each `<li>`. Removed static terracotta margin line.
- ~~**Hebrew font in calendar events looks bad.**~~ **Done.** Added Heebo (wght 300–700) to Google Fonts; inserted before Frank Ruhl Libre in `--serif` and `--sans` stacks. Fraunces doesn't cover Hebrew so Hebrew chars fall through to Heebo automatically.
- ~~**"The Rifman Almanac · Kitchen Edition" brand line is too small and unnecessary.**~~ **Done.** Removed. Greeting is now slightly larger (clamp 30–44px).
- ~~**Calendar card huge and empty; routine card too narrow.**~~ **Done 2026-04-24.** Grid changed `1.65fr / 1fr` → `0.95fr / 1.1fr` (kid columns ~290px, was ~210). Calendar rebuilt as Google-Cal-style day view: 06:00–22:00 timeline with hour markers, all-day chips strip, terracotta "now" line, future days as compact list below. Eyebrow meta = TODAY.
- **Follow-up if routines still feel cramped during active windows:** swap left/right placements via a body class set by JS (e.g. `body.mode-morning` / `body.mode-evening` → routine takes the wider left slot, calendar moves right). Not built — see how the current resize feels first.
- ~~**Numbers in Fraunces are pretty but hard to read for young kids.**~~ **Done 2026-04-24.** Swapped clock + weather temp from Fraunces serif to JetBrains Mono 700 weight (already loaded). Mono digits are chunkier and easier for early readers. Tabular times (calendar / transit) were already mono.
- **Weather card is wide — open slot for a sibling.** After the right column got wider, the weather card spans the full width and feels stretched. We can split it into a 2-column row inside the right column: weather on the left (~60%), and a small companion card on the right (~40%). Good candidates for the companion: TfL transit (always-visible instead of routine-conditional), Family Countdowns, Bin schedule (multi-bin), Spotify Now Playing, Daily Photo, or a "next event in N min" callout. Easy implementation: wrap weather + companion in a `<div class="wx-row">` that's `display: grid; grid-template-columns: 1.5fr 1fr` inside `.right`.

## ✨ UX Polish

- ~~**Celebration animation when a kid completes their morning routine.**~~ **Done.** Toast "🎉 You did it!" + kid's name in their accent colour, 44 confetti emoji falling, kid's name pulses for 1.3s. Once-per-day per kid via `localStorage` key `celebrated:YYYY-MM-DD:{kid}`. Morning only.
- ~~**To-Dos fade out 15s after being checked.**~~ **Done.** Tick → 15s grace → CSS `.fading` (opacity + slide-right 350ms) → API delete. Un-tick within window cancels timer. Items already-done from a phone (caught on the 60s poll) get the same 15s grace.

## 🎮 Fun / Vibe

12. ~~**🎵 Spotify integration**~~ **Done 2026-05-10.** Full now-playing + control + launcher + audio-on-Terry. See "Spotify Integration" section below for architecture, login gotchas, and the version pin (8.6.x ONLY — newer versions break login on this degoogled Portal).
13. ~~**🌍 World Clock**~~ **Done.** Tel Aviv time line under the masthead date (🇮🇱 + city + time, mono). Add more cities by extending the `tick()` worldclock block.
14. **📸 Photo Slideshow Background** — Rotate family photos behind the dashboard cards. *Needs: photo source (Drive folder? Google Photos OAuth?).*
15. **🐾 Chore Wheel** — Random daily chore assignments per family member, resets each morning. *Needs: chore list.*
16. ~~**💡 Daily Quote / Mantra**~~ **Done.** 36 rotating quotes, picked by day-of-year, fixed at bottom-center of screen in subtle italic. Edit the `QUOTES` array in `index.html` to add more.

---

## 📚 Guides for Elul (work that needs your hands, not Pi's)

### ~~Spotify "Now Playing" — full setup~~ **SHIPPED 2026-05-10. See "Spotify Integration" section below for the as-built architecture.**

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

---

## 🎵 Spotify Integration (shipped 2026-05-10)

**Architecture (3 surfaces, all working):**

1. **Dashboard now-playing display** — proxy endpoint `/api/spotify-now` in `family-dashboard-proxy` returns currently-playing JSON for the account whose refresh token is in `SPOTIFY_REFRESH_TOKEN`. Dashboard polls every 10s; proxy caches 8s. Returns `{isPlaying:false}` when nothing playing or on token refresh failure.
2. **Full player card** — lives inside `.primary-panel` as a `body[data-mode="spotify"]` view. **Auto-opens** on the false→true playback transition (music just started) **but ONLY in `todo` mode** — kids' morning/evening checklists win. Auto-closes after 90s back into time-of-day mode. ⏮ ⏸ ⏭ buttons fire `kiosk://spotify/{cmd}` URIs the kiosk WebView turns into Android broadcast intents to Spotify on Terry.
3. **Green ♫ corner button** — bottom-left mirror of the exit button. Tap when music playing → opens dashboard's full player card (re-entry after auto-close). Tap when nothing playing → fires `kiosk://spotify/launch` → Spotify APK on Terry.

**Vercel env vars (in `family-dashboard-proxy`, all marked sensitive):**
- `SPOTIFY_CLIENT_ID` — Spotify dev app
- `SPOTIFY_CLIENT_SECRET` — Spotify dev app
- `SPOTIFY_REFRESH_TOKEN` — long-lived; refreshed via authorization_code grant (one-time)

**Hard pin: Spotify-on-Terry must be 8.6.x (specifically 8.6.98.900 universal nodpi from APKMirror).** Spotify 9.x forces browser-based auth via Custom Tabs; Terry is a degoogled Portal with no Google Play Services and only bare `org.chromium.chrome`, so the post-captcha `spotify://` deep-link back to the app never fires → infinite login loop. **Don't "upgrade" Spotify on Terry.** Tolerate the "outdated app" nags.

**Spotify Connect cross-account:** by design Spotify Connect only lets devices on the same account target each other. Wife (separate account) can't see Terry. Workarounds (in order of friction):
- (a) Share login: she logs into Terry's Spotify account on her phone too — Connect just works.
- (b) Spotify Jam (Group Session): Premium-host starts Jam targeting Terry, sends QR/link, guest joins. Link is ephemeral — re-issued each session.
- (c) Bluetooth-pair her phone to Terry — bypasses Spotify on Terry entirely. Strip won't show her music.

**Files touched (all in `index.html` + the kiosk + proxy):**
- `family-dashboard/index.html` — full player CSS/HTML, green button, JS render + auto-open logic
- `family-dashboard-proxy/api/spotify-now.js` — refresh-token-backed currently-playing endpoint
- `kiosk-webview/app/src/main/java/xyz/erapps/kiosk/MainActivity.kt` — `kiosk://spotify/{launch,play,pause,prev,next}` URI handler + OverlayService start/stop hooks + `AUTO_RETURN_MS=2min`
- `kiosk-webview/.../AutoReturnReceiver.kt` — `RESCHEDULE_MS=2min`, call-aware deferral
- `kiosk-webview/.../OverlayService.kt` — floating "← Dashboard" pill (top-right, y=68dp clears Portal status bar)
- `kiosk-webview/AndroidManifest.xml` — `<service>` registration for OverlayService

**Refresh-token recovery if the env var is ever lost:** redo the OAuth dance once. Steps:
1. Open `https://accounts.spotify.com/authorize?client_id=<CLIENT_ID>&response_type=code&redirect_uri=https%3A%2F%2Ffamily-dashboard-proxy.vercel.app%2Fapi%2Fspotify-callback&scope=user-read-currently-playing+user-read-playback-state`. Approve.
2. Spotify redirects to a 404 (callback endpoint doesn't exist — that's fine). Copy the `code=` value from the URL.
3. `curl -X POST https://accounts.spotify.com/api/token -u "$ID:$SECRET" -d grant_type=authorization_code --data-urlencode "code=$CODE" --data-urlencode "redirect_uri=https://family-dashboard-proxy.vercel.app/api/spotify-callback"` → grab `refresh_token`.
4. `printf '%s' "$REFRESH" | vercel env rm SPOTIFY_REFRESH_TOKEN production --yes; printf '%s' "$REFRESH" | vercel env add SPOTIFY_REFRESH_TOKEN production --sensitive` then `vercel --prod`.

**Kiosk APK install for Terry trust:** first adb command after a long gap may show `unauthorized` — Terry's screen has a "Allow USB debugging from this computer?" prompt, tap "Always allow from this computer".

---

## ✅ Open Tests / Follow-ups

### 2026-06-15 — BUG (dashboard): duplicate Spotify launch buttons, left one mis-behaves

Elul observed (2026-06-15) that the dashboard sometimes shows **two Spotify buttons**, and tapping the **bottom-left** one just makes it **disappear** instead of launching Spotify — and the behavior differs when the kids' to-do / routine card is showing. Likely overlap between `#spotify-corner` (bottom-left, routine-window-gated) and the in-card "Open Spotify" launcher, with one tap path clearing the music-override / hiding itself rather than firing `kiosk://spotify/launch`. Dashboard-side only (`index.html`), not kiosk.
- [ ] Reproduce: note exact mode (morning/evening routine vs default spotify) + which button is the "left" one.
- [ ] Fix so every visible Spotify launcher reliably launches Spotify; no state where a launcher just dismisses itself.

### 2026-06-12 — Spotify auto-return is now music-gated (kiosk-side, NOT yet applied)

Elul reversed the old "music is not a deferral signal" rule after weeks of use — the flat 2-min yank out of Spotify was annoying. AGENTS.md is updated to the new intent. **The code change lives in `kiosk-webview/MainActivity.kt` + `AutoReturnReceiver` on the Mac — it is NOT in this repo and is NOT live on Terry until applied + rebuilt** (see "🔧 Kiosk APK Rebuild Recipe" above).

- [x] Apply the change: on Spotify, no auto-return while `AudioManager.isMusicActive`; poll every `MUSIC_POLL_MS` (30s); return only after `MUSIC_GRACE_MS` (2 min) of continuous silence. `on_spotify` / `music_stopped_since` flags in `SharedPreferences("kiosk_prefs")`, cleared in `onStart`. Non-Spotify backgrounding keeps flat `AUTO_RETURN_MS` (2 min). **Applied + built + installed on Terry 2026-06-15.**
- [x] Rebuild + reinstall APK on Terry — done 2026-06-15 (built 11:44, installed over WiFi-ADB `.110:5555`).
- Verified on Terry 2026-06-15 (via `dumpsys` + founder visual confirm):
  - [x] Music playing → never auto-returns (stayed on Spotify 2.5+ min, foreground never flipped to kiosk).
  - [x] Manual "← Dashboard" pill still returns instantly.
  - [x] Stop music → returns ~2 min later (silence clock started at first poll, returned, prefs cleared).
  - [ ] Brief pause / track gap (<2 min) does NOT return — NOT yet tested on-device (logic-guaranteed: receiver resets `music_stopped_since=0` whenever `isMusicActive`).
  - [ ] In-call deferral still works — NOT re-tested (code path unchanged; needs a real call to verify).
- [x] **Screensaver side-effect fixed (2026-06-15).** Keeping the kiosk on Spotify let Portal's screensaver appear after ~1 min / screen blank at 5 min. `OverlayService` now toggles `FLAG_KEEP_SCREEN_ON` on its overlay window while `on_spotify && isMusicActive`. Verified: screen stayed ON with no dream for 2.5+ min of playback; lock released ~15-22s after pause. See AGENTS.md kiosk section.

### 2026-05-17 reliability + polish ship (Phase 0–2 of 5.5→9.5 plan)

**Dashboard side (already deployed via `main` push, GitHub Pages auto):**
- [ ] Pause/play icon now flips within 1s of tap (cache-bust + skip-grace working)
- [ ] `#spotify-corner` visible during default daytime (no longer hidden outside morning/evening)
- [ ] "Open Spotify" pill visible in playing state, ⤴ glyph in top-right of album art
- [ ] Long-press album art (700ms) launches Spotify APK with haptic feedback
- [ ] Track-name marquee animates only on long titles, short titles stay still
- [ ] Stale badge ("— connection — retrying") appears under controls if proxy is silent ≥60s
- [ ] Typography scale-up legible at kitchen distance (~1.5–2m); routine cards still fit without scroll
- [ ] Phase 0 console.warn `[sp] poll {...}` visible via `chrome://inspect` on Mac

**Kiosk APK (requires manual rebuild from Mac Terminal.app — sandbox blocks Gradle):**
```
/Users/elul/Projects/apps/kiosk-webview/build.sh && \
adb -s 192.168.1.116:5555 install -r /Users/elul/Projects/apps/kiosk-webview/app/build/outputs/apk/debug/app-debug.apk && \
terry restart
```
- [ ] After install + Terry restart, you'll see a Toast "Enable Notification access for Kiosk → reliable Spotify control" and be dropped into Settings → Notification access
- [ ] Tap "Allow" on the Kiosk entry. After that, `adb -s 192.168.1.116:5555 logcat -s TerryKiosk` will show `notif=true` on next launch
- [ ] Verify reliable media commands: tap ⏸ on dashboard → logcat shows `media cmd=pause beforeState=3 listener=true` → after 600ms either `changed=true` OR a `fallback=transportControls` line
- [ ] If listener grant is denied, kiosk still works — logcat shows `listener=false` and no verify lines; bare media key still dispatched

### Out-of-scope (path to 10/10, separate plan)

Documented in `docs/plans/2026-05-17-reliability-and-polish-9-5.md`:
- Spotify Web Playback SDK in the dashboard (kills polling lag entirely, removes 8.6.x APK pin)
- Wake-word voice via Picovoice Porcupine ("Terry, pause")
- Phone widget firing `kiosk://spotify/*` over Tailscale → Pi → ADB to Terry
- LMK exemption for Spotify (root tweak, Portal is already hacked)

### Pre-2026-05-17 follow-ups (still open)

1. **Routine-window auto-open suppression.** Music starting at 07:30 (weekday morning) or 18:00 (evening) should NOT auto-pop the player card. Only triggers in default `spotify` mode.
2. **Overlay button position post-2026-05-10 size bump** (`y=68dp`, text 15sp, padding 18×11dp). Re-confirm clear of Portal status bar AND tappable.

## 🎨 Design Refinement Pass (queued 2026-05-10)

User feedback at session close: *"the screens are a bit small and hard to look at a glance (except the checklists for the kids which are generally fine size). I think we need an overall refinement."* Kitchen viewing distance ~1.5–2m. Likely scope: masthead clock, context-strip labels (weather desc, transit names, stars X/10), calendar event titles, eyebrow labels. Spotify full player card may also need tonal alignment with the editorial palette (currently a bit Spotify-branded green). **Handled in a separate session via a dedicated brief — see chat history of 2026-05-10 for the prompt.**
