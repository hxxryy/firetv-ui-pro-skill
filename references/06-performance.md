# 6. Performance on weak TV hardware

A Fire TV Stick has a quad Cortex-A35/A53, a small PowerVR/Mali GPU and often 1 GB of RAM shared
with the GPU. Fill rate (pixels drawn per frame) and memory are the limits, not Java speed.
Everything below was measured; numbers are from a 3rd-gen stick.

## Measure first, every time

```bash
adb shell dumpsys gfxinfo <pkg> reset          # start a window
# … do the interaction …
adb shell dumpsys gfxinfo <pkg>                # frames, janky %, percentiles, GPU caches
adb shell setprop debug.hwui.profile true      # then restart the app: per-frame table of
                                               # Draw / Prepare / Process / Execute (ms)
adb shell dumpsys meminfo <pkg>                # "Graphics" / "GL mtrack" = GPU memory
adb shell dumpsys meminfo | sed -n '/Total PSS by process/,/OOM/p'   # who uses RAM
```

How to read it:
- **Idle frame count** matters most: 6 s on the home screen doing nothing should not be 360 frames.
- In the per-frame table, *Draw* = UI-thread recording, *Process* = render-thread work issuing GPU
  commands. Over 16.7 ms total = a dropped frame. High Process → too much drawing per frame.
  High Draw → something re-records views every frame.
- Fire OS flags nearly every frame as "high input latency", which inflates gfxinfo's janky %.
  Trust the per-stage table more.
- `framestats` may be empty on Fire OS; `am profile` needs a debuggable build (add aapt2's
  `--debug-mode` temporarily, profile with `am profile start --sampling 250 <pkg> <file>`, parse
  the trace, then reinstall the release build). Debug builds exaggerate UI-thread times.

**A/B switches on the device**: to find what costs what without rebuilding each time, have the
app read a flags file (e.g. `/sdcard/exp.txt`) at start and skip individual effects (shadows,
rims, glass, scrims, wallpaper) when flagged. Restart the app between runs and compare the
per-stage averages. Remove the switches afterwards. On the test stick every layer cost ~0.5–1.5 ms
and the full design ~11 ms of the 16.7 ms budget: no single culprit, so each fix below matters.

## The fixes, in order of impact

