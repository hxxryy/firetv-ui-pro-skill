# 3. Build a launcher APK without Gradle

Gradle and Android Studio are heavy and slow for a single-module, dependency-free TV app. The
Android SDK's own command-line tools build an APK in about 20 seconds.

## Tools

macOS (Homebrew):

```bash
brew install --cask android-platform-tools android-commandlinetools microsoft-openjdk@11
sdkmanager "build-tools;34.0.0" "platforms;android-28"
```

Pick the **platform matching the stick's API level** (Fire OS 7 = API 28) as the compile target,
and keep `targetSdkVersion` at that level too: higher targets add background and permission
restrictions Fire OS doesn't need, and hidden-API rules get stricter.

## Project layout

```
launcher/
  app/AndroidManifest.xml
  app/res/…            (icon, TV banner 320x180, styles, accessibility config XML)
  app/assets/…         (wallpapers etc.)
  app/src/<package>/*.java
  keystore/            (signing key: keep it forever)
  build.sh
```

No support libraries: plain `android.*` APIs keep the APK tiny (~3 MB, mostly wallpapers) and
the build trivial. Build the UI in code rather than XML layouts if you like; both work.

## The pipeline (what build.sh does, in order)

1. **aapt2 compile** the `res/` directory into a flat archive.
2. **aapt2 link** it with the manifest, assets and `android.jar` → a base APK plus a generated
   `R.java`. Pass `--min-sdk-version` / `--target-sdk-version` here. A `--debug-mode` switch
   (behind an environment variable) is handy: it marks the app debuggable for profiling (phase 6).
3. **javac** all sources plus `R.java` against `android.jar` as the boot classpath, `-source 8 -target 8`.
   Lambdas need `core-lambda-stubs.jar` (ships in build-tools) on the classpath.
4. **d8** the `.class` files into `classes.dex` (`--release --min-api <n>`).
5. Add `classes.dex` to the APK (`zip -j`).
6. **zipalign** `-p 4`.
7. **apksigner sign** with your keystore. Create the keystore once with `keytool -genkeypair`.

## Signing key: the one thing you must not lose

Android only installs an update over an existing app if it's signed with the **same key**. Lose
the keystore and the only way forward is uninstalling (losing the app's settings). Back it up
with every snapshot and keep it out of public repositories.

## Install

```bash
adb -s $S install -r -g app.apk    # -r replace, -g grant runtime permissions
```

Then the extra grants a launcher needs (phase 4): `WRITE_SECURE_SETTINGS`, enabling the
accessibility service and notification listener. Finish with
`cmd package compile -m speed -f <your.package>` so your launcher runs fully AOT-compiled.

## Gotchas met on Java 8 / API 28

- `Activity` already has a final `isResumed()`: a method of your own with that name won't compile.
- Field initialisers run before the constructor body: don't use a field set in the constructor
  inside another field's initialiser (it will be `null`).
- `View.forceHasOverlappingRendering`, `setDefaultFocusHighlightEnabled`, `PixelCopy` need API
  24/26; guard with `Build.VERSION.SDK_INT` if you support older sticks.
- Re-adding a View that already has a parent throws "The specified child already has a parent";
  detach it first when rebuilding lists. Install a default uncaught-exception handler that
  relaunches your launcher after a crash, so a bug never strands the user on Amazon's home.
