# 9. QA: deep on-device smoke test

Don't call it done from code review. Drive the stick and look at every screen.

## Tools

```bash
adb shell input keyevent 19|20|21|22   # up / down / left / right
adb shell input keyevent 23            # OK
adb shell input keyevent 4             # Back
adb shell input keyevent 82            # Menu (☰)
adb shell input keyevent KEYCODE_HOME
adb shell screencap -p /sdcard/s.png && adb pull /sdcard/s.png shot.png
adb shell dumpsys window | grep mCurrentFocus     # which app/window is in front
```

Scripted key sequences are fragile: after Back, focus returns somewhere you may not expect, and
the next OK can launch an app. Check `mCurrentFocus` between steps, and re-centre with an explicit
"go home" intent before each scenario. Zoom into screenshots (crop and enlarge) to inspect
corners, edges and small text; at full-screen size clipping bugs are easy to miss.

## Checklist

**Boot and Home button**
- [ ] Three cold reboots land on your launcher.
- [ ] Home from inside 3 different apps returns to your home page with no visible Amazon home.
- [ ] Home while on one of your sub-pages resets to the home page.
- [ ] The escape hatch works and the way back in works.

**Every page, every sheet**, screenshot and check:
- [ ] Something is focused the moment the page opens. (A ScrollView that is focusable can take
      focus itself before layout and leave nothing highlighted: make it non-focusable.)
- [ ] The focused element is fully visible: not clipped at the top of a scroll area, not clipped
      square inside a padded sheet. Containers that host growing focus states need
      `setClipToPadding(false)` and `setClipChildren(false)`.
- [ ] Focus platters are rounded (the system default focus highlight is off).
- [ ] Left/Right at row ends don't jump to unrelated elements; Up/Down move between rows sensibly.
- [ ] Scrolling never leaves a card sliced in half under the page title: snap scroll stops to the
      gaps between rows, and fade content into the frost at the scroll area's edges (a strip of the
      same frosted glass with an alpha gradient; ScrollView's built-in fading edge bands badly on
      translucent glass).
- [ ] No text wraps to two lines in cards or footers; long app names ellipsize.
- [ ] Status lines are true (e.g. "1 of 2 on", not "On", when only one light is on).

**Readability**
- [ ] Cycle to the brightest wallpaper: clock, date, captions, pill text and card text all readable.
- [ ] Focused text (dark on white platter) and unfocused text (white on glass) both readable.

**Behaviour**
- [ ] Hold OK / ☰ on apps: add to Dock, move, remove, App Info, Uninstall all work.
- [ ] Installing or removing an app updates All Apps and the Dock without a restart.
- [ ] Screensaver appears after the delay; any key wakes it and does nothing else.
- [ ] Wallpaper crossfades smoothly; glass follows it.
- [ ] Every feature sheet opens and Back closes it, returning focus where it was.

**Performance** (phase 6): idle frame count, transition frame times, launcher memory with an app open.

**Robustness**
- [ ] Crash safety: a deliberate exception relaunches the launcher.
- [ ] `heal` repairs a deliberately broken setup; `restore` returns to stock (test on request only).
- [ ] Wi-Fi off / no internet: weather shows cached data or hides gracefully; nothing hangs.
