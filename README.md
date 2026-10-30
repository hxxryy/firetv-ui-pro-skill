# Fire TV UI Pro: a Claude skill

Teach Claude (or yourself) how to turn an Amazon Fire TV Stick into a fast, ad-free, Apple TV-style
device with a custom **Liquid Glass** home screen, and keep it that way after Amazon updates.

It's a field guide written from a real build on a Fire TV Stick 3rd gen (Fire OS 7.7, 1 GB RAM):
every limit was tested and every performance number measured on the device.

## What it covers

1. **Connect**: find the stick on Wi-Fi, ADB over the network, survive IP changes
2. **Debloat**: ads, tracking, Alexa, Kids, Photos, Silk, smart home, screensaver, sleep timer, with
   verified package names, what never to touch, and how to undo it
3. **Build**: a launcher APK with the SDK command-line tools only (no Gradle, no Android Studio)
4. **Take over Home**: boot into your launcher and get the remote's Home button to come back to it
   on Fire OS, which ignores default launchers and swallows the Home key
5. **Design**: an iOS 26-style Liquid Glass system for a 10-foot screen, readable on any wallpaper
6. **Speed**: measured fixes for weak TV GPUs and 1 GB of RAM (idle redraws 367 → 75 per 6 s,
   launcher GPU memory while Netflix runs 91 MB → 0.1 MB)
7. **Extras**: weather, Now Playing, Philips Hue Bluetooth bulbs without a Bridge, sunrise wake-ups
8. **Keep it**: self-healing, backups, restore to stock
9. **QA**: an on-device smoke-test checklist
10. **Limits & troubleshooting**

## Install

Claude Code, for all your projects:

```bash
git clone https://github.com/hxxryy/firetv-ui-pro-skill.git ~/.claude/skills/firetv-ui-pro
```

Or just for one project: clone it into that project's `.claude/skills/firetv-ui-pro`.

Then ask Claude something like *"debloat my Fire TV Stick and give it an Apple TV-style home
screen"*. The skill loads automatically. You can also just read the files in `references/`: they're
written for people too.

## What you need

- A Fire TV Stick or Fire TV device (Fire OS 6–8). Non-Android smart TVs (TiVo OS, Tizen, webOS)
  can't be customised this way.
- A Mac or PC on the same network, with Android platform tools (ADB).
- ADB debugging switched on in the stick's Developer Options.

## Safety

Everything here is reversible: packages are disabled, never uninstalled; nothing is rooted or
flashed; backups come first; a restore path is part of the plan. Claude is instructed to ask before
disabling anything. You use it at your own risk; it's not affiliated with or endorsed by Amazon,
Apple or Signify.

## License

MIT. See [LICENSE](LICENSE).
