# 10. Limits and troubleshooting

## What can't be done without root (tell the user plainly)

- **Intercepting the Home key.** Fire OS handles it first. Redirecting after Amazon's home appears
  is the best available, with a few seconds hidden behind a snapshot.
- **Fully blocking system updates.** The main OTA packages are protected. You can disable the
  forced-update prompt package, and heal after updates.
- **Disabling protected packages** (ad deep-links, crash/log managers, OTA).
- **TV volume from software** on most TVs (HDMI output ignores it; CEC volume isn't exposed to apps).
- **Replacing the boot animation or Amazon's settings screens.**
- **Smart TVs that aren't Android** (TiVo OS, Tizen, webOS, VIDAA): no ADB, nothing to install.

## Troubleshooting

| Symptom | Likely cause and fix |
| --- | --- |
| `adb connect` times out | Different network/VLAN, guest Wi-Fi isolation, ADB debugging off, or the stick changed IP: rescan port 5555. |
| `unauthorized` | Accept "Allow USB debugging?" on the TV (tick Always allow). |
| `Cannot disable a protected package` | Protected; mark it in your list and move on. |
| Lands on Amazon's home after boot | Accessibility service got disabled (re-grant / heal), or the boot receiver isn't registered for `LOCKED_BOOT_COMPLETED`. |
| Home shows Amazon's home for ~5 s | Expected app-switch lockout; add the snapshot cover and launch retries. |
| Heal keeps "re-enabling" the service | You're checking for the long component name but Android stored the short form (`pkg/.Service`). |
| An Amazon package keeps coming back | Normal drift (Alexa especially); heal re-disables it. |
| BLE scan finds nothing | Location permission not granted or location mode off (Android 9 requirement); bulb not in discoverable mode; out of range. |
| GATT writes fail randomly | Concurrent operations; queue them one at a time; retry on 133. |
| Phone app can't reach the bulbs | The stick is holding the connection; disconnect when idle. |
| UI stutters | Phase 6: check idle frames, offscreen layers, mutable bitmaps, perpetual animations. |
| Netflix etc. slow to start | Launcher holding GPU memory in the background (phase 6, fix 1); too many background services. |
| Focused card cut off | `clipToPadding`/`clipChildren` on the container or scroll content. |
| Square grey box over focus | Android's default focus highlight: disable it per view. |
| Nothing focused on a new page | A focusable ScrollView took focus before layout. |
| Music player shows nothing | The app publishes no media session (web-player wrappers), or the notification listener isn't enabled. |
| Music stops when Home is pressed | The app has no background playback (foreground service); nothing the launcher can fix. |
| Agent can't tap / capabilities unchanged | Toggle the accessibility service off and on after changing its XML abilities. |
| Agent presses Play but nothing starts | A pop-up or "keep/clear queue?" dialog is up, or the button ignores ACTION_CLICK (use a gesture tap). |
| Emoji show as a box | Newer than the device's emoji font (Android 9 lacks e.g. 🪩). |
| Stick sleeps anyway | `stay_on_while_plugged_in`, `sleep_timeout`, `screen_off_timeout` reset by an update: heal; or the TV's USB port cut power. |
