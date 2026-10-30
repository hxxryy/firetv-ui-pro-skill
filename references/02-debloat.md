# 2. Debloat: ads, Alexa, clutter, screensaver, sleep

## Back up first

Save `pm list packages -d` (already disabled), `-e` (enabled) and `settings list system|secure|global`
to a timestamped folder before touching anything. The restore script (phase 8) re-enables exactly
what you disabled.

## ASK: what to remove

Present groups, not package names. Typical Fire OS groups and representative packages (names vary
by model and version, so always check with `pm list packages | grep amazon` first):

| Group | What the user loses | Packages seen on Fire OS 7.7 |
| --- | --- | --- |
| Ads, sponsored rows, shopping | Ads, "sneak peek" previews, Amazon shopping | `com.amazon.kso.blackbird`, `com.amazon.hybridadidservice`, `com.amazon.sneakpeek`, `com.amazon.shoptv.client`, `com.amazon.shoptv.firetv.client`, `com.amazon.minitv.android.app`, `com.amazon.tv.easyupgrade`, `com.amazon.tv.website_launcher` |
| Viewing tracking & telemetry | Nothing visible | `com.amazon.tv.acr` (content recognition), `com.amazon.dp.logger`, `com.amazon.tv.fw.metrics`, `com.amazon.wirelessmetrics.service` |
| Tutorials & release-note popups | Nothing useful | `com.amazon.tmm.tutorial`, `com.amazon.storm.lightning.tutorial`, `com.amazon.tv.releasenotes` |
| Alexa / voice | Voice search and answers | `com.amazon.vizzini`, `com.amazon.vizzini.ftvcds`, `com.amazon.tv.alexaalerts`, `com.amazon.tv.alexanotifications`, `com.amazon.alexa.externalmediaplayer.fireos`, `com.amazon.alexamediaplayer.runtime.ftv`, `com.amazon.alexadirectivebrokerservice` |
| Amazon Kids | Kids profiles | `com.amazon.tahoe` |
| Photos & Music | Amazon Photos, music | `com.amazon.bueller.photos`, `com.amazon.bueller.music` |
| Silk & Games | Browser, game hub | `com.amazon.cloud9`, `com.amazon.gamehub` |
| Smart home & multi-room | Matter, Echo multi-room audio | `com.amazon.tv.matter`, `com.amazon.tv.mattercohost`, `com.amazon.whad`, `com.amazon.whasettings` |
| Screensaver | Amazon's ambient screensaver | `com.amazon.ftv.screensaver` |
| Forced updates | Forced OTA prompts (partial; see limits) | `com.amazon.tv.forcedotaupdater.v2` |

Names change between models and Fire OS versions: always confirm each one exists
(`pm list packages | grep amazon`) before offering it. Protected on 7.7 (refuse to disable):
`com.amazon.ftvads.deeplinking`, `com.amazon.device.crashmanager`, `com.amazon.device.logmanager`,
`com.amazon.device.software.ota`, `com.amazon.device.software.ota.override`.

**Never disable**: the Amazon launcher package itself (it's the fallback home and hosts system
UI pieces), `com.amazon.tv.settings*`, input methods/keyboard, Bluetooth, the Appstore (`com.amazon.venezia`:
app updates and licence checks for apps installed from it), DRM/media services, and anything
the user's apps depend on. When unsure, leave it on.

## Disable, idempotently

For each chosen package: skip if not installed (`pm path`), skip if already disabled
(`pm list packages -d <pkg>`), else `pm disable-user --user 0 <pkg>`.

Some packages are **protected** and refuse (`Cannot disable a protected package`); see the list
above. Comment them in your list as protected so the next run doesn't retry. Without root they
can't be removed.

## Settings worth changing

```bash
# Amazon's screensaver off (your launcher will provide its own idle mode)
settings put secure screensaver_enabled 0
settings put secure screensaver_activate_on_sleep 0

# Never sleep on a timer; only TV-off (HDMI-CEC standby) sleeps the stick
settings put system screen_off_timeout 2147483647
settings put secure sleep_timeout -1
settings put secure contextual_screen_off_timeout 2147483647   # Fire OS-specific
settings put global stay_on_while_plugged_in 7                  # AC|USB|wireless
settings put secure block_cec_standby 0                         # keep obeying TV-off
settings put global hdmi_control_enabled 1

# Snappier system animations
settings put global window_animation_scale 0.5
settings put global transition_animation_scale 0.5
settings put global animator_duration_scale 0.5
```

Verify with `dumpsys power | grep -E "mStayOn=|mScreenOffTimeoutSetting|mSleepTimeoutSetting"`.
**ASK** before disabling sleep (it uses a little more power) and explain that turning the TV off
still sleeps the stick via CEC. Sticks powered from the TV's USB port lose power with the TV anyway.

## Expect drift

Amazon services re-enable some packages over time (Alexa's `vizzini` is a repeat offender) and
updates reset settings. Don't fight it by hand: the heal script (phase 8) compares the list to
`pm list packages -e` and re-runs the debloat only when something came back.

## "Make Netflix faster"

What actually helps on a 1 GB stick:
- Fewer background processes (the debloat itself).
- **Your launcher using less RAM while another app runs** (phase 6: GPU trim). This was the
  single biggest measured win.
- Clearing app caches of rarely used apps; keeping >1 GB storage free.

What doesn't: `cmd package compile -m speed` on big third-party apps (measured: Netflix cold start
got *slower*, 1.25 s → 1.5 s, from the larger code). Leave other apps on `speed-profile`; compile
only your own small launcher with `speed`.
