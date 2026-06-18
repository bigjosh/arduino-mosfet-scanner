# Arduino MOSFET Scanner (v1, 0–5 V)

A ~$30 bench rig — an **Arduino Uno R3** plus two **MCP4725** DACs — that answers
one question about a MOSFET: *is it viable for cascaded digital logic?* It sweeps
gate and drain voltages and reports **gate leakage** plus **Ids(Vds, Vgs)** maps
for positive and negative gate bias.

Pins are named **High / Gate / Low** (not drain/source/gate) so the same rig
works for N- and P-channel parts. Drive it from your **phone**, a **desktop
browser**, or **Python** — same firmware, same protocol.

▶ **Try the UI right now, no hardware:** <https://bigjosh.github.io/arduino-mosfet-scanner/?demo>

![Wiring schematic](hardware/wiring.png)

## What's in this repo

| Folder | What |
| --- | --- |
| [`firmware/`](firmware/) | The Arduino sketch (`mosfet_scanner.ino`) + flashing guide. Deliberately dumb: it sets DACs and reports raw ADC counts; all the physics is done on the host. |
| [`hardware/`](hardware/) | Wiring diagram, bill of materials, measurement notes. |
| [`docs/`](docs/) | The **web app** (a PWA served by GitHub Pages): live charts, scan history, guided bring-up. |
| [`android/`](android/) | The **Android shell app** — a thin WebView wrapper that adds a native USB-serial bridge (plain Chrome on Android can't reach wired serial devices). |
| [`host-tools/`](host-tools/) | Optional **Python** desktop tools (`scan_arduino.py`, `bring-up.py`) — the original reference the web app mirrors. |
| [`plan.md`](plan.md) · [`spec.md`](spec.md) | Design rationale, calibration strategy, v2 roadmap. |

## Getting started: fresh build → first scan

### 1. Build the rig
Wire an Uno + 2× MCP4725 + a few resistors per **[hardware/README.md](hardware/README.md)**
(full ~$30 parts list there).

### 2. Flash the firmware
Upload `firmware/mosfet_scanner/mosfet_scanner.ino` to the Uno — Arduino IDE or
`arduino-cli`, both covered in **[firmware/README.md](firmware/README.md)**. Once
per board.

### 3. Open the app and connect
Same UI everywhere — pick whatever's handy:

- **📱 Android phone** — install the shell app (APK). Walkthrough:
  **[how-to-install.md](how-to-install.md)**. (Plain Chrome on Android can't open
  wired serial ports, so the phone needs this app.)
- **💻 Desktop** — open **<https://bigjosh.github.io/arduino-mosfet-scanner/>** in
  **Chrome or Edge**, plug the Uno into a USB port, click **Connect**.
- **🐍 Python** — **[host-tools/README.md](host-tools/README.md)** to run from a
  terminal instead.

No hardware yet? Append **`?demo`** (or tap **Demo**) for a fully simulated rig.

### 4. Bring-up (first time on a new rig)
Open the **Bring-up** tab and run the wizard: link → harness check → DAC comms →
no-DUT self-test → jumper-wire functional tests → DMM calibration, with pass/fail
and wiring hints at each step. This catches wiring mistakes before you trust any
measurement. (Terminal equivalent: `python bring-up.py`.)

### 5. Scan a device
Drop a MOSFET into the socket, set the sweep on the **Scan** tab (defaults give a
~2–3 min quick-look), and run. Live curves; results save to in-app History and
export as CSV.

## What you get out

Three views per device cycle:
- **Gate leakage** — forward/reverse |Igs| against the ~1 µA "leaky" ceiling.
- **+Vgs map** and **−Vgs map** — Ids-vs-Vds curve families, one line per
  commanded gate voltage.

CSV columns lead with the legacy `Vds (V), Vgs (V), Ids (uA)` grid, then measured
extras. Signs: positive Ids = High→Low; positive Igs = into the gate. Full schema
and measurement math in [hardware/README.md](hardware/README.md).

## Scope

v1 covers **0–5 V** on all pins and resolves gate leakage to **~1 µA** — a
deliberate limit, since the goal is *viability*, not lab-grade accuracy. The
0–15 V range and wider current sensing are the v2 roadmap in [plan.md](plan.md).

## Links
- **Live app:** <https://bigjosh.github.io/arduino-mosfet-scanner/>
- **Android APK:** [latest release](https://github.com/bigjosh/arduino-mosfet-scanner/releases/latest)
- **Demo (no hardware):** <https://bigjosh.github.io/arduino-mosfet-scanner/?demo>
