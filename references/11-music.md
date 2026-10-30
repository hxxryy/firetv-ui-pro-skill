# 11. Music: Apple Music and Spotify from the home screen

Goal: browse playlists, play, shuffle, skip and change speaker volume from the launcher, with the
remote, without ever touching the music app's own (phone-style) screens.

## Where it lives in the UI

Don't build a separate "app". Put small glass buttons next to the weather pill:

- **♪ Music**: opens a large music sheet. While something plays, show the cover art inside the
  button instead of the note icon.
- **🔊 Speaker**: appears only while something plays; opens a small sound sheet (volume, mute,
  Bluetooth speakers). Show the volume % in it when the stick controls the volume.

The music sheet, top to bottom: source switch (Apple Music | Spotify, only installed ones,
remembered), now playing (big cover, song, artist · album, progress, ⤮ ⏮ ⏯ ⏭ ↻ ♥), an **Up Next**
shelf of covers, and a **Playlists** shelf (the user's playlists, the app's picks, Search). Use the
current cover, blurred and darkened, as the sheet's background: it changes with every song and looks
like iOS's Now Playing. Back must return focus to the button that opened the sheet.

Make the screensaver follow along: while music plays, show the cover, song, artist and progress
(and move the clock up). Let the remote's media keys control playback without waking it.

## Controlling any music app (the media-session route)

Everything except listing/starting the user's own library goes through Android's media APIs:

- Read sessions with `MediaSessionManager.getActiveSessions(listenerComponent)`; that requires a
  (do-nothing) `NotificationListenerService` enabled via
  `cmd notification allow_listener <pkg>/<pkg>.<Listener>`.
- **Wake the app without opening it**: connect a `MediaBrowser` to the app's
  `android.media.browse.MediaBrowserService` (find it with `queryIntentServices`). Even if the app
  refuses to share its library, the connection starts its playback service and
  `browser.getSessionToken()` gives you a `MediaController`.
- Transport: `play/pause/skipToNext/skipToPrevious/skipToQueueItem`.
- **Shuffle / repeat / love** are usually the app's *custom actions*
  (`PlaybackState.getCustomActions()`). The on/off state isn't reported directly, but the icon
  the app publishes for each action changes: read its resource *name* via
  `getResourcesForApplication(pkg).getResourceEntryName(icon)` (e.g. `…shuffle_on` / `…shuffle_off`).
  Cache the `Resources` and the id→state map.
- **Up Next**: `MediaController.getQueue()` with `getActiveQueueItemId()`; covers come as bitmaps or
  URIs (`content://` or `https://`): decode at display size, cache by bytes.
- **Play by name** (catalogue playlists, artists, songs): `playFromSearch(query, extras)`; for a
  playlist add `MediaStore.EXTRA_MEDIA_FOCUS = Playlists.ENTRY_CONTENT_TYPE` and
  `EXTRA_MEDIA_PLAYLIST = name`. Good for "Today's Hits"-style picks and free-text search. **Not**
  reliable for the user's own playlists: it searches the whole catalogue, and names like "🌴" match
  nothing in particular.
- **Cache metadata from the callbacks** (`onMetadataChanged`, `onPlaybackStateChanged`,
  `onQueueChanged`). Every `getMetadata()` is a fresh binder copy *including the album-art bitmap*.
- Compare tracks by a key (media id + title + artist), never by bitmap identity: each metadata copy
  carries a new bitmap object.

## Apple Music specifics

- **No official Fire TV app in the UK.** Third-party "Apple Music" apps in the Appstore are mostly
  web-player wrappers: no media session (nothing for your player to read) and no background
  playback (Home stops the music). Test any of them in a minute: install, play, then check
  `dumpsys media_session` and `dumpsys audio` (players with `usage=USAGE_MEDIA`).
- **The real Android app works on a Fire TV Stick** when sideloaded. On a 32-bit, Android 9 stick:
  Apple Music 5.0+ needs Android 10, so use **4.9.7.2** (Android 7.0+). The user downloads it
  (e.g. APKMirror); don't fetch it yourself from unofficial sources. Verify before installing:
  `apksigner verify --print-certs` (signer "Apple Inc", Cupertino, certificate from 2015; the
  bundle carries a Google Play source stamp). From a bundle (.apkm = zip of splits), install only
  `base.apk` + the `armeabi_v7a` split + the density split (`xhdpi` at 320 dpi) with
  `adb install-multiple`.
