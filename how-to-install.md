# Install the MOSFET Scanner on an Android phone

The scanner UI can't reach the rig from **plain Chrome on Android** — current
Chrome blocks wired CDC serial devices. So on a phone you run a small **shell
app** instead: it loads the same web UI and adds a native USB-serial bridge with
no such restriction. (On a desktop you don't need this — just open
[the web app](https://bigjosh.github.io/arduino-mosfet-scanner/) in Chrome/Edge.)

**Before you start:** flash the firmware onto the Uno once
([firmware/README.md](firmware/README.md)) and wire the rig
([hardware/README.md](hardware/README.md)).

Everything below is done **on the phone**.

## 1. Download the app
Tap to download the latest APK (~2 MB):

**➡️ [Download app-release.apk](https://github.com/bigjosh/arduino-mosfet-scanner/releases/latest/download/app-release.apk)**

If that doesn't start a download, open the
[Releases page](https://github.com/bigjosh/arduino-mosfet-scanner/releases/latest)
and tap `app-release.apk` under **Assets**.

## 2. Install it
1. Open the downloaded file (browser download bar, or **Files → Downloads**).
2. Android will ask to **allow installing unknown apps** for whatever you
   downloaded with (Chrome, Files, …). Enable it, then go back. One-time prompt.
3. Tap **Install**, then **Open**.

> Updating later: install a newer APK over the top — history and settings survive
> (same signing key).

## 3. Connect the rig
1. Plug the rig into the phone with a **USB-C OTG** adapter/cable (the phone is
   the USB host and powers the Uno).
2. Open **MOSFET Scanner** and tap **Connect**.
3. Approve the **USB permission** dialog. Tick **"Use by default for this USB
   device"** so future connects are instant.

You're connected when the status chip reads **`Android USB`**. CSV exports land in
the phone's **Downloads** folder.

The app loads its UI from the web, so it updates itself — you only need a new APK
if the USB bridge changes.

## 4. Bring-up, then scan
- First time on a freshly built rig: open the **Bring-up** tab and run the wizard
  (link → harness → DACs → self-test → jumper tests → calibration).
- Then drop a MOSFET into the socket, set the sweep on the **Scan** tab, and run.

Want to try the UI first? Tap **Demo** (or open
[the demo](https://bigjosh.github.io/arduino-mosfet-scanner/?demo)) — a simulated
rig, no hardware needed.

## Troubleshooting
- **"App not installed" / blocked** — make sure the download finished, and that
  you enabled *install unknown apps* (step 2). Uninstall any older copy first.
- **"No USB serial device found"** — use a known-good **data** OTG cable (not
  charge-only), reseat both ends, confirm the Uno's power LED is on, tap Connect
  again.
- **"Could not claim…" / busy** — another serial app grabbed the device.
  Force-stop it, clear its *Open by default*, replug, retry.
- **Chip says `WebUSB (CDC)` or `Web Serial`, not `Android USB`** — you opened the
  website in a browser, not the installed app. Open the **MOSFET Scanner** app.
- The **USB?** tab shows a timestamped connect log if you need to see where an
  attempt stops.
