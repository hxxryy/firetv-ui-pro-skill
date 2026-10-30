---
name: firetv-ui-pro
description: Turn an Amazon Fire TV Stick (Fire OS 6/7/8) into a fast, ad-free, Apple TV-style device with a custom Liquid Glass home screen. Covers finding the stick over Wi-Fi with ADB, debloating (ads, Alexa, screensaver, sleep, updates), building and installing a custom launcher without Gradle, making the remote's Home button return to it, iOS-26-style glass UI design for a 10-foot screen, GPU/memory performance tuning on weak TV hardware, extras (weather, Now Playing, Philips Hue Bluetooth bulbs), self-healing after Amazon updates, and on-device QA. Use when someone wants to debloat, speed up, customise, replace the launcher of, or build a home-screen app for a Fire TV / Fire Stick / Android TV box.
---

# Fire TV UI Pro

A field guide for replacing a Fire TV Stick's ad-heavy home screen with your own fast, clean,
Apple TV-style launcher, and keeping it that way. Everything here was learned on real hardware
(Fire TV Stick 3rd gen, Fire OS 7.7.1.7 / Android 9, 1 GB RAM): measured, not guessed.

## How to use this skill

Work in phases. Each phase has a reference file with the detail; read it when you get there.
Stop and ask the user at every point marked **ASK**: their TV, their apps and their tolerance
for risk are theirs to decide.

| Phase | Goal | Read |
| --- | --- | --- |
| 1. Connect | Find the stick on Wi-Fi, connect ADB, record device facts | [references/01-connect.md](references/01-connect.md) |
| 2. Debloat | Disable ads, Alexa, bloat, screensaver, sleep timer; speed tweaks | [references/02-debloat.md](references/02-debloat.md) |
| 3. Build | A launcher APK with no Gradle and no Android Studio | [references/03-build-without-gradle.md](references/03-build-without-gradle.md) |
| 4. Take over Home | Boot into your launcher; make the Home button come back to it | [references/04-launcher-takeover.md](references/04-launcher-takeover.md) |
| 5. Design | Liquid Glass UI that reads well on any wallpaper at 3 metres | [references/05-liquid-glass-design.md](references/05-liquid-glass-design.md) |
| 6. Speed | Smooth on a weak GPU, light on RAM so other apps launch faster | [references/06-performance.md](references/06-performance.md) |
| 7. Extras | Weather, Now Playing, Hue Bluetooth lights, wake-ups, screensaver | [references/07-features.md](references/07-features.md) |
| 8. Keep it | Self-heal after Amazon updates, backups, a way back to stock | [references/08-heal-backup-restore.md](references/08-heal-backup-restore.md) |
| 9. QA | Deep on-device smoke test before calling it done | [references/09-qa-checklist.md](references/09-qa-checklist.md) |

Limits and fixes for common failures: [references/10-limits-troubleshooting.md](references/10-limits-troubleshooting.md).

## Ground rules

- **ASK before disabling anything.** Show the user a grouped list (ads, Alexa/voice, Kids,
  Photos/Music, Silk/Games, smart home, updates) and let them choose. Never remove their own apps.
- **Back up first** (package lists, settings) so every change can be undone. Ship a restore path.
- **Only `pm disable-user --user 0`, never `pm uninstall` of system packages.** Disabling is
  reversible with `pm enable`; a factory reset also undoes it. Never root, never flash.
- **Keep a way back.** The user must always be able to reach Amazon's settings and, if they want,
  Amazon's home screen (e.g. a long-hold escape hatch).
- **Measure on the device.** TV hardware is far weaker than a phone; a design that looks fine in
  your head can run at 20 fps. Use `dumpsys gfxinfo` (phase 6) before and after each change.
- **Verify on screen.** Take `screencap` screenshots of every page and sheet and look at them
  (phase 9). Most launcher bugs are visual: clipped focus states, unreadable text, missing focus.
- **Treat Amazon updates as hostile to your setup.** They re-enable packages and reset settings.
  Build healing in from the start (phase 8).

## Architecture at a glance

```
Mac / PC  ──ADB over Wi-Fi (5555)──►  Fire TV Stick
  build script (aapt2→javac→d8→zipalign→apksigner)      ├─ your launcher APK
  debloat / heal / backup / restore scripts             │    ├─ Activity: HOME + LEANBACK_LAUNCHER filters
                                                        │    ├─ AccessibilityService: watches for Amazon's
                                                        │    │   home window, sends the user back to yours
                                                        │    ├─ BOOT_COMPLETED receiver: start + self-repair
                                                        │    └─ NotificationListener: media sessions
                                                        └─ settings: screensaver off, sleep never, anims 0.5x
```

The non-obvious parts, each explained in its reference file:

1. Fire OS ignores the "default home app" choice and keeps Amazon's launcher; the Home key is
   handled before any app or accessibility service sees it. You win by *reacting* to Amazon's
   home appearing, not by intercepting the key (phase 4).
2. After Home is pressed, Android blocks app launches for ~5 seconds. A pixel snapshot of your
   launcher, shown as an accessibility overlay, hides that gap (phase 4).
3. `WRITE_SECURE_SETTINGS`, granted once over ADB, lets the app repair its own accessibility
   service and power settings at every boot, without a computer (phases 4 and 8).
4. Fire OS keeps an app with a bound accessibility service "foreground", so Android never tells it
   to release GPU memory. Release it yourself when another app opens: on a 1 GB stick that's
   ~90 MB more for Netflix (phase 6).

## Definition of done

- Boots straight into the custom launcher, every time (test 3 cold reboots).
- Home from inside any app returns to it; a documented escape hatch returns to Amazon's home.
- No ads, no Alexa (if chosen), no Amazon screensaver, no sleep timer (TV-off still sleeps it).
- Idle home screen does not redraw at 60 fps; other pages don't redraw at all when idle.
- Every page, sheet and focus state checked in screenshots; all text readable on a white wallpaper.
- `heal` script fixes a deliberately broken setup; `restore` returns the stick to stock.
- A README for the user: how to use it, how to update it, what can't be done and why.
