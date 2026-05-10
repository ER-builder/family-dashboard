# Spotify on Terry — Your Complete Setup Guide

Everything in the repos is already built and pushed. This guide covers the three things only you can do: getting the Spotify token, deploying the proxy, and updating the kiosk app on Terry.

---

## What's already done (no action needed)

- **`family-dashboard-proxy`** — a new `/api/spotify-now` endpoint is live in the repo. It fetches what's currently playing on your Spotify account using a refresh token you'll provide below.
- **`family-dashboard`** — the dashboard now has:
  - A **green ♫ Spotify button** in the bottom-left corner (mirrors the exit button). Tap it to launch the Spotify app on Terry.
  - A **compact "now playing" strip** in the context bar (alongside weather/transit/stars). Appears automatically when music is playing. Shows album art, track name, and artist. Hidden when nothing is playing.
  - A **full player card** that slides into the primary panel when you tap the strip. Shows large album art, track/artist, and ⏮ ⏸ ⏭ controls. Auto-closes after 90 seconds.

Terry will pick up the dashboard changes on its next 30-minute auto-refresh, or immediately if you restart the kiosk app.

---

## Part 1 — Get your Spotify refresh token (one-time, ~5 minutes)

This is the only fiddly step. You do it once from your Mac browser and never again.

### Step 1.1 — Create a Spotify developer app

