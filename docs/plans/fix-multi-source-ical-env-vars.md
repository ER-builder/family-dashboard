# Fix Family Dashboard Calendar — Multi-Source iCal Env Vars (Autonomous)

## Context

The dashboard's calendar is currently broken in production. The proxy code (`~/family-dashboard-proxy/api/cal.js`) was already updated to read `ICAL_URLS` (comma-separated) + `ICAL_LABELS`, but the env vars are in a bad state:

- `ICAL_URLS` in Production has a corrupted value (likely the literal placeholder `<FAMILY_ICAL_URL>,<PERSONAL_ICAL_URL>` from a previous paste — debug endpoint reports `Invalid URL` from both sources).
- `ICAL_URLS` is missing from Preview entirely.
- `ICAL_LABELS` is in an unclear state (earlier writes hit interactive-prompt issues).

Two prior attempts failed:
1. CLI piping (`printf '%s\n\n' | vercel env add ...`) — `vercel env add` blocks on a "which Git branch?" prompt for the preview environment. The `\n\n` trick to dismiss the prompt got captured into the value, corrupting it.
2. Vercel REST API via short-lived personal token — token is rejected as `invalidToken: true` on every endpoint (`/v2/user`, `/v2/teams`, project lookup). Likely created with no scope or in the wrong account. Cannot iterate without user help.

User's directive: **"just fix it — do whatever is needed without my help, get this cal up and running."**

## Key insight

`vercel env add --help` reveals options that weren't used in earlier attempts:

```
vercel env add name [environment] [git-branch] [options]
  --value <VALUE>    Value for the variable (non-interactive). Otherwise use stdin
                     or you will be prompted.
  --force            Force overwrites when a command would normally fail
  --non-interactive  Run without interactive prompts
  -y, --yes          Skip the confirmation prompt
```

`--value` + `--force` + `--non-interactive` + `--yes` together = fully non-interactive add/upsert with no stdin newline pollution and no prompt to dismiss. This is the right tool.

The downside: passing the value as a CLI flag means it's visible in `ps` for the duration of each `vercel env add` call (~1-2 seconds per call). On a single-user Pi where the URLs already exist on disk in `~/.ical-urls` (which we'll shred at the end), this is acceptable residual risk. No new attack surface.

## Files involved

- `~/family-dashboard-proxy/` — Vercel project (already linked, CLI authenticated to scope `ers-projects-959059ce`)
- `~/family-dashboard-proxy/api/cal.js` — proxy code (no changes — already supports `ICAL_URLS` + `ICAL_LABELS` + `?debug=1`)
- `~/.ical-urls` — file with the two secret iCal URLs (currently contains shell-style `URLS="..."` line + `LABELS="..."` line; URLs extracted via regex)
- `~/.vercel-token` — invalid token, will shred (no security cost since token is dead)
- `/home/elul/projects/family-dashboard/ROADMAP.md` — add a one-line note that multi-source iCal is now wired

## Plan

### 1. Verify file state (read-only)
```bash
grep -cE 'https://[^",[:space:]]+\.ics' ~/.ical-urls   # must be 2
```

### 2. Set env vars non-interactively
```bash
URL_LIST=$(grep -oE 'https://[^",[:space:]]+\.ics' ~/.ical-urls | tr '\n' ',' | sed 's/,$//')
LABELS="Family,Elul"
cd ~/family-dashboard-proxy

# ICAL_URLS — sensitive, prod + preview (sensitive vars can't be in development)
vercel env add ICAL_URLS production --sensitive --force --yes --value "$URL_LIST"
vercel env add ICAL_URLS preview    --sensitive --force --yes --value "$URL_LIST"

# ICAL_LABELS — encrypted (default), all 3 envs
vercel env add ICAL_LABELS production  --force --yes --value "$LABELS"
vercel env add ICAL_LABELS preview     --force --yes --value "$LABELS"
vercel env add ICAL_LABELS development --force --yes --value "$LABELS"

unset URL_LIST LABELS
```

`--force` upserts (replaces existing). `--yes` skips confirmation. No stdin involved → no newline corruption. No interactive prompts → no blocking.

### 3. Trigger production redeploy
```bash
vercel --prod --yes
```

### 4. Verify with debug curl
```bash
sleep 5   # let the alias swap in
curl -s "https://family-dashboard-proxy.vercel.app/api/cal?debug=1&t=$(date +%s)" | jq '.diagnostics'
```

Expected: both sources `ok: true`, non-zero `count`, and `totalAfterDedupe > 0`.

If `ok: false` for either source, the URL itself is malformed in `~/.ical-urls`. Bail and report — at that point user input is required.

### 5. Cleanup
```bash
shred -u ~/.ical-urls ~/.vercel-token
```

User should also revoke the (invalid) token at https://vercel.com/account/tokens — I'll mention in the wrap message.

### 6. Roadmap touch-up
Mark the multi-source iCal item as done in `/home/elul/projects/family-dashboard/ROADMAP.md` (the "Calendar — Multi-Source Merge" section). Keep the "Guides for Elul" section since it documents how it was set up.

## Verification

End-to-end:
1. `vercel env ls` shows `ICAL_URLS` in Production + Preview, `ICAL_LABELS` in Production + Preview + Development.
2. Debug curl returns `diagnostics.sources` with both Family and Elul reporting `ok: true` and event counts > 0.
3. Plain (non-debug) curl returns sorted merged events: `curl -s https://family-dashboard-proxy.vercel.app/api/cal | jq '.events | length'` returns ≥1.
4. Dashboard at https://er-builder.github.io/family-dashboard/ shows events from both calendars on next 5-min refresh (cannot verify from Pi — user will see on Terry).

## Out of scope

- Removing the bad ICAL_LABELS placeholder if it landed in production with bad value (the `--force` upsert overwrites it).
- Cleaning up the `.vercel/` directory in `~/family-dashboard-proxy/` (it's the project link metadata, harmless).
- Rotating the actual iCal URLs (those are the user's secret addresses; user owns rotation).
