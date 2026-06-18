# Firmware

The Arduino sketch for the scanner. It's deliberately dumb: it sets the two DACs,
drives the LOW_IO pin, and reports **raw averaged ADC counts** over a line-based
serial protocol. All unit conversion and physics happens on the host (web app or
Python), so the firmware never needs reflashing when the analysis changes.

- Board: **Arduino Uno R3 / clone (ATmega328P)**
- No external libraries (only the built-in `EEPROM`; I2C is bit-banged)
- Serial: **115200 baud**, 8N1 — one command per line, one reply line per command

## Flash it (once per board)

### Arduino IDE (easiest)
1. Install the [Arduino IDE](https://www.arduino.cc/en/software).
2. Open `firmware/mosfet_scanner/mosfet_scanner.ino`.
3. **Tools → Board → Arduino Uno**, then **Tools → Port → (your Uno)**.
4. Click **Upload** (→).
5. **Close the Serial Monitor** afterwards — it holds the COM port and the
   scanner app/tools can't open the board while it's open.

### arduino-cli (scripted)
`build.ps1` (Windows PowerShell) compiles and optionally uploads:
```powershell
cd firmware
.\build.ps1                 # compile only
.\build.ps1 -Upload         # compile + upload, auto-detect the port
.\build.ps1 -Upload -Port COM5
```
It finds `arduino-cli` on PATH or at `%LOCALAPPDATA%\arduino-cli`. macOS/Linux
equivalent:
```bash
arduino-cli compile --fqbn arduino:avr:uno firmware/mosfet_scanner
arduino-cli upload  --fqbn arduino:avr:uno -p /dev/ttyACM0 firmware/mosfet_scanner
```

## First power-up — important

MCP4725 DACs ship with **mid-scale (~2.5 V) stored in EEPROM** and replay it at
power-on. The firmware zeroes both DACs at boot; to make a *cold* power-up safe,
run **`SAVEZERO` once** (writes 0 V into the DAC EEPROMs). The bring-up wizard
does this for you.

## Serial command reference

One command per line; the board replies with exactly one line.

| Command | Reply / effect |
| --- | --- |
| `IDN?` | `ArduinoMosfetScanner v1 DACH=0x60 DACG=0x60 VREFINT_MV=1100` (also printed at boot) |
| `SETH <volts>` / `SETG <volts>` | Set DAC_H / DAC_G, 0–5.000 V, VDD-compensated, clamped → `OK CODE=… VDD_MV=…` |
| `RAWH <code>` / `RAWG <code>` | Raw 12-bit DAC code (bring-up/debug) |
| `LOWIO 0\|1\|Z` | Drive LOW_IO (D3) low / high / float |
| `MEAS?` | Averaged counts for A0–A3 under **both** ADC refs, plus measured VDD |
| `AVG <n>` | Samples per pin per ref (1–200, default 32) |
| `VDD?` | Measure the 5 V rail via the internal bandgap |
| `CALBG <mV>` / `CALBG?` | Store / read the true bandgap voltage (EEPROM; calibration) |
| `SCAN?` | Probe 0x08–0x77 on both I2C buses |
| `RESCAN` | Re-detect DACs after rewiring (no replug) |
| `PINTEST [sec]` | Harness diagnostic on D4/D5/D6 (idle / pull-up / short / bridge) |
| `SAVEZERO` | Write 0 V into the MCP4725 EEPROMs (safe cold boot) |

The board **auto-resets when the host opens the serial port** (DTR) and prints
its banner; the apps wait for that banner, or poll `IDN?` if it's missed.

The dual-reference ADC strategy (how < 5 % accuracy is reached with no trimming)
is documented in [../plan.md](../plan.md).
