# Spotify Integration — Options & Recommendation

**Project:** The Rifman Almanac (family-dashboard)
**Device:** Meta Portal "Terry" — 1280×800, Android WebView (aging Chromium)
**Date:** May 2026

---

## Context & Constraints

Before evaluating options, three hard constraints shape every decision:

**1. Spotify API (February 2026 changes).** As of March 9 2026, Development Mode apps require a Spotify Premium account and are capped at 5 authorised users. Critically, the **Player endpoints** (`GET /me/player/currently-playing`, `GET /me/player`, playback control) are **still available** in Development Mode — they were not in the list of removed endpoints. The `user-read-currently-playing` and `user-read-playback-state` scopes remain valid for personal projects. Playback *streaming* via the Web Playback SDK requires Premium; read-only "now playing" polling does not stream and has no Premium gate on the API call itself (only on the underlying Spotify account playing music).

**2. Portal's aging WebView.** Terry runs an old Chromium-based WebView. It has known quirks (`:has()` selector unreliable, `element.hidden` unreliable, no `Intl.DateTimeFormat#formatToParts`). Audio autoplay in WebView is blocked by Android's autoplay policy — a user gesture is required to start audio. This is a critical constraint for any option that tries to *play* audio through the dashboard itself.

**3. Screen real estate.** The dashboard is 1280×800 with a fixed two-column layout. The left column holds the calendar (~0.55fr). The right column (~1.45fr) has a masthead, a context strip (weather + transit + stars), and a primary panel (~500px tall) that cycles between morning routine, evening routine, and to-do. Any Spotify addition must not disrupt the routine cards — the kids depend on them.

---

## The Four Options

### Option A — "Now Playing" Ambient Card in the Context Strip *(ROADMAP original)*

**What it is:** A small, always-visible read-only display in the context strip showing the current track, artist, and album art. Polls the Vercel proxy every 30 seconds. Hides automatically when nothing is playing.

**How it works technically:**
- Add a `GET /api/spotify-now` endpoint to `family-dashboard-proxy` (Vercel serverless). It uses a stored refresh token to obtain a fresh access token and calls `GET /me/player/currently-playing`. Returns `{ track, artist, albumArt, isPlaying }`.
- In `index.html`, add a compact `.cs-spotify` block to the context strip (alongside weather, transit, stars). Poll every 30s. Hide via CSS when `!isPlaying`.
- The proxy handles token refresh server-side — the refresh token never touches the browser.

**Pros:**
- Exactly fits the existing context strip pattern (weather, transit, stars are all read-only ambient data).
- Zero audio permissions needed — no autoplay issues on the WebView.
- No new screen, no navigation complexity — the dashboard stays a single glanceable surface.
- The ROADMAP already has a complete step-by-step guide for this exact implementation.
- Minimal code: ~60 lines in the proxy, ~30 lines in the dashboard.
- Works whether music is playing from a phone, a speaker, or any Spotify Connect device on the account.

**Cons:**
- Read-only — you can see what's playing but cannot control it from Terry.
- Requires a one-time OAuth dance from a browser to capture the refresh token.

**Effort:** ~1–2 hours end-to-end (proxy endpoint + dashboard card + one-time token setup).

---

### Option B — "Now Playing + Controls" Card in the Primary Panel

**What it is:** A fourth mode for the primary panel (alongside morning routine, evening routine, to-do) — a Spotify player card with album art, track/artist name, and playback controls (play/pause, skip forward, skip back). Activated by a tap on a Spotify button in the context strip or masthead.

**How it works technically:**
- Extends Option A's proxy endpoint with additional endpoints: `POST /api/spotify-pause`, `POST /api/spotify-play`, `POST /api/spotify-next`, `POST /api/spotify-prev`. These call the Spotify Connect Web API (`PUT /me/player/play`, `POST /me/player/next`, etc.) — all available in Development Mode.
- Adds `body[data-mode="spotify"]` to the existing mode-switching system. A tap on a Spotify glyph in the context strip sets this mode; tapping elsewhere or after a timeout returns to `todo` mode.
- The primary panel shows a large album art card with controls when in spotify mode.

**Pros:**
- Full remote control of whatever device is playing (phone, speaker, etc.) — no audio through the WebView.
- Fits naturally into the existing `body[data-mode]` architecture.
- Still no audio streaming through the browser — avoids the autoplay problem entirely.
- Album art at large size looks visually striking in the primary panel.

**Cons:**
- More proxy endpoints to build and maintain.
- Adds a new navigation mode — need to ensure routine windows still take priority and the Spotify mode does not accidentally block a child from accessing their routine card.
- Slightly more complex state management (auto-timeout back to `todo` mode is important).

**Effort:** ~3–4 hours (extends Option A + control endpoints + mode integration).

---

### Option C — Spotify iFrame Embed (Inline Player)

**What it is:** Embed Spotify's official `<iframe>` player directly in the dashboard — either in the primary panel or as a persistent bottom bar. The embed is Spotify's own UI: album art, progress bar, play/pause, skip.

**How it works technically:**
- Uses Spotify's [iFrame API](https://developer.spotify.com/documentation/embeds/tutorials/using-the-iframe-api): load `https://open.spotify.com/embed/iframe-api/v1`, call `IFrameAPI.createController()` with a playlist or track URI.
- No OAuth setup required for the embed itself — it uses the user's existing Spotify session in the browser.
- The iFrame API exposes `togglePlay()`, `loadUri()`, and playback events.

