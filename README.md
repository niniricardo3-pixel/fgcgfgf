# FortniteHub

Launcher + tools built around the OGFN Manager 2.6.4 base (`ogfn-by-dhyper_1.apk`, package `com.dhyper.ogfnmanager`).
Status: nothing here has been built or run on a device. The Python tools and their tests were run (17/17 pass); the patcher pipeline ran end to end on a sample APK.

## What is grounded (read from real artifacts on 2026-11-22)
| Fact | Source |
|---|---|
| Base app = Compose manager that downloads Epic CDN builds, runs a bundled Node + LawinServer in a `:backend` process | base APK manifest, dex strings, `libnode.so`, `assets/LawinServer-main` |
| Start backend: exported transparent activity `com.dhyper.ogfnmanager.ACTION_TRIGGER_BACKEND` calls `BackendManager.start()` and finishes; refuses if onboarding is incomplete | dex disassembly |
| Stop backend: broadcast `com.dhyper.ogfnmanager.STOP_BACKEND` | manifest + dex |
| LawinServer listens on `process.env.PORT \|\| 8080`; probe route `/fortnite/api/version` | `index.js`, `structure/version.js` |
| Matchmaker hands the game `[GameServer] ip/port` from `Config/config.ini` (default 127.0.0.1:7777) | `matchmaking.js`, `config.ini` |
| Uptodown lists 376 Fortnite APKs (1.0.0 … 43.10.0-60118918), 90–330 MB each | live site |
| Uptodown downloads sit behind Cloudflare Turnstile | download page loads `challenges.cloudflare.com/turnstile` |
| Reboot Launcher release asset: one `reboot_launcher-<tag>-windows-setup.exe`, latest 10.0.9 | GitHub API |

## Not known (so not invented)
- The port the manager's NodeRunner really sets (app setting, default 8080).
- Any per-build patch point inside a Fortnite APK (main activity, Epic host strings, TLS handling). No Fortnite APK has been inspected yet.
- Reboot Launcher installer silent switches; its game-server port (read it from the launcher's host tab).
- Whether a given Uptodown build accepts this backend. Mark each build tested/untested yourself.

## Steps
1. Catalog (needs your browser, Uptodown blocks scripts):
   open https://fortnite.fr.uptodown.com/android/versions, paste `tools/catalog_console.js` in the console, then
   `python tools/uptodown_catalog.py normalize fortnite_catalog.raw.json catalog.json`
2. APKs (you press the download button, Turnstile is yours to pass):
   `python tools/download_queue.py catalog.json --downloads "%USERPROFILE%\Downloads" --out apks --oldest-first --major-max 10`
   (drop `--major-max` for all 376, about 60 to 90 GB by the sizes seen)
3. PC host: `host\Install-RebootHost.ps1` (Reboot Launcher), `-CheckRdp` / `-EnableRdp`, `-SetupNoVnc`.
4. App: copy `catalog.json` to `launcher-app/app/src/main/assets/`, then
   `cd launcher-app && gradle wrapper --gradle-version 8.7 && ./gradlew assembleDebug testDebugUnitTest`
   Versions pinned: AGP 8.5.2, Kotlin 1.9.24, Compose compiler 1.5.14, Compose BOM 2024.06.00, Coil 2.6.0.
5. On the phone: install OGFN Manager and finish its onboarding once, install the Fortnite build, open Fortnite Hub, tap a version, "Play offline".

## Per-APK patching (in-game "Offline" button)
`tools/patch_apk.py` (one APK) and `tools/patch_all.py` (every APK in `apks/index.json`) need Java and these two jars from GitHub:
apktool 3.0.3 (`apktool_3.0.3.jar`) and uber-apk-signer 1.3.0 (`uber-apk-signer-1.3.0.jar`, sha256 e1299fd6...d1835).
`python tools/patch_all.py apks patched --apktool apktool_3.0.3.jar --signer uber-apk-signer-1.3.0.jar`

Proven on a throwaway copy of an unrelated APK (not a Fortnite build): launcher activity found from the manifest, `onResume` hooked,
3 smali classes added in a new `classes2.dex`, `<queries>` added, rebuilt, signed (v1/v2/v3, zipalign verified).
Not proven: running on a device; behaviour inside any Fortnite build. Fortnite's real game activity may differ from its launcher/splash activity:
pass `--activity <class>` once a real build has been inspected.
The button only starts the OGFN Manager backend. Redirecting a build's Epic hostnames to that backend is NOT implemented (needs real builds inspected).
Re-signing changes the signature, so the stock app has to be uninstalled first.

## Getting the launcher APK
This machine cannot reach Google/Maven, so the APK is built by GitHub Actions: put `catalog.json` in `launcher-app/app/src/main/assets/`, then in this folder
`git init && git add . && git commit -m init`, create an empty GitHub repo, `git remote add origin <url> && git push -u origin HEAD`, open the Actions tab,
run "build-launcher", and download the `FortniteHub-debug-apk` artifact (debug-signed, installable).

## Disk
The drive holding Downloads had about 14 GB free on 2026-11-22. All 376 APKs are roughly 60 to 90 GB, patched copies need as much again.
`download_queue.py` and `patch_all.py` stop when space is short; use `--out` on a bigger drive.

## Host screen
Embedded noVNC in a WebView (touch = mouse, soft keyboard = keyboard). Needs a VNC server on the PC (TightVNC or UltraVNC, port 5900). RDP and Winlator are launched as external apps; the `rdp://` handler and the Winlator package name are unverified.