### 1. Release GPU memory when an app is in front (biggest win for other apps)
An accessibility service bound to your process keeps it at foreground priority, so Android
**never sends `TRIM_MEMORY_UI_HIDDEN`**, and the renderer kept ~65 MB of purgeable GPU cache (91 MB
"Graphics") while Netflix ran: more than Netflix itself. Fix: in `onStop` (after a short delay, and
only if still stopped and the snapshot overlay isn't showing) ask the window manager to trim:
reflectively call `WindowManagerGlobal.getInstance().trimMemory(TRIM_MEMORY_COMPLETE)` (a
greylisted hidden API on API 28; wrap it in try/catch). Also do it from `onTrimMemory(level >= BACKGROUND)`.
Result: Graphics 91 MB → 0.1 MB while another app runs. Display lists rebuild in a frame or two on return.

### 2. Nothing animates forever
- A wallpaper Ken Burns animation driven by an `ObjectAnimator` renders 60 fps forever, for motion
  of ~1 px per second. **Step it ~12 times a second** with a Handler (bilinear filtering hides the
  steps), and re-align glass on the same tick. Idle frames: 367 → 75 per 6 s.
- **Pause it whenever the wallpaper is hidden** (frosted pages are opaque: set the wallpaper view
  INVISIBLE once a page has fully arrived) and during page transitions.
- Card sway/breathing effects: run a couple of cycles after focus, then stop.
- Result: other pages render **0 frames** when idle.

### 3. One GPU pass per glass surface
Wallpaper + vibrancy + veil + accent + tint drawn as separate round-rects = 4–6 full passes per
surface, and a full-screen frosted page is a full-screen pass each. Since every overlay is a solid
colour at constant alpha, fold them into a single `ColorMatrix` (saturation, then scale by the
product of `(1 - alpha_i)` and add the colour offsets) on the bitmap-shader paint. Cache the filter
and rebuild only when the (quantised) veil or accent changes.

### 4. No offscreen layers for fades
`withLayer()` and alpha on views that report overlapping rendering make the renderer draw into an
offscreen buffer and composite it: for full-screen pages that's two extra full-screen passes per
frame, and a cascade of 30 cards fading in meant 30 offscreen buffers. Call
`forceHasOverlappingRendering(false)` on pages, cards, buttons and fading containers so alpha is
applied per draw. The visual difference during a 300 ms fade is invisible.

### 5. Immutable bitmaps
On Android 9 a **mutable** bitmap drawn by a view is **copied every time that view re-records**.
The wallpaper (8 MB), the blurred backdrop (once per glass surface per re-record) and pre-scaled
banners were all mutable. When you finish drawing into a bitmap, make an immutable copy
(`bitmap.copy(config, false)`) and recycle the original.

### 6. Bake static overlays into the wallpaper
The dark gradients behind the clock and Dock are static per wallpaper but cost almost a full
screen of fill every frame as views. Draw them into the decoded wallpaper bitmap on the
background thread (computing their strength from the blurred image's brightness first).

### 7. Don't draw what nobody can see
- Drop resting shadows (elevation) on dark frosted pages: invisible there, but 30 shadow meshes a frame.
- Drop text shadows on text that sits on frost.
- Pre-scale app banners to their display size once (some apps ship 1280×720 banners), with mipmaps.
- **Never move a view whose alpha is 0** (e.g. a light spot before it fades in): it forces its
  whole parent chain to re-record each frame. Skip the update while it's invisible.
- Cascade only the rows on screen.

### 8. Expensive one-offs at the wrong time
`PixelCopy` snapshots are a full-screen GPU readback plus an 8 MB allocation. Taken 700 ms after
every focus change, they landed in the middle of transitions. Take them at quiet moments
(phase 4).

### 9. Sheets and pop-ups (lessons from a music player)
- **Hide what a big sheet covers.** Behind a large modal, draw one frosted, darkened copy of the
  wallpaper and stop drawing Home and the photo once the sheet has faded in; also cut the sheet's
  area out of that backdrop (`clipOutRect`, inset by the corner radius) once it has settled.
- **Pause ambient motion** (wallpaper drift) while a sheet is open.
- **Don't clip a whole sheet to its rounded outline** (`setClipToOutline` on a big container makes
  every child draw through a curved clip); draw only the background with rounded corners.
- **Progress bars**: set the level directly on each tick (no 200 ms glide every second, which meant
  ~13 redraws a second), and skip sub-pixel changes.
- **Self-sizing labels whose text changes every second** (elapsed / remaining time) trigger a
  measure and layout of the *whole window* each time. Give them a fixed size.
- **Animated shadows** (translationZ on focus) are recomputed every frame; on dark sheets they're
  invisible anyway. Keep the grow animation, drop the lift.
- **Colour emoji are re-rasterised at every new scale** on Android 9: a growing focus animation
  over an emoji label re-draws it every frame. Render emoji labels to a bitmap once.
- **Shelves (HorizontalScrollView)**: if the GPU can't hold 60 fps while a whole row glides, jump
  (`setSmoothScrollingEnabled(false)`); a crisp jump feels faster than a juddering glide.
- **Pre-bake filters**: darken a blurred cover once per song rather than applying a colour filter
  on every frame.
- Skip setters when nothing changed: `setText` with the same text still requests layout.
- Typical result: a music sheet sitting open went from 106 to 6 frames per 6 s; moving around it
  from 30–35 ms to 18–22 ms a frame.

### 10. Your launcher, AOT
`cmd package compile -m speed -f <your.package>` after each install. (Not for big apps: see phase 2.)

## Results on the test stick

| Metric | Before | After |
| --- | --- | --- |
| Idle Home, frames per 6 s | 367 | 75 (other pages: 0) |
| Launcher GPU memory while Netflix runs | 91 MB | 0.1 MB |
| Launcher total PSS while Netflix runs | 166 MB | ~16 MB resident |
| Home frame cost (render thread) | ~14.5 ms | ~11 ms |
| Page transition, 90th percentile frame | 44 ms | ~23–30 ms |

Transitions on this GPU sit close to the 16.7 ms budget; getting every frame under it would mean
visibly cutting the design. Say so honestly rather than over-promising "60 fps".