**Pros:**
- No OAuth setup, no proxy endpoints — the simplest path to a visible player.
- Official Spotify UI — familiar to the family.
- Can load a specific family playlist URI directly.

**Cons:**
- **Requires a logged-in Spotify session in the WebView.** The kiosk WebView would need to be signed into Spotify's web player — this is fragile (sessions expire, cookies clear on WebView restart).
- **Audio autoplay is blocked** by Android WebView's autoplay policy. The user must tap to start playback — acceptable for an intentional "open Spotify" interaction, but the embed will sit silently until tapped.
- The embed has a fixed Spotify-branded look that may clash with the dashboard's editorial aubergine aesthetic.
- The embed cannot show "what's currently playing on another device" — it creates a new playback context in the WebView, which would interrupt whatever is playing on the home speaker.
- Does **not** integrate with the existing context strip or mode system.

**Effort:** ~1 hour to embed, but ongoing maintenance burden (session management, visual mismatch).

---

### Option D — Dedicated Spotify Screen (Second Page / Slide-Over)

**What it is:** A full separate screen — either a second HTML page (`spotify.html`) or a CSS slide-over panel — that fills the entire 1280×800 display with a Spotify-focused UI: large album art, full controls, queue, recently played. Accessed via a prominent button on the main dashboard.

**How it works technically:**
- Two approaches: (a) a second `index.html`-style page served from the same repo, navigated to via `window.location.href`; or (b) a full-screen CSS overlay (`position: fixed; inset: 0`) that slides in over the dashboard.
- Uses the same proxy-based approach as Option B for now-playing data and controls.
- The slide-over approach is preferable — it avoids a full page reload and keeps the dashboard state intact (routine checkboxes, to-do list).

**Pros:**
- Maximum screen space for album art and controls — visually impressive.
- Clean separation of concerns — the dashboard is not cluttered.
- Can include richer features: queue display, recently played, playlist picker.

**Cons:**
- **Most complex to build** — a full second UI surface needs its own layout, CSS, and state.
- The slide-over approach requires careful z-index management and a reliable "close" gesture on a touch screen.
- If implemented as a second page, the dashboard loses its state (routine progress, to-do items) on navigation — requires `localStorage` or URL state to restore.
- The Portal auto-returns to the dashboard after 5 minutes anyway — a full Spotify screen would be interrupted by that timer, which may be frustrating mid-song.
- Overkill for a kitchen dashboard used primarily for glancing, not for music curation.

**Effort:** ~6–8 hours for a polished implementation.

---

## Recommendation

**Build Option A first, then extend to Option B.**

Option A (ambient "Now Playing" in the context strip) is the right starting point for three reasons. First, it matches the dashboard's core design philosophy — every widget in the context strip is ambient, read-only, and glanceable. Music playing in the kitchen is background information, not a primary task. Second, it is already fully designed in the ROADMAP with a step-by-step guide, meaning the implementation path is clear and low-risk. Third, it requires no changes to the existing layout, the routine cards, or the mode-switching system.

Once Option A is live and you have confirmed the proxy token flow works reliably, extending to Option B (adding playback controls to the primary panel) is a natural second step. The architecture is identical — you are just adding three more proxy endpoints and a new `data-mode="spotify"` state. This gives you the ability to tap a small Spotify glyph in the context strip, have the primary panel flip to a large album-art + controls view, and return to to-do mode after 60–90 seconds of inactivity.

Option C (iFrame embed) should be avoided — the session management burden and the conflict with the home speaker make it unsuitable for a kiosk. Option D (dedicated screen) is appealing aesthetically but is disproportionate in effort for a kitchen display, and the 5-minute auto-return timer would make it frustrating to use.

---

## Implementation Summary

| Option | Read-only | Controls | Audio via Terry | OAuth needed | Effort | Recommendation |
|---|---|---|---|---|---|---|
| A — Context strip "Now Playing" | Yes | No | No | Yes (one-time) | ~2h | **Start here** |
| B — Primary panel + controls | Yes | Yes | No | Yes (same token) | ~4h total | **Extend to this** |
| C — iFrame embed | Yes | Yes | Yes (blocked) | No | ~1h + ongoing | Avoid |
| D — Dedicated screen | Yes | Yes | No | Yes | ~8h | Overkill for now |

---

## One-Time OAuth Setup (required for A and B)

This is the only manual step that cannot be automated. From a browser on your Mac or phone:

1. Go to [developer.spotify.com/dashboard](https://developer.spotify.com/dashboard), create an app, note the **Client ID** and **Client Secret**.
2. Add redirect URI: `https://family-dashboard-proxy.vercel.app/api/spotify-callback`.
3. Open the authorisation URL in a browser, approve, capture the `?code=` from the redirect.
4. Exchange the code for a refresh token via `curl` (one command).
5. Store `SPOTIFY_REFRESH_TOKEN`, `SPOTIFY_CLIENT_ID`, `SPOTIFY_CLIENT_SECRET` as Vercel environment variables.

The full curl commands are already documented in `ROADMAP.md` under "📚 Guides for Elul — Spotify Now Playing".
