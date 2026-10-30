# 5. Liquid Glass design for a 10-foot screen

Goal: the calm, premium look of tvOS / iOS 26 Liquid Glass, on hardware with no live blur, readable
from the sofa on any wallpaper (including a white one).

## Layout grid (960 × 540 dp)

- Safe margins: **48 dp** left/right, **34–38 dp** top. TVs overscan less than they used to, but
  nothing important should touch the edges.
- Home: small clock top-left (time ~30 sp, date ~13 sp), status pill top-right, a single **Dock** row
  of app cards near the bottom, and a row of small captioned quick actions below it. The
  wallpaper is the star; keep the middle of the screen empty.
- App cards are **16:9**, ~112 × 63 dp in the Dock, ~120 × 68 dp in an All Apps grid of 6 columns.
  Use the app's TV banner, centre-cropped; fall back to its icon on a card tinted with the icon's
  average colour (darkened) when there's no banner.
- Settings-style pages: a frosted full-screen page, title top-left (26 sp bold), small clock
  top-right, content in a centred 4-column grid of cards (~172 × 84 dp, 14 dp gaps), with small
  uppercase letter-spaced section labels aligned to the grid's left edge.
- Sheets (options, editors, forecasts): centred, 280–620 dp wide, 22 dp corner radius, a dim shade
  behind. Back closes them.

**Don't make it huge.** First attempts usually oversize everything. TV UI at 960 dp wide with
11–16 sp body text, as above, looks right at 3 metres.

## Typography

Use the device's own UI family (Fire OS ships **Amazon Ember** / Ember Display in `/system/fonts`;
reference it by family name). Display weights for the clock and titles, regular for secondary text.
Single-line titles and subtitles with ellipsis: two-line wraps in cards always look broken.
24-hour clock by default in most of the world; make it a setting.

## The glass recipe (no live blur available)

1. **Blur once per wallpaper**, on a background thread: centre-crop to the screen's aspect, shrink
   to ~384 × 216, run a 3-pass box blur (≈ gaussian). Tiny, fast, and blurry enough.
2. Each glass surface draws that blurred image **behind its own shape**, mapped so it lines up
   with the wallpaper underneath (account for the surface's position and any wallpaper pan/zoom),
   and slightly **magnified around the surface's centre** (×1.1) so it reads as a lens.
3. **Vibrancy**: boost saturation ~1.25× with a colour matrix.
4. **Legibility veil**: sample the blurred image under the surface (a 3×3 grid is enough), compute
   luminance, and darken more on bright areas: about 8% black on dark scenery up to ~55% on a white
   sky. This is what keeps white text readable on white wallpapers.
5. A faint **accent wash** of the wallpaper's own signature colour (saturation-weighted average of
   the blurred image) so glass belongs to the scene.
6. **Specular rim**: a 1 dp stroke with a diagonal gradient (bright top-left, almost clear middle,
   soft catch-light bottom-right), plus a soft white **sheen** fading down from the top half.
7. **Do steps 2–5 in a single GPU pass** (see phase 6): each overlay is a solid colour at fixed
   alpha, so the whole stack collapses into one colour matrix applied to the bitmap shader.

"Material" glass for surfaces on top of an already-frosted page skips the wallpaper and uses a
translucent white tint (~10%) + rim. Sheets use a nearly opaque dark tint.

Keep **app icons and banners in their real colours**: glass is for chrome, not content.

## Focus (the most important state on TV)

- tvOS-style: the focused control **scales up** (~1.07 for buttons, ~1.10–1.14 for app cards),
  lifts with a soft shadow, and glass controls fade in a **near-white platter** (~95% white) while
  their text and icon animate to dark ink. Unmistakable from across the room.
- App cards: tilt in from the direction focus arrived (rotationX/Y ~7–9°, spring back flat), a
  light spot sliding opposite the tilt, a one-off glint sweep, an accent-coloured shadow
  (`setOutlineSpotShadowColor`, API 28). Let it **sway gently a couple of times, then rest**:
  continuous motion costs GPU forever (phase 6).
- Labels under Dock cards appear only on focus; in All Apps they're always visible.
- Disable Android TV's own **default focus highlight** on every focusable custom view
  (`setDefaultFocusHighlightEnabled(false)`, API 26), or it paints a square grey overlay on
  top of your rounded platter.
- Left on the first item / Right on the last item of a row should stay put: Android's focus search
  otherwise jumps diagonally to whatever is furthest that way (e.g. a pill in the top corner).

## Readability, everywhere

- Text over the photo gets a soft shadow (≈10 dp blur, 50% black). Text on frosted pages doesn't need it.
- Darken the top-left (clock) and bottom (Dock) of each wallpaper with gradients whose strength
  follows that region's brightness. Bake them into the wallpaper bitmap (phase 6).
- Test on the brightest wallpaper you can find (snow, white sky). If any text is hard to read
  there, the veil or scrim is too weak.

## Motion language

Consistent depth metaphor, all 200–380 ms, ease-out curves (and a light overshoot for springs):
- Opening a page: Home **recedes** (scale 0.92, fade, lift 1.5%); the new page **arrives** from
  slightly larger (1.06 → 1.0, fade in).
- Closing: the page dismisses forward (→ 1.05, fade) while Home returns from depth.
- Content **cascades** in: items rise ~14–18 dp and fade, staggered 14–30 ms, capped so the
  last starts within ~360 ms. Only cascade what's on screen.
- Pressing: a quick 0.94 "press" pop before an app opens; the system scale-up animation from the card.
- Returning from an app: the wallpaper settles from 1.04 → 1.0 and the Dock glides up.
- Wallpaper: slow Ken Burns pan/zoom (1.02 → 1.10 over the display time) and an ~1.8 s crossfade
  every 30 s–15 min (user setting). Glass tints crossfade with it.

## Status and extras chrome

A **status pill** top-right (glass, 36 dp tall): weather icon + temperature, and other small
facts you can actually read on the stick. A **Now Playing pill** top-centre that only exists while
media plays. Both focusable, both open sheets. Keep pills compact: one line, no clutter.

## Screensaver

Your own, inside the launcher: after N idle minutes on the home page, fade the controls out and
show a large clock (~72 sp) bottom-left over faster-cycling wallpapers. Any key wakes it and that
key does nothing else. Turn Amazon's screensaver off so it never takes over.
