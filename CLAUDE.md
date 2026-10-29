# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repo is

A kit for debloating Android TV / Google TV / Fire TV sets over ADB and replacing the home screen with a
small native launcher (`com.gathin.tvhome`). Three parts:

- `PROMPT.md`: the prompt users paste into a coding assistant. Most users only ever touch this. Keep it
  brand-neutral; brand-specific knowledge goes in `examples/`. A fix for an assistant getting stuck usually
  belongs here as one extra sentence, not in a script.
- `tools/`: a pure-Python ADB client (`tv.py`, built on `adb-shell`) used instead of the `adb` binary, which
  recent macOS blocks under Local Network privacy. Plus `b64push.py` (APK upload as base64 chunks + md5,
  because `adb-shell` push hangs on large files) and `disable.sh` / `restore.sh` that apply/undo a debloat list.
- `launcher/`: the Android launcher app (plain Java, Gradle).

`examples/<brand>-<model>/debloat.txt` holds per-TV package lists: one package per line, grouped under
comment headers, trailing `# ...` comments allowed (the shell scripts strip them). Header must state exact
model, Android version and firmware (`tools/tv.py --probe`). New examples also get a row in `examples/README.md`.

## Commands

There is no test suite or linter. Verification is done on a real TV: inject key presses and pull a screenshot.

```bash
# Python tools setup (venv path is hard-coded in the scripts)
python3 -m venv tools/venv && tools/venv/bin/pip install -r tools/requirements.txt
export TV_HOST=<tv ip>                                    # TV_PORT defaults to 5555
tools/venv/bin/python -I tools/tv.py --probe
tools/venv/bin/python -I tools/tv.py '<shell cmd>'
printf 'cmd1\ncmd2\n' | tools/venv/bin/python -I tools/tv.py --batch
tools/venv/bin/python -I tools/tv.py 'input keyevent KEYCODE_DPAD_DOWN; screencap -p /sdcard/s.png'
tools/venv/bin/python -I tools/tv.py --pull /sdcard/s.png s.png

# Build only
cd launcher && ./gradlew assembleDebug        # -> app/build/outputs/apk/debug/app-debug.apk

# Build + install + make it home (detects Fire TV via ro.product.manufacturer == Amazon)
TV_HOST=<ip> launcher/build-and-install.sh    # STOCK_LAUNCHER=<pkg> overrides the Android TV launcher to disable
```

Build needs JDK 17 and Android SDK platform 34 + build-tools 34.0.0. `build-and-install.sh` defaults
`JAVA_HOME`/`ANDROID_HOME` to Homebrew paths, so on Linux/WSL export both first. It writes
`launcher/local.properties` (`sdk.dir`) only if the file is missing. `local.properties` is gitignored and may
also hold `tvhome.userName=<name>` for the greeting (injected as the `user_name` string resource in
`app/build.gradle`).

Dock icons for YouTube/Netflix/Prime/AirPlay are trademarked and gitignored; generate them locally with
`tools/make_dock_icons.py` (see its docstring; needs pillow + cairosvg). Without them, apps fall back to their
own icon/banner, so the build works either way.

## ADB gotchas that affect how you script the TV

- Run `pm` commands one per shell call; chained `pm a; pm b` may silently stop after the first. `--batch`
  sends each stdin line as a separate shell call on one connection.
- Disable packages with `pm disable-user --user 0`, never uninstall. Everything must stay reversible.
- Install the new launcher before disabling the stock one, or Home lands on a factory test app.
- On Fire TV, never disable Amazon's launcher (`com.amazon.tv.launcher`): it hosts Settings. ADB debugging
  turns off on every TV restart. Some packages are protected and refuse to disable.

## Launcher architecture

All code is in `launcher/app/src/main/java/com/gathin/tvhome/`:

- `MainActivity`: the HOME activity. Top bar (greeting/clock/date, Inputs/Wi-Fi/Settings pills), time-of-day
  background, the dock and app grid in one RecyclerView, hero label and ambient colour for the focused dock
  app, the Inputs popup (TV input framework: HDMI, AV, Live TV), and explicit D-pad key handling.
- `AppAdapter`: builds the tile list (pinned dock apps first, then the rest alphabetically, minus
  `hidden_apps`). Pin/unpin/reorder via hold-OK or Menu; the dock order is saved in SharedPreferences
  (`tvhome` / `dock`) and only seeded from `config.xml`'s `dock_apps` on first run. Also maps packages to the
  optional `dock_*` drawables and samples a vivid colour from banners.
- `DockDecoration` / `GlassView`: dock band drawing and the single "glass" focus highlight that springs
  between tiles (AndroidX DynamicAnimation), instead of per-tile focus borders.
- `PhotoDream`: DreamService screensaver (bundled `saver_*.jpg`, slow zoom, crossfade, clock).
- `HomeRedirectService`: Fire TV only. Fire OS ignores the default-home setting and doesn't pass Home to
  accessibility services, so this AccessibilityService watches window changes and opens `MainActivity` when
  Amazon's `HomeActivity` appears, except within a grace window after an Amazon settings window (that home
  flash is part of entering Settings). Toggled from the launcher (Menu on the Settings pill) using
  `WRITE_SECURE_SETTINGS`, which `build-and-install.sh` grants over ADB.

TV-specific values (starting dock, hidden apps, Live TV package, column count) belong in
`app/src/main/res/values/config.xml`, not in code.

## Constraints for launcher changes (from CONTRIBUTING.md)

- Targets TVs with ~1 GB RAM: no blur, no video, no network polling, no extra background services.
- Plain Java + AndroidX RecyclerView/DynamicAnimation only. No Kotlin, no Compose, no new dependencies without
  discussion. No recommendation rows, feeds, ads or telemetry.
- D-pad navigation must stay deterministic: Up/Down move exactly one row, Up from the dock goes to the top bar,
  the ends of the top bar don't fall through. Test every direction from every edge, on a real TV.
