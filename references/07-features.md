# 7. Extras: weather, Now Playing, Hue lights, wake-ups

## Weather pill and forecast sheet

- Location without GPS or permissions: an IP geolocation service (e.g. ipwho.is) gives approximate
  latitude, longitude and city. Cache it; refetch if the city is missing.
- Forecast: **Open-Meteo** (`api.open-meteo.com/v1/forecast`, free, no key) with
  `current`, `hourly` and `daily` variables and `timezone=auto`. Map WMO weather codes to your
  own icon set, with day and night variants (`is_day`).
- Refresh every ~20 minutes while the home screen is visible; show cached data immediately.
- Sheet: city, big current temperature, condition and today's high/low; the next 12 hours as an
  icon row; 5 days with a min-max range bar coloured from cool to warm. Credit the data source.

## Now Playing pill

- Register a `NotificationListenerService` (it can be empty) and enable it over ADB:
  `cmd notification allow_listener <pkg>/<pkg>.<ListenerClass>`.
- That unlocks `MediaSessionManager.getActiveSessions(listenerComponent)`: titles, artist, album
  art and transport controls of whatever plays (Spotify, YouTube, etc.).
- Show the pill only while a session is playing or paused. OK = play/pause, Left/Right =
  previous/next.

## TV volume

Fire OS ignores software volume changes for HDMI output on most TVs, and CEC volume commands from
apps are not exposed. Test it (`input keyevent VOLUME_UP`, `media volume`); if the TV doesn't
react, say it can't be done rather than shipping a slider that does nothing.

## Philips Hue Bluetooth bulbs (no Bridge)

Many users have Bluetooth Hue bulbs paired only to their phone. The stick can control them
directly over BLE GATT:

- Make the bulb discoverable first: Hue app → Settings → Voice assistants → Amazon Alexa → Make
  discoverable (the bulb then accepts a new BLE bond for a while).
- Scan for the Signify service UUID `0xFE0F` (`0000fe0f-0000-1000-8000-00805f9b34fb`) (needs `ACCESS_FINE_LOCATION` granted and location
  mode on: on Android 9, BLE scans return nothing without it).
- Bond with `createBond()`; watch `ACTION_BOND_STATE_CHANGED`.
- Light control service `932c32bd-0000-47a2-835a-a8d455b859dd`, with characteristics
  `932c32bd-0002-…` power (1 byte: 0/1), `932c32bd-0003-…` brightness (1–254) and
  `932c32bd-0004-…` colour temperature (uint16 little-endian mireds, ~153–454; white-only bulbs
  don't have it). The bulb's name is readable from characteristic
  `97fe6561-0003-4f62-86e9-b71ee2da3d22` (search all services for it).
- Android's GATT client allows **one operation at a time**: queue writes, coalesce repeated writes
  to the same characteristic, retry on status 133, and read state after connecting.
- Bulbs accept few simultaneous connections. Hold the connection only while the lights page is
  open, otherwise close after ~15 s idle, so the phone app keeps working.
- Useful UI: All lights / per-bulb cards (OK toggles, hold for brightness and warmth), scenes
  (Bright, Read, Relax, Night light as brightness + mireds pairs), "Add light" search flow.

### Wake-ups (sunrise fade)

BLE bulbs can't store schedules, so the stick runs them: an exact alarm
(`setExactAndAllowWhileIdle(RTC_WAKEUP, …)`) at wake time minus fade length, then one alarm a
minute stepping brightness up with an ease-in curve from 1 to 254 and warmth from ~2200 K to
~4000 K. Use `goAsync()` and finish a few seconds after the writes. Re-arm all alarms at boot and
after edits; cancel alarms of deleted wake-ups. Caveat to tell the user: the stick must be
powered at that time; if it's powered from the TV's USB port it's off when the TV is off.

### Movie Mode

Dim chosen lights to a level when an app is launched from the launcher, restore the saved
levels when the launcher resumes.

## Your own wallpapers

Scan a folder such as `/sdcard/Pictures/Wallpapers` (needs `READ_EXTERNAL_STORAGE`) alongside
bundled ones; `adb push photo.jpg /sdcard/Pictures/Wallpapers/` adds one. Bundle only images you
may redistribute (NASA and USGS imagery is public domain) and keep a credits file.
