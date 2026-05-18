# SDK rebuild — execution plan for today (2026-05-18)

Companion to `2026-05-18-sdk-rebuild-to-10.md`. That doc is the architecture;
this is the wall-clock playbook for shipping it in one day.

## Approach

- **2 agents.** One parallel for OAuth re-issue (pure human-in-loop work);
  one main (this terminal) for all the code.
- **Build `index-sdk.html` as a separate file** during the work. Family's
  Terry keeps running the live `index.html` (9.5 ship). Swap at the end.
- **Two hard verification gates** at Terry. Don't proceed past either if it
  fails. Each is a 5-10 min check on the actual device.
- **Phase E (picker UX) deferred** — keep the Spotify APK launcher as today's
  "browse music on Terry's screen" path. Building an in-dashboard picker is
  a follow-on if the family ever asks.
- **Phase G (soak) deferred** to tomorrow — runs in real use after we swap
  the default index.

## Wall-clock estimate

| Phase | Wall-clock | Notes |
|---|---|---|
| A — OAuth scope re-issue | 30 min | Parallel agent + user browser |
| B — Audio playback spike | 1 h | Build + verify on Terry |
| GATE 1 | 10 min | User loads sdk-test-v2.html, confirms audio + Connect |
| C — Player layer in index-sdk.html | 3 h | Full SDK integration |
| GATE 2 | 15 min | User loads index-sdk.html, controls work, state syncs |
| D — Phone Connect flow | 15 min | User tests from phone(s) |
| F — Strip dead code + APK rebuild | 1.5 h | Includes kiosk rebuild on user side |
| **Total** | **~6.5 h** | Could compress if no gate fails |

## Hand-off contract

**User runs in parallel terminal (start now):**
The Phase A prompt below. Reports "DONE" when refresh token has the
new scopes deployed to Vercel.

**User runs in this terminal (at each gate):**
- GATE 1: `adb -s 192.168.1.114:5555 shell am start -a android.intent.action.VIEW -d "https://er-builder.github.io/family-dashboard/sdk-test-v2.html"` → confirm audio plays + describe what's on screen
- GATE 2: same command but URL `.../family-dashboard/index-sdk.html` → confirm card renders + tap controls work
- Phase D: from phone Spotify, tap Devices → Terry → music plays
- Phase F: `cd /Users/elul/Projects/apps/kiosk-webview && ./build.sh && adb -s 192.168.1.114:5555 install -r app/build/outputs/apk/debug/app-debug.apk && adb -s 192.168.1.114:5555 shell am force-stop xyz.erapps.kiosk && adb -s 192.168.1.114:5555 shell am start -n xyz.erapps.kiosk/.MainActivity`

**This terminal handles:**
Building sdk-test-v2.html, index-sdk.html, the eventual swap, Phase F
Kotlin strip preparation, all commits + pushes, AGENTS.md updates.

## Parallel agent prompt (Phase A — paste into new Claude Code terminal)

```
Execute Phase A of the SDK rebuild plan at
/Users/elul/Projects/family-dashboard/docs/plans/2026-05-18-sdk-rebuild-to-10.md

Goal: get a new Spotify refresh token with the expanded scopes
(streaming, user-modify-playback-state, user-read-playback-state,
user-read-email, plus the existing user-read-currently-playing and
user-read-recently-played) and deploy it to the
family-dashboard-proxy Vercel project's SPOTIFY_REFRESH_TOKEN env var.

Steps the plan describes (AGENTS.md section "Refresh-token recovery"
in /Users/elul/Projects/family-dashboard/AGENTS.md has the canonical
curl flow):
1. Pull current env from Vercel into a local file (don't commit it):
   cd /Users/elul/Projects/apps/family-dashboard-proxy
   vercel env pull .env.local
2. Read SPOTIFY_CLIENT_ID and SPOTIFY_CLIENT_SECRET from that file
3. Construct the authorize URL:
   https://accounts.spotify.com/authorize?
     client_id=<CLIENT_ID>&response_type=code&
     redirect_uri=https%3A%2F%2Ffamily-dashboard-proxy.vercel.app%2Fapi%2Fspotify-callback&
     scope=streaming+user-read-email+user-read-playback-state+user-modify-playback-state+user-read-currently-playing+user-read-recently-played
4. Print the URL to me (the user). I'll open in browser, approve,
   and paste back the code= value from the resulting 404 URL
5. Run the curl POST to exchange the code for a new refresh_token
6. Update SPOTIFY_REFRESH_TOKEN in the family-dashboard-proxy Vercel
   project env (production) — use `vercel env rm` then `vercel env add`
   pattern; mark sensitive
7. Redeploy: `vercel --prod` (or trigger via git push of an empty
   commit if redeploy needs to be triggered cleanly)
8. Verify by curling
   https://family-dashboard-proxy.vercel.app/api/spotify-now
   and confirming it returns valid JSON (not 500)
9. Delete the local .env.local immediately
10. Report DONE back to me

Don't touch any other code. This is config-only.
Don't commit .env.local. Don't paste any tokens into chat.
```

## Hard rules during build

- I do not edit `index.html` (the live dashboard) during the build.
  All work lives in `sdk-test-v2.html` and `index-sdk.html` until the
  swap at the end.
- I commit + push per phase so each step is independently revertable.
- Every phase has a clear go/no-go signal — no "looks fine to me."
- If a gate fails, we stop and triage. Better to ship 9.5 than ship
  broken 10.
