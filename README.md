# Family Dashboard

Single-file HTML dashboard for the hacked Meta Portal ("Terry"). Runs inside the
kiosk WebView app at `~/Projects/apps/kiosk-webview/`.

Live: https://er-builder.github.io/family-dashboard/

## Editing

Edit `index.html`, commit, push — Portal picks it up on next 30-min refresh
(or restart the kiosk app).

## Configuration

- **OpenWeather key:** edit `OPENWEATHER_API_KEY` near the bottom of `index.html`.
- **Calendar:** replace the `<iframe src="...">` with your own Google Calendar
  embed URL (Settings → your calendar → "Integrate calendar" → "Public URL to this calendar"
  → use the embed builder).
