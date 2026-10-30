# 4. Take over Home: boot into your launcher, and make Home come back to it

## Why the normal way doesn't work

On Android TV you'd declare a `HOME` activity and pick it as the default. **Fire OS ignores that**:
Amazon's launcher stays the home, `cmd package set-home-activity` is refused or reverted, and the
remote's Home key is consumed by the system before apps or accessibility key filters see it
(`onKeyEvent` never receives `KEYCODE_HOME` on Fire OS 7). Disabling Amazon's launcher package is
not an option either: it breaks system UI and Fire OS re-enables it.

The approach that works is **react, don't intercept**.

## The manifest

- Main activity with intent filters for `MAIN` + `HOME` + `DEFAULT`, `MAIN` + `LAUNCHER`, and
  `MAIN` + `LEANBACK_LAUNCHER` (so it also appears in Amazon's apps row as a way back in).
  `launchMode="singleTask"`, excluded from recents, a TV banner (320×180 xhdpi).
- An **AccessibilityService** listening for `typeWindowStateChanged`, with
  `canRetrieveWindowContent=false` (it only needs package and class names).
- A `BOOT_COMPLETED` / `LOCKED_BOOT_COMPLETED` / `MY_PACKAGE_REPLACED` receiver that starts the
  launcher and runs self-repair.
- Permissions: `RECEIVE_BOOT_COMPLETED`, `WRITE_SECURE_SETTINGS` (granted over ADB, below).

## The redirect loop

1. The service sees a window-state change whose package is Amazon's launcher and whose class is
   its home activity (on Fire OS 7: a class name containing `HomeActivity`).
2. It starts your launcher with `NEW_TASK | CLEAR_TOP | SINGLE_TOP` and an extra meaning
   "go to the home page" (so pressing Home while deep in your Settings page resets to Home).
3. **The 5-second lockout.** After Home is pressed, Android defers activity starts from
   background apps for ~5 s (the "app switch" protection). Your first start is silently delayed.
   Retry: start immediately, again at ~5.6 s, and once more at ~8 s, stopping as soon as your
   activity is in front.
4. **Hide the gap with a snapshot.** While waiting, show a full-screen
   `TYPE_ACCESSIBILITY_OVERLAY` window (an accessibility service may add one without the
   "draw over other apps" permission) containing a bitmap of your launcher, so the user never
   sees Amazon's home. Make it focusable and swallow keys so Amazon's home can't act on presses
   behind it. Remove it ~200 ms after your real window reports in front, with a short fade, and
   always on a timeout as a safety net.
5. **Taking the snapshot.** `PixelCopy.request(window, bitmap, …)` (API 26+) copies your window
   exactly. It's a full-screen GPU readback, so don't do it on every focus change: take one when
   the user launches an app from your launcher (that's when they'll come back via Home) and one
   after a few seconds of idle on the home page. Recycle the previous bitmap, unless the overlay is
   showing it at that moment.

## Grants (once, over ADB)

```bash
adb shell pm grant <pkg> android.permission.WRITE_SECURE_SETTINGS
adb shell settings put secure enabled_accessibility_services <pkg>/<pkg>.<ServiceClass>
adb shell settings put secure accessibility_enabled 1
```

Append to the existing `enabled_accessibility_services` value (colon-separated) rather than
overwriting someone's screen reader. Note that Android may store your entry in short form
(`pkg/.Service`): when checking whether it's present, accept both forms or compare
`ComponentName`s, or your heal script will add duplicates.

With `WRITE_SECURE_SETTINGS` the app can re-enable its own service at boot and on every resume,
which makes the takeover survive most resets without a computer.

## The escape hatch (required)

Give the user a way back to Amazon's home that can't be triggered by accident, for example:
**hold Back for 10 seconds** on your home page, with a countdown hint appearing after 2 s. It
sets a "paused" flag (the service stops redirecting) and starts Amazon's home activity explicitly.
Opening your launcher again (from the apps row) or rebooting clears the flag. Long-press Home
can't be used: Fire OS reserves it for its quick menu.

## Behaviour details users notice

- Back on your home page must do nothing (never "exit" the launcher).
- Re-entering from an app should feel intentional: a short "return" animation, unless the snapshot
  cover was used (then the user already saw the screen; animating again looks like a glitch).
- Boot: Fire OS shows its own home first; the boot receiver plus the accessibility service bring
  yours up within seconds. Test three cold reboots (`adb reboot`, wait ~60 s, check `mCurrentFocus`
  in `dumpsys window`).
