# Family Dashboard — Reliability + Polish, 5.5 → 9.5 (→ 10 path)

**Date:** 2026-05-17 (revised after neutral review)
**Goal:** Make Terry the device the family can trust. 100% reliability on play/pause/skip/launch. 5× polish on UX/UI. Honest path to a true 10/10 mapped at the end.
**Today's score:** 5.5 / 10. **Target:** 9.5 / 10 in this plan; 10 in a follow-on.
**Time estimate:** ~8h split across 2–3 sessions (Mac + Pi + ~30 min at Terry).

---

## What's actually broken (root causes from code reading + neutral review)

| # | Bug user reports | Root cause | Confidence |
|---|---|---|---|
| 1a | "Can't stop songs" — pause looks broken | **Proxy cache lag (15s while playing).** `spCommand` re-polls at 800ms/2200ms (index.html:2921–2922) but proxy edge-cache returns the *pre-pause* `isPlaying:true` until 15s expires. Icon doesn't flip → kid taps again → fires `pause → play`. They're fighting the cache, not the kiosk. | High |
| 1b | Pause actually fails sometimes | `dispatchMediaKey()` in `MainActivity.kt:264` is fire-and-forget. If Spotify lost media-session focus (Portal video call ended, notification ding, app reaped) the keypress disappears. No verify, no retry. | High |
| 1c | Pause card stays in playing-UI for 30s | `setMusicOverride` grace timer (index.html:2814 + 2848–2853) gives a 30s window before clearing override. When user *explicitly* pauses, grace shouldn't apply. | High |
| 2 | "Buttons mostly don't work" | **Misdiagnosed in v1.** Routine items (which the user says work) have no `touchend` handler at all — only Spotify card buttons + corner + exit pill do. So Bug 2 is Bug 1 surfacing as Bug 2 — the family is reporting Spotify control failures, not a systemic touch problem. Fix: de-double **only** the Spotify-card handlers; leave routines alone. | High (after review) |
| 3 | "Launch Spotify is gone" | `index.html:882` literally `#spotify-corner { display: none }` outside morning/evening. In default `spotify` mode the corner is hidden AND in-card launcher only renders in `data-state="idle"`. Music playing in daytime = zero launchers anywhere. **One-line CSS fix.** | High (CSS-proven) |
| 4 | "Spotify randomly stops" | 3 plausible: (a) Android LMK reaping Spotify, (b) audio focus stolen by Portal system sounds, (c) the 6-hour `<meta http-equiv="refresh">` wiping JS state mid-track → card briefly idle until 15s re-poll. (c) is the one we missed in v1. Needs telemetry. | Medium (needs data) |

---

## Plan (4 phases — restructured from v1)

### Phase 0 — Two-line surgical test (30 min, ship immediately)

Before any instrumentation: prove the diagnosis with the smallest possible change. If these don't move the device from 5.5 → 7+ in real use, the diagnoses are wrong and a longer instrumentation pass is justified.

1. **Delete `#spotify-corner { display: none; }`** at `index.html:882` (the rule, not the element). Corner Spotify launcher now appears in `spotify` mode too. **Bug 3 done.**
2. **Cache-bust the post-command re-polls.** In `spCommand` (`index.html:2917–2923`), add `?t=${Date.now()}` to both `loadSpotify` calls so they bypass the 15s proxy edge cache. The icon flips within 800ms of tap. **Bug 1a substantially fixed.**
3. **Skip-grace on explicit pause.** In `spCommand`, when `cmd === "pause"`, call `setMusicOverride(false)` immediately. **Bug 1c done.**
4. Commit, push, GitHub Pages auto-deploys. Test at Terry with `?v=$(date +%s)` cache-buster.

**Exit criteria:** family uses the dashboard for an evening of music control. If pause "just works" most of the time and the Spotify-app button is reachable, proceed to Phase 1. If not, the v1 plan's deeper instrumentation (logcat tags + tap log + Spotify state log) becomes Phase 0.5 before continuing.

### Phase 1 — Reliability hardening (3–4h)

Same goal as v1 Phase 1, narrower scope after the review.

#### 1a. Reliable media commands (Bug 1b) — kiosk side
- Wrap `dispatchMediaKey()` in verify-and-fallback via `MediaSessionManager.getActiveSessions()`:
  ```kotlin
  fun reliableMediaCmd(cmd: String) {
    val before = currentSpotifyPlaybackState()
    dispatchMediaKey(keycodeFor(cmd))
    handler.postDelayed({
      if (currentSpotifyPlaybackState() == before) {
        spotifyMediaController()?.transportControls?.let { tc ->
          when (cmd) { "pause" -> tc.pause(); "play" -> tc.play();
                       "next" -> tc.skipToNext(); "prev" -> tc.skipToPrevious() }
        } ?: startActivity(packageManager.getLaunchIntentForPackage("com.spotify.music"))
      }
    }, 600)
  }
  ```