1. Open [developer.spotify.com/dashboard](https://developer.spotify.com/dashboard) in your browser and log in with your Spotify account.
2. Click **Create app**.
3. Fill in any name (e.g. "Terry Dashboard") and description.
4. In the **Redirect URIs** field, enter exactly:
   ```
   https://family-dashboard-proxy.vercel.app/api/spotify-callback
   ```
5. Under "Which API/SDKs are you planning to use?", tick **Web API**.
6. Click **Save**.
7. On the app page, click **Settings**. Copy your **Client ID** and **Client Secret** — you'll need them in a moment.

### Step 1.2 — Authorise your account (get the code)

Open this URL in your browser, replacing `YOUR_CLIENT_ID` with the Client ID you just copied:

```
https://accounts.spotify.com/authorize?client_id=YOUR_CLIENT_ID&response_type=code&redirect_uri=https%3A%2F%2Ffamily-dashboard-proxy.vercel.app%2Fapi%2Fspotify-callback&scope=user-read-currently-playing+user-read-playback-state
```

Spotify will ask you to approve access. Click **Agree**. Your browser will then redirect to a URL that looks like:

```
https://family-dashboard-proxy.vercel.app/api/spotify-callback?code=AQD...very-long-string...
```

The page will probably show an error (the callback endpoint doesn't exist yet — that's fine). **Copy the `code=` value from the URL** — everything after `code=` and before any `&`. It looks like a long random string starting with `AQ`.

### Step 1.3 — Exchange the code for a refresh token

Open Terminal on your Mac and run this command, replacing the three placeholders:

```bash
curl -X POST https://accounts.spotify.com/api/token \
  -d grant_type=authorization_code \
  -d code=PASTE_YOUR_CODE_HERE \
  -d redirect_uri=https://family-dashboard-proxy.vercel.app/api/spotify-callback \
  -u YOUR_CLIENT_ID:YOUR_CLIENT_SECRET
```

The response will be a JSON blob. Find the `"refresh_token"` field — it's a long string. Copy it.

---

## Part 2 — Deploy the proxy with your credentials (~3 minutes)

You need the Vercel CLI. If you don't have it:

```bash
npm install -g vercel
```

Then, from the `family-dashboard-proxy` directory on your Mac (clone it first if needed):

```bash
cd ~/Projects/apps/family-dashboard-proxy   # or wherever you keep it
git pull                                     # get the new spotify-now.js endpoint
```

Add the three environment variables to Vercel (run each line separately — Vercel will prompt you to confirm):

```bash
printf '%s' 'YOUR_CLIENT_ID'     | vercel env add SPOTIFY_CLIENT_ID     production --sensitive
printf '%s' 'YOUR_CLIENT_SECRET' | vercel env add SPOTIFY_CLIENT_SECRET  production --sensitive
printf '%s' 'YOUR_REFRESH_TOKEN' | vercel env add SPOTIFY_REFRESH_TOKEN  production --sensitive
```

Then deploy:

```bash
vercel --prod
```

**Test it works** — open this URL in your browser while music is playing on your Spotify account:

```
https://family-dashboard-proxy.vercel.app/api/spotify-now
```

You should see JSON like `{"isPlaying":true,"track":"Song Name","artist":"Artist",...}`. If you see `{"isPlaying":false}` it means nothing is playing right now (that's correct behaviour). If you see an error, double-check the three env vars.

---

## Part 3 — Update the kiosk app on Terry (~20 minutes)

This is the only part that requires rebuilding the Android app. The kiosk-webview needs to handle two new `kiosk://` URIs:

- `kiosk://spotify/launch` — launch the Spotify APK
- `kiosk://spotify/play`, `pause`, `prev`, `next` — playback controls (optional for now)

### Step 3.1 — Sideload the Spotify APK

First, test whether Spotify runs on Terry at all:

1. Download the Spotify APK for **arm64-v8a** from [APKMirror](https://www.apkmirror.com/apk/spotify-ab/spotify-music-podcasts/). Search for "Spotify" and pick the latest version. On the download page, choose the **arm64-v8a** variant (not armeabi-v7a, not universal).
2. In Terminal on your Mac:
   ```bash
   adb connect 192.168.1.116:5555
   adb install -r ~/Downloads/spotify-arm64.apk
   ```
3. Test that it launches:
   ```bash
   adb shell am start -n com.spotify.music/.MainActivity
   ```
   If Spotify opens on Terry and you can log in and play music — you're good. If it crashes, try the `armeabi-v7a` variant instead.

### Step 3.2 — Add the URI handler to kiosk-webview

Open `kiosk-webview` in Android Studio (or your editor). In `MainActivity.kt`, find the `shouldOverrideUrlLoading` method inside your `WebViewClient`. It already handles `kiosk://exit`. Add the Spotify handler alongside it:

```kotlin
override fun shouldOverrideUrlLoading(view: WebView?, request: WebResourceRequest?): Boolean {
    val url = request?.url?.toString() ?: return false

    // Existing exit handler
    if (url == "kiosk://exit") {
        finish()
        return true
    }

    // Spotify handlers
    if (url.startsWith("kiosk://spotify/")) {
        val cmd = url.removePrefix("kiosk://spotify/")
        handleSpotifyCommand(cmd)
        return true
    }

    return super.shouldOverrideUrlLoading(view, request)
}

private fun handleSpotifyCommand(cmd: String) {
    when (cmd) {
        "launch" -> {
            // Launch the Spotify app (sideloaded APK)
            val intent = packageManager.getLaunchIntentForPackage("com.spotify.music")
            if (intent != null) {
                startActivity(intent)
            } else {
                // Spotify not installed — open a toast or do nothing
                android.widget.Toast.makeText(this, "Spotify not installed", android.widget.Toast.LENGTH_SHORT).show()
            }
        }
        // Playback controls via broadcast intent (Spotify listens for these)
        "play"  -> sendSpotifyBroadcast("com.spotify.mobile.android.ui.widget.PLAY")
        "pause" -> sendSpotifyBroadcast("com.spotify.mobile.android.ui.widget.PAUSE")
        "prev"  -> sendSpotifyBroadcast("com.spotify.mobile.android.ui.widget.PREVIOUS")
        "next"  -> sendSpotifyBroadcast("com.spotify.mobile.android.ui.widget.NEXT")
    }
}

private fun sendSpotifyBroadcast(action: String) {
    val intent = Intent(action)
    intent.`package` = "com.spotify.music"
    sendBroadcast(intent)
}
```

> **Note on playback controls:** Spotify's broadcast intents (`com.spotify.mobile.android.ui.widget.*`) work on sideloaded APKs. They do not require any SDK — they are standard Android broadcasts that Spotify registers. The play/pause/prev/next buttons on the dashboard will work once Spotify is running in the background.

### Step 3.3 — Build and install the updated kiosk app

From your Mac Terminal:

```bash
cd ~/Projects/apps/kiosk-webview
./build.sh
adb connect 192.168.1.116:5555
adb install -r app/build/outputs/apk/release/app-release.apk
```

Then restart the kiosk app on Terry (or reboot Terry) so the new version takes effect.

---

## How it all works once set up

| What you do | What happens |
|---|---|
| Music starts playing anywhere on your Spotify account (phone, laptop, speaker) | Within 30 seconds, the album art + track name appears in the dashboard's context strip |
| Tap the now-playing strip | The primary panel flips to a full player card with large album art and controls |
| Tap ⏮ ⏸ ⏭ on the card | The kiosk app sends a broadcast to the Spotify APK, which controls playback |
| Tap ✕ or wait 90 seconds | The full card closes, primary panel returns to to-do |
| Tap the green ♫ button (bottom-left) | The Spotify app opens on Terry so you can browse and pick music |
| Music stops | The now-playing strip disappears automatically |

---

## Troubleshooting

**The now-playing strip never appears**
Check `https://family-dashboard-proxy.vercel.app/api/spotify-now` in a browser while music is playing. If it returns an error, the Vercel env vars are missing or wrong.

**Spotify APK crashes on launch**
Try the `armeabi-v7a` APK variant from APKMirror instead of `arm64-v8a`. Some Portal models use the 32-bit ABI.

**The green Spotify button does nothing**
The kiosk-webview hasn't been updated yet (Step 3.2–3.3 not done). The button fires `kiosk://spotify/launch` which the old kiosk app doesn't know about.

**Playback controls don't work**
The Spotify broadcast intents require Spotify to already be running in the background. Launch it first via the green button, then use the controls from the dashboard.
