# Family Dashboard — Roadmap / Future Ideas

A backlog of ideas for the Portal ("Terry") dashboard. Pick & build whenever.

---

## 🐞 Open Bugs

- **Morning + Evening routine cards both show outside their windows on Portal.** CSS `[hidden] { display: none !important }` rule was added but didn't help — both still visible at 23:00 BST after `pm clear`. Likely cause: Portal's old WebView ignoring `hidden` attribute differently, or `nowMinutesLondon()` returning wrong value on-device. Next debug step: add a tiny on-screen `<span id="dbg">` that prints the computed `mins` value so we can see what the Portal actually evaluates. Or wrap visibility logic so failure → hide both (current code may be defaulting to "show"). Parked April 2026.

## 🔧 Pending Infra

0. **Rebuild + reinstall kiosk APK with `LOAD_NO_CACHE`.** Source already updated in `~/Projects/apps/kiosk-webview/app/src/main/java/xyz/erapps/kiosk/MainActivity.kt`. Run from Terminal:
   ```
   /Users/elul/Projects/apps/kiosk-webview/build.sh && \
   adb install -r /Users/elul/Projects/apps/kiosk-webview/app/build/outputs/apk/debug/app-debug.apk && \
   adb shell am force-stop xyz.erapps.kiosk && \
   adb shell am start -n xyz.erapps.kiosk/.MainActivity
   ```
   After this, every dashboard push propagates within the 5-min meta-refresh — no more `pm clear` needed.

---

## 🏠 Smart Home Hub

1. **🚇 Live Transport** — TfL API (free) showing next trains/tubes from your nearest station. *"Northern Line: 2 min, 5 min, 8 min"*
2. **📦 Delivery Tracker** — Show expected deliveries today (parse from Gmail via Google API).
3. **🗓️ Family Countdown** — Big visual countdowns: *"Holiday in 23 days 🏖️"*, *"Mum's birthday in 5 days 🎂"*.
4. **🏫 School / Activities** — Daily schedule for kids: *"Swimming 4pm, Piano 5:30pm"*.
5. **💬 Family Message Board** — Shared note area anyone can update from their phone (Google Sheet backend, no server needed).
6. **🌅 Daily Photo Memory** — "On this day"-style memory surface from Google Photos.

## 📊 Useful Glanceable Info

7. **☀️ Hourly Weather** — *"Will it rain at 3pm?"* with a visual timeline bar.
8. **🗑️ Bin Day Reminder** — Color/alert changes the night before collection (big UK win).
9. **🛒 Shared Shopping List** — Google Sheet backend, anyone adds from their phone.
10. **📰 News Headlines** — BBC RSS, scrolling ticker at the bottom.
11. **💷 Budget Tracker** — Monthly family spending from a Google Sheet (or Rifman Family Budget Firestore).

## 🚪 Exit-to-Portal

- **"Back to Portal" button to exit kiosk mode for video calls.** Add a small unobtrusive button (corner of screen?) that drops out of the kiosk WebView app back to Terry's native launcher so the family can use Portal's video-call/photo-frame features. Two paths to investigate:
  1. **Built-in Portal escape:** check whether Portal has a native gesture (long-press home button? swipe from edge? specific button combo?) that already exits a foregrounded app. If yes, no app changes needed — just document it for the family.
  2. **Custom in-app button:** if no native escape exists, add a button in the dashboard HTML that fires a custom URL scheme (e.g. `kiosk://exit`) the kiosk app intercepts via WebViewClient, then `finishAndRemoveTask()` to return to launcher. Or add a long-press gesture on a corner so kids can't trigger it accidentally.
- Caveat: the kiosk app's `BootReceiver` will re-launch on next boot, so this is "exit for now," not "uninstall."

## ✨ UX Polish (queued)

- **Celebration animation when a kid completes their morning routine.** Trigger when all items in the kid's morning column are checked: confetti burst (or stars/balloons), short "🎉 You did it!" message overlay, and the kid's name pulses in their accent color. Fires once per day per kid (track in localStorage so it doesn't re-trigger after a refresh). Skip evening for now — could later trigger on a milestone like PJ if we want.
- **To-Dos fade out 15s after being checked.** When a to-do is ticked, leave it visible for 15 seconds (so you can un-tick if it was a misclick), then animate fade + slide out and delete from the Sheet. Replaces current "sink to bottom forever" behavior. Cleans the list automatically — list never bloats. Cancel the timer if user un-ticks within the 15s window.

## 🎮 Fun / Vibe

12. **🎵 Spotify "Now Playing"** — What's playing on the home speaker (free Spotify API).
13. **🌍 World Clock** — *"Tel Aviv 11:45pm 🇮🇱 | London 9:45pm 🇬🇧"* for family abroad.
14. **📸 Photo Slideshow Background** — Rotate family photos behind the dashboard cards.
15. **🐾 Chore Wheel** — Random daily chore assignments per family member, resets each morning.
16. **💡 Daily Quote / Mantra** — Rotating inspirational lines.

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