- Requires `BIND_NOTIFICATION_LISTENER_SERVICE` grant.
- **Graceful degradation:** if grant denied, skip the verify, keep simple `dispatchMediaKey`. Don't block ship on permission UX. Phase 0 cache-bust already removes the visible failure mode for most cases.

#### 1b. De-double Spotify-card handlers (narrow Bug 2 fix)
- Drop `e.preventDefault()` + `touchend` handlers from lines 2933, 2951, 2955, 2959, 2969, 2988, 3033 — keep only `click`.
- Add CSS `touch-action: manipulation; -webkit-tap-highlight-color: transparent` to `.sp-prev, .sp-next, .sp-play-pause, #spotify-corner, #sp-launch-btn, #routine-back, #exit-button`. (Line 906 already has it for `#spotify-corner` — extend the rule.)
- **Do NOT touch routine items.** They work; no PointerEvent refactor.
- **Do NOT introduce `bindTap` abstraction.** No new patterns until on-device test proves single-handler `click` is reliable on Portal Chromium.

#### 1c. Always-on Spotify launcher polish (continuation of Phase 0 #1)
- When playing, add a slim "Open Spotify" pill below the play controls in the card (not just corner). Long-press album art (700ms) as backup.
- Tiny ⤴ glyph in the top-right of the album art so the affordance is visible.

