# Handoff — Spotify auto-return is now music-gated (2026-06-13)

**For the next agent (running on Elul's Mac).** Pick up an in-progress task. Most of
it is done; the only thing left is installing to Terry and verifying on-device,
which was blocked because Terry was offline.

## The goal (Elul's request)

> "Change the auto-back-to-dashboard when I'm on Spotify — it's annoying. Only
> when on Spotify, keep the manual back button we have today + back to dash if
> the user stopped the music (after 2 min of no music)."

Concretely:
- **On Spotify, music playing → never auto-return.** Manual OverlayService
  "← Dashboard" pill is the only way back.
- **On Spotify, music stopped → auto-return after 2 min of continuous silence.**
  Brief pauses / track gaps under 2 min do not count.
- **Non-Spotify backgrounding → unchanged** (flat `AUTO_RETURN_MS` = 2 min).
- **In-call deferral → unchanged.**

This **reverses** the old "music is intentionally NOT a deferral signal" rule
(the flat 2-min yank out of Spotify was annoying in real use).

## What's already DONE

### Dashboard repo (this repo, `ER-builder/family-dashboard`)
- `AGENTS.md` kiosk section + `ROADMAP.md` "Open Tests / Follow-ups" updated to
  document the new music-gated behavior.
- Committed as `78f406e` and **pushed to both `main` and
  `claude/spotify-back-dashboard-xuaijt`**. GitHub Pages only deploys
  `index.html`, which was NOT touched — there is no dashboard-side code change.

### Kiosk app (`kiosk-webview`, NOT a git repo — local only at `/Users/elul/Projects/apps/kiosk-webview/`)
Code changes were **applied on this Mac and compiled successfully**
(`BUILD SUCCESSFUL`, APK at `app/build/outputs/apk/debug/app-debug.apk`).
Backups exist: `MainActivity.kt.bak`, `AutoReturnReceiver.kt.bak`.

Files: `app/src/main/java/xyz/erapps/kiosk/`

**MainActivity.kt — 3 edits:**

1. In `launchSpotifyApk()`, on successful `startActivity(intent)`, set the flag:
```kotlin
            try {
                startActivity(intent)
                // 2026-06-13: now on Spotify - auto-return switches to music-gated
                // polling (see AutoReturnReceiver). Reset the silence clock.
                getSharedPreferences("kiosk_prefs", MODE_PRIVATE).edit()
                    .putBoolean("on_spotify", true)
                    .putLong("music_stopped_since", 0L)
                    .apply()
            } catch (e: Exception) {
                Log.w(TAG, "launchSpotifyApk failed", e)
            }
```

2. In `onStart()`, after `cancelAutoReturn()`, clear the flags:
```kotlin
        cancelAutoReturn()
        // Back on the dashboard - clear the Spotify music-gate state so the next
        // backgrounding starts fresh.
        getSharedPreferences("kiosk_prefs", MODE_PRIVATE).edit()
            .putBoolean("on_spotify", false)
            .putLong("music_stopped_since", 0L)
            .apply()
        try { stopService(Intent(this, OverlayService::class.java)) } catch (_: Exception) {}
```

3. In `scheduleAutoReturn()`, pick a shorter initial delay when on Spotify:
```kotlin
        val pi = PendingIntent.getBroadcast(this, 1, intent, flags)
        val alarmManager = getSystemService(ALARM_SERVICE) as AlarmManager
        // 2026-06-13: on Spotify, poll music state sooner (MUSIC_POLL_MS) so the
        // silence countdown in AutoReturnReceiver is measured from when music
        // actually stops, not a flat AUTO_RETURN_MS later.
        val onSpotify = getSharedPreferences("kiosk_prefs", MODE_PRIVATE)
            .getBoolean("on_spotify", false)
        val delay = if (onSpotify) AutoReturnReceiver.MUSIC_POLL_MS else AUTO_RETURN_MS
        val triggerAt = System.currentTimeMillis() + delay
```

**AutoReturnReceiver.kt — full file (as applied):**
```kotlin
package xyz.erapps.kiosk

import android.app.AlarmManager
import android.app.PendingIntent
import android.content.BroadcastReceiver
import android.content.Context
import android.content.Intent
import android.media.AudioManager
import android.os.Build

/**
 * Fired by AlarmManager after the kiosk goes to background (manual exit,
 * incoming call, screen-off, launching Spotify, etc.).
 *
 *   1. ACTIVE CALL -> don't interrupt. Reschedule for RESCHEDULE_MS later.
 *
 *   2. ON SPOTIFY (music-gated, 2026-06-13) -> do NOT auto-return while music
 *      is playing. Re-check every MUSIC_POLL_MS; only return after
 *      MUSIC_GRACE_MS of CONTINUOUS silence (brief pauses / track gaps under
 *      that don't count). The OverlayService "<- Dashboard" pill is the
 *      deliberate manual escape hatch while music plays. (Reverses the earlier
 *      "music is never a deferral signal" rule - the flat 2-min yank out of
 *      Spotify was annoying in real use.)
 *
 *   3. OTHERWISE -> bring the dashboard back.
 */
class AutoReturnReceiver : BroadcastReceiver() {
    companion object {
        const val RESCHEDULE_MS = 2L * 60L * 1000L
        const val ACTION = "xyz.erapps.kiosk.AUTO_RETURN"
        // Spotify music-gate: how often to re-check music, and how long music
        // must be continuously silent before we return to the dashboard.
        const val MUSIC_POLL_MS = 30L * 1000L
        const val MUSIC_GRACE_MS = 2L * 60L * 1000L
    }

    override fun onReceive(context: Context, intent: Intent?) {
        if (isInCall(context)) {
            // Don't disrupt an active call. Reschedule and try again later.
            reschedule(context, RESCHEDULE_MS)
            return
        }

        // Spotify music-gate: stay out while music plays; return only after
        // MUSIC_GRACE_MS of continuous silence.
        val prefs = context.getSharedPreferences("kiosk_prefs", Context.MODE_PRIVATE)
        if (prefs.getBoolean("on_spotify", false)) {
            if (isMusicActive(context)) {
                prefs.edit().putLong("music_stopped_since", 0L).apply()
                reschedule(context, MUSIC_POLL_MS)
                return
            }
            val stoppedSince = prefs.getLong("music_stopped_since", 0L)
            val now = System.currentTimeMillis()
            when {
                stoppedSince == 0L -> {
                    prefs.edit().putLong("music_stopped_since", now).apply()
                    reschedule(context, MUSIC_POLL_MS)
                    return
                }
                now - stoppedSince < MUSIC_GRACE_MS -> {
                    reschedule(context, MUSIC_POLL_MS)
                    return
                }
                // else: silent for >= MUSIC_GRACE_MS -> fall through and return.
            }
        }

        // Returning to the dashboard -> clear the music-gate state.
        prefs.edit()
            .putBoolean("on_spotify", false)
            .putLong("music_stopped_since", 0L)
            .apply()

        val launch = Intent(context, MainActivity::class.java).apply {
            addFlags(Intent.FLAG_ACTIVITY_NEW_TASK or Intent.FLAG_ACTIVITY_REORDER_TO_FRONT)
        }
        try {
            context.startActivity(launch)
        } catch (_: Exception) {
            // Best-effort; some OEMs restrict background-launches. Next onStop
            // (or BootReceiver) will reschedule and recover.
        }
    }

    private fun isInCall(context: Context): Boolean {
        return try {
            val audio = context.getSystemService(Context.AUDIO_SERVICE) as? AudioManager
                ?: return false
            val mode = audio.mode
            // 3 = MODE_IN_COMMUNICATION (VoIP / Portal video), 2 = MODE_IN_CALL.
            mode == AudioManager.MODE_IN_COMMUNICATION || mode == AudioManager.MODE_IN_CALL
        } catch (_: Exception) {
            false
        }
    }

    private fun isMusicActive(context: Context): Boolean {
        return try {
            val audio = context.getSystemService(Context.AUDIO_SERVICE) as? AudioManager
                ?: return false
            audio.isMusicActive
        } catch (_: Exception) {
            false
        }
    }

    private fun reschedule(context: Context, delayMs: Long) {
        val pendingIntent = Intent(context, AutoReturnReceiver::class.java)
            .setAction(ACTION)
        val flags = if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.M)
            PendingIntent.FLAG_UPDATE_CURRENT or PendingIntent.FLAG_IMMUTABLE
        else
            PendingIntent.FLAG_UPDATE_CURRENT
        val pi = PendingIntent.getBroadcast(context, 1, pendingIntent, flags)
        val alarmManager = context.getSystemService(Context.ALARM_SERVICE) as AlarmManager
        val triggerAt = System.currentTimeMillis() + delayMs
        try {
            if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.M) {
                alarmManager.setExactAndAllowWhileIdle(AlarmManager.RTC_WAKEUP, triggerAt, pi)
            } else {
                alarmManager.setExact(AlarmManager.RTC_WAKEUP, triggerAt, pi)
            }
        } catch (_: SecurityException) {
            try {
                alarmManager.setExact(AlarmManager.RTC_WAKEUP, triggerAt, pi)
            } catch (_: Exception) {
                alarmManager.set(AlarmManager.RTC_WAKEUP, triggerAt, pi)
            }
        }
    }
}
```

## What's PENDING — the only remaining work

Install the built APK to Terry and verify on-device. This was blocked on
2026-06-13 evening because **Terry was unreachable**: its MAC
(`a4:0e:2b:74:d4:85`) was absent from the LAN, no host had ADB port 5555 open,
and the old IPs `.114` / `.116` are now reassigned to other devices. Terry was
asleep / powered off / rebooted (which closes the WiFi-ADB port). Elul was away
and couldn't poke it physically.

### Resume steps (run when Terry is awake on WiFi)

1. Wake Terry (tap its screen) so it rejoins WiFi.
2. Reconnect ADB:
   ```bash
   adb connect 192.168.1.116:5555
   ```
   If the IP shifted, rediscover (but DON'T trust `.114`/`.116` blindly — they're
   other devices now):
   ```bash
   for i in $(seq 1 254); do ping -c1 -t1 192.168.1.$i >/dev/null 2>&1 & done; wait
   arp -an | grep -i 'a4:e:2b:74:d4:85\|a4:0e:2b:74:d4:85'
   ```
   The `~/bin/terry` wrapper is also meant to auto-discover via ARP — `terry restart`
   alone may self-connect.
3. If `adb connect` is refused even with Terry awake → the port closed on a reboot.
   Plug Terry into the Mac via USB once, run `adb tcpip 5555`, unplug, retry step 2.
4. Install (swap `<ip>` for the real one):
   ```bash
   adb -s <ip>:5555 install -r /Users/elul/Projects/apps/kiosk-webview/app/build/outputs/apk/debug/app-debug.apk
   terry restart
   ```
   The APK is already built from the current source, so no rebuild is strictly
   needed. If you changed any `.kt`, rebuild first:
   `/Users/elul/Projects/apps/kiosk-webview/build.sh`

### Verify on Terry
- Open Spotify, play music → kiosk does **NOT** auto-return while music plays.
- Floating "← Dashboard" pill still returns to the dashboard instantly.
- Stop the music → returns to dashboard after ~2 min of silence.
- Brief pause / track gap < 2 min → does **NOT** return.
- Non-Spotify backgrounding → still returns after the flat 2 min.
- Incoming call → not interrupted.

Diagnostic log: `adb -s <ip>:5555 logcat -s TerryKiosk`

## One nuance to confirm with Elul

The 2-min silence clock also counts "on Spotify but nothing playing" (e.g.
browsing for a playlist). So sitting in Spotify for >2 min without playing will
return to the dashboard. This matches the literal "2 min of no music" ask. If
Elul would rather the countdown only start *after* music has played at least
once this session, adjust `AutoReturnReceiver`: track a `music_started` flag in
`kiosk_prefs` (set true once `isMusicActive` is observed true) and only begin the
silence countdown when `music_started` is true.

## After it's verified
- Tick the boxes in `ROADMAP.md` → "Open Tests / Follow-ups" → the
  2026-06-12 Spotify auto-return entry.
- The `kiosk-webview` change is NOT version-controlled (not a repo), so there's
  nothing to commit there — the on-device APK + the local `.kt` files ARE the
  source of truth. Consider whether to git-init that project someday.
