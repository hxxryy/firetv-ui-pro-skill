# 1. Connect: find the stick and talk to it over ADB

## On the TV (the user does this)

1. Settings → My Fire TV → About → click the device name 7 times to unlock Developer Options
   (newer Fire OS). Older builds show Developer Options already.
2. Settings → My Fire TV → Developer Options → **ADB debugging: ON**.
   (Apps from Unknown Sources is not needed for ADB installs.)
3. The computer must be on the **same network** as the stick. Guest Wi-Fi and mesh "IoT"
   networks often isolate devices; if nothing is found, this is the usual reason.

## Find it

- Settings → My Fire TV → About → Network shows the IP. Easiest if the user can read it out.
- Otherwise scan the subnet for the ADB port (5555) and confirm the model:
  ```bash
  NET=$(ipconfig getifaddr en0)            # macOS; on Linux: hostname -I
  for i in $(seq 2 254); do (nc -z -G 1 ${NET%.*}.$i 5555 2>/dev/null && echo ${NET%.*}.$i) & done; wait
  ```
  Fire TV models report `ro.product.model` starting with `AFT` (e.g. `AFTSS`, `AFTMM`, `AFTKA`).
- **Smart TVs with TiVo OS, Tizen, webOS, VIDAA etc. are not Android**: no ADB, no launcher swap.
  Say so plainly and suggest a Fire TV Stick or Google TV dongle instead.

## Connect

```bash
adb connect <ip>:5555
adb -s <ip>:5555 shell getprop ro.product.model
```

The TV shows **"Allow USB debugging?"**: the user must tick "Always allow" and press OK.
Until then you'll see `unauthorized` or `failed to authenticate`.

## Record device facts before changing anything

```bash
adb -s $S shell getprop ro.build.version.name     # Fire OS version
adb -s $S shell getprop ro.build.version.sdk      # Android API level (28 = Android 9)
adb -s $S shell wm size; adb -s $S shell wm density   # usually 1920x1080 @ 320 → 960x540 dp
adb -s $S shell cat /proc/meminfo | head -3        # 1 GB sticks need extra care (phase 6)
adb -s $S shell pm list packages -s > stock-system-packages.txt
adb -s $S shell pm list packages -3 > user-apps.txt
```

Design for the logical size: at density 320 the screen is **960 × 540 dp**. Every layout number in
phase 5 assumes that.

## Make scripts survive an IP change

Routers reassign addresses, and a new router means a new subnet. Put the stick's IP in one shared
config file, and have every script fall back to the subnet scan above (checking the model starts
with `AFT`) when the stored IP doesn't answer, then rewrite the config with the new address.
Alternatively, ask the user to give the stick a DHCP reservation in their router.

## Shell gotchas

- In zsh, `A="adb -s x shell"; $A ls` fails (no word splitting). Use a function: `a(){ adb -s $S shell "$@"; }`.
- `adb shell` inside a `while read` loop eats the loop's stdin. Read the list on another file
  descriptor (`while read -r p <&3; do …; done 3< list.txt`) or pass `</dev/null` to adb.
- Strip `\r` from `adb shell` output before comparing strings (`| tr -d '\r'`).