#### 1d. Bug 4 instrumentation (logcat-only, lightweight)
- Add `Log.i("TerryKiosk", ...)` on every URI received + every media-key dispatch.
- One JS log line per Spotify poll: `console.warn("[sp] poll", {isPlaying, ts})`. View via Chrome DevTools remote inspection (`chrome://inspect` from Mac when ADB connected).
- **One specific test to run:** trigger a manual reload at Terry while music plays; watch whether the card flashes idle. If yes, the 6-hour `<meta refresh>` is the culprit — replace with a soft-reset that preserves Spotify JS state (re-fetch but don't reload).
- Audio-focus hardening: in `MainActivity`, do NOT request audio focus (we don't play sound). Verify the WebView isn't requesting it either via `<media-session>` API.
- No foreground service (cut from v1 — Portal SDK 29 would show a persistent notification; verify-and-fallback covers the LMK case anyway).

### Phase 2 — UX/UI polish pass (2–3h)

Anchored to the 2026-05-10 brief ("screens small, hard to glance"). 1.5–2m kitchen viewing distance.

1. **Typography scale-up, scoped:**
   - Context strip labels (weather desc, transit names, stars X/10): 14px → 17px
   - Calendar event titles: 18px → 22px
   - Eyebrow labels (TODAY, NEXT BUS, NOW PLAYING): 11px → 13px, +letter-spacing
   - Spotify card track title: full editorial weight DM Serif Display
   - Masthead clock: +6%
   - **Routine items: untouched.** They're at 17px today and the no-scroll hard constraint is non-negotiable.
2. **Spotify card de-Spotifyed:** drop the brand-green dominant button in favor of editorial cream/ochre on aubergine. Keep one small green Spotify glyph in the corner for recognition. Matches dashboard palette.
3. **Visible tap feedback (across the kitchen):**
   - 200ms scale-down (0.97) + 120ms ochre flash on `:active` for `.sp-prev, .sp-next, .sp-play-pause, #spotify-corner`.
   - `navigator.vibrate(20)` on tap (Portal WebView supports it).
   - Spinner (or pulsing icon) on play/pause between tap and confirmed state-change.
4. **Calendar density:** agenda block vertical padding 12 → 8px. 1–2 more events visible.
5. **Track-name marquee:** when track exceeds container, slow left-scroll CSS keyframe (one cycle / 12s). Better than ellipsis-truncation.
6. **Visible error state:** if proxy 5xx or no reply in 60s, Spotify card shows a small "— connection — retrying" line instead of stuck-playing UI.

### Phase 3 — Verification on Terry (45 min, with user at Terry)

Hard gate before ship-to-main. 12 items, all must pass.

1. Tap exit pill → Portal launcher within 1s
2. After 2 min idle, kiosk auto-returns
3. Default daytime mode: corner Spotify button visible AND tappable → Spotify APK opens
4. Music playing, tap ⏸ → pause within 1s, icon flips within 1s (cache-bust test)
5. Tap ⏸ again → resumes within 1s
6. Tap ⏭ → next track in 2s, dashboard updates in 5s
7. Tap ⏮ → prev track
8. Long-press album art → Spotify APK opens
9. Pause music for 5 min → no auto-restart loop
10. Stress test: start music, end Portal video call, immediately tap ⏸ → fallback path fires (logcat shows MediaController path), pause works
11. 10 consecutive routine-item taps, 0 misses
12. Morning + evening routine card auto-shows in window, Spotify card returns out of window

Any fail → fix → re-run all 12. No partial passes.

### Phase 4 — Strip + ship (30 min)

- Remove the JS debug `console.warn`s (keep kiosk logcat tags — free + useful for next regression).
- Update `AGENTS.md`:
  - "Spotify control: cache-bust on user-initiated polls (proxy edges-cache 15s while playing). Don't remove `?t=${Date.now()}` from `spCommand`."
  - "Spotify card buttons: single `click` handler only. Adding `touchend` + `preventDefault` masks the `click` on Portal Chromium."
  - "Routine items use plain `click`. Do not migrate to PointerEvent without on-device verification — Portal Chromium support is uneven."
- Commit + push. Rebuild kiosk APK from Mac Terminal: `~/Projects/apps/kiosk-webview/build.sh && terry restart`.

---

## Risks (revised)

| Risk | Mitigation |
|---|---|
| MediaSessionManager Notification Listener grant is scary UX | Phase 1a degrades gracefully if denied; Phase 0 cache-bust covers most cases anyway |
| Removing `touchend`+`preventDefault` introduces 350ms tap delay on old Portal Chromium | `touch-action: manipulation` CSS eliminates the delay without needing the handler |
| +15% type scale pushes routine items into scroll (violates hard constraint) | Type scale is scoped — routines untouched. Verify on 1280×800 viewport in Phase 3 before push |
| GitHub Pages cache lag (5–15min) makes Phase 3 frustrating | `?v=$(date +%s)` cache-buster during testing |
| `<meta refresh>` may be Bug 4's actual cause | Phase 1d explicitly tests this; replace with state-preserving soft-reset if confirmed |
| MediaSessionManager API surface changes between SDK 29 and modern Spotify | Verify on Terry's exact Android version before Phase 1a code; fallback path (restart Spotify) is the safety net |
| Phase 0 fix-and-test loop tempts skipping the verification gate | Phase 3 is non-negotiable. No exceptions. |

---

## The honest path to 10/10 (revised — v1 was rationalizing)

The v1 plan called 10/10 out of scope because of the Spotify 8.6.x version pin and single-account Connect. The neutral review pushed back: those are real ceilings but **not the actual gap**. The actual gap to 10 is **latency + control surface**:

1. **Spotify Web Playback SDK in the dashboard itself.** Dashboard becomes a Connect device. State changes pushed via `player_state_changed` event — zero polling, zero proxy lag, sub-second UI updates. Bonus: removes the 8.6.x APK pin entirely; Spotify-on-Terry becomes optional. Requires Premium (we have it) + one-time OAuth per family member. **Estimated effort: 1 weekend.**
2. **Wake-word voice control via Picovoice Porcupine** (free for personal use, on-device, no cloud). Runs as a small Android service alongside the kiosk, or as WASM in the WebView. "Terry, pause" → fires `kiosk://spotify/pause`. Removes the entire touch failure mode for the dominant kitchen use case (hands wet from dishes). **Estimated effort: 1 day.**
3. **Phone widget — pause from anywhere in the house.** Tiny Tailscale-served endpoint on the Pi → ADB-pushed intent to Terry. iOS Shortcut + Android widget = both phones. **Estimated effort: 2 hours.**
4. **Low-memory-killer exemption for Spotify via root.** Portal is already hacked; add Spotify to `oom_adj` whitelist. Kills Bug 4 root cause if (a) LMK is the culprit. **Estimated effort: 1 hour with cable access.**

What's NOT the path to 10: typography pass, marquee, ochre flash. Those move 5.5 → 7. Phase 1 reliability moves 7 → 9. **Phase 2 polish moves 9 → 9.5. The four items above move 9.5 → 10.**

If 9.5 is enough this month, ship this plan. If 10 is the real ask, this plan is the foundation (you need reliable kiosk control + a less-broken UX before layering the Web Playback SDK on top); the 10/10 work is a follow-on session, ~3 days total.

---

## Suggested ship order (revised)

Each is independently shippable. Stop after step 2 and you're at ~7.5. Stop after step 4 and you're at ~9.

1. **Phase 0** (30 min, today) — 3 surgical fixes, push, dogfood that evening
2. **Phase 1a + 1b + 1c** (3h, next session) — reliable media + de-doubled handlers + always-on launcher
3. **Phase 1d** (1h) — Bug 4 instrumentation, run for 24h, targeted fix
4. **Phase 2** (2–3h) — polish pass
5. **Phase 3** (45 min, at Terry, with you)
6. **Phase 4** (30 min) — strip + ship + AGENTS.md updates
7. **(Follow-on plan)** — the 10/10 quartet: Web Playback SDK + Porcupine + phone widget + LMK exemption
