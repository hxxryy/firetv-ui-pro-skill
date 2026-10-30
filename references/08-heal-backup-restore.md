# 8. Keep it: self-heal, backups, restore

Amazon updates and background services undo customisations: packages come back, settings reset,
an update can even remove your accessibility grant. Plan for it.

## Two layers of healing

**On the stick, at every boot and resume (no computer needed)**, using `WRITE_SECURE_SETTINGS`:
- Re-add your accessibility service to `enabled_accessibility_services` and set
  `accessibility_enabled=1` if missing.
- Re-apply the power settings: `stay_on_while_plugged_in=7`, `sleep_timeout=-1` (secure),
  `contextual_screen_off_timeout` (secure). `screen_off_timeout` is a *system* setting and needs
  the `WRITE_SETTINGS` app-op: grant it once with `appops set <pkg> WRITE_SETTINGS allow`.
- Re-arm alarms (wake-ups), restart your launcher.

**From the computer, a `heal` script** that is safe to run any time (and can run on a schedule):
1. Connect; if the stick isn't reachable, log it and exit 0 (it's probably off).
2. Launcher missing? Reinstall and re-grant.
3. Accessibility service missing (accept both the long and short component forms)? Re-enable.
4. Any package from the debloat list enabled again? Re-run the debloat.
5. Notification listener disallowed? `cmd notification allow_listener …` again.
6. Screensaver back on, sleep timer back? Turn them off again.
7. Log what was fixed, or "all good".

Test it by breaking things deliberately (disable your service, re-enable a package, set a sleep
timeout) and running it.

## Backups

A `backup` script that snapshots, into a timestamped folder:
- `pm list packages -d`, `-e`, `-3`
- `settings list system`, `secure`, `global`
- your APK and **your signing keystore**

Keep the first ("original") snapshot forever; prune the rest to the newest ~10.

## Restore to stock (always ship this)

Ask for confirmation, then: `pm enable` every package in the debloat list, turn Amazon's
screensaver back on, restore the default sleep and animation settings, remove your service from
`enabled_accessibility_services`, uninstall your launcher. The stick is back to Amazon's home.
A factory reset (Settings → My Fire TV → Reset) also undoes everything.

## Updating the launcher later

Rebuild, then `adb install -r` with the **same keystore**, re-run the grants (they usually
survive an update, but it's cheap), and re-compile with `speed`. Keep a README with these steps
for the user's future self.

## Source control

Keep the project in a **private** repository if it contains the signing key or device details.
Ignore build output, backups and logs.