- **Signing in** with a remote is painful: the privacy screen's focus sits on a link, the code
  screen misbehaves. Help with ADB taps to reach the fields, and offer a **hidden-input typing
  helper** (`read -s` on the computer, then `adb shell input text`, escaping spaces as `%s` and
  other symbols with backslashes) so the user types their own credentials. Never type a user's
  password yourself. Apple's documented fallback for awkward devices: **password followed directly
  by the 6-digit code** in the password box.
- **Apple refuses to share the library**: its browse service returns `__EMPTY_ROOT__` to any caller
  it doesn't recognise (Android Auto, Wear etc. only), whatever root hints you send.

## The accessibility "agent" (listing and starting the user's own playlists)

When the app won't cooperate, the launcher's accessibility service (already present for the Home
button) can operate the app's screens for the user. Done carefully, it's fast and invisible:

1. **Cover first.** Take a `PixelCopy` frame of the launcher (with the sheet open) and show it as a
   full-screen `TYPE_ACCESSIBILITY_OVERLAY` with a small status card ("Shuffling 🌴…"). Make it
   `FLAG_NOT_FOCUSABLE | FLAG_NOT_TOUCHABLE` so the app underneath stays active and receives taps.
   Filter remote keys while it runs.
2. **Launch** the app's launch intent, then **poll** `getRootInActiveWindow()` every ~350 ms with
   an overall timeout (20 s). Navigate by what's on screen, never by coordinates: if the playlists
   list is visible, go on; else if the PLAYLISTS tab is visible, click it; else if the Library
   tab (by view id) is visible, click it; else press global Back (up to a few times).
3. **Close pop-ups before every step.** Apple Music shows sheets like "Music + Friends" (a
   `dialog_view` with a `dismiss` button) that swallow taps.
4. **Read the list** from the title nodes, after **scrolling to the top** (the app remembers its
   scroll position), then scroll forward until nothing new appears. **Ignore nodes inside the
   player panel**: the expanded player also has a "title" (the current song).
5. **Start one**: find the title node equal to the name, click its clickable ancestor, then press
   the page's own Play/Shuffle button. Two traps: a *content description* "Play" also matches the
   mini player's button (skip ids containing `mini_player` / `play_pause`), and Apple's play buttons
   **ignore `ACTION_CLICK`**: send a real tap with `dispatchGesture` at the node's centre (requires
   `canPerformGestures="true"`). Apple may then ask **"keep playing or clear the queued songs?"**:
   answer *Clear* (the user asked for that playlist).
6. **Verify**: the mini player's title must change within a few seconds; if not, retry once.
7. **Return** to the launcher (no entrance animation; reopen the sheet focused on the tile that was
   used), then remove the cover.

Plumbing that matters:
- `canRetrieveWindowContent` / `canPerformGestures` are XML capabilities. Android only re-reads them
  when the service is switched on, so the install script must **toggle the service off and on**
  after an update (check `dumpsys accessibility` capabilities).
- Switch `flagReportViewIds | flagIncludeNotImportantViews | flagRetrieveInteractiveWindows` on with
  `setServiceInfo` **only while the agent runs**; they add overhead to every app otherwise.
- Say in the service description that it operates the music app's screens when asked.
- Log each step (`log` tag) so a changed app layout is easy to diagnose; show "Couldn't start …"
  rather than hanging.

## Spotify

Spotify's TV app has a browse service too and may share playlists through it: try the root's
children (and a "playlists/library" folder) and offer them as tiles (`playFromMediaId`). Keep an
"Open Spotify" button: its TV app is remote-friendly. Let users add playlist names as a fallback.

## Speaker volume and Bluetooth speakers

- Over HDMI, Fire OS pins the stick's music volume at 100% (the TV owns it; `isVolumeFixed()` or
  100%): say "Set on TV" instead of offering a dead slider.
- Through a **Bluetooth speaker** the stick's `STREAM_MUSIC` volume *is* the speaker's (absolute
  volume): `adjustStreamVolume(… ADJUST_RAISE/LOWER/TOGGLE_MUTE)` works, and gives remotes without
  volume keys a volume control.
- List paired audio devices (major class AUDIO_VIDEO or audio/render service, or anything the A2DP
  proxy reports connected). `BluetoothA2dp.connect/disconnect(device)` are hidden on Android 9 but
  callable by reflection with `BLUETOOTH_ADMIN`; fall back to opening Fire TV's Bluetooth settings.
  Note Hue bulbs and the remote are paired LE devices: don't mistake them for speakers.
