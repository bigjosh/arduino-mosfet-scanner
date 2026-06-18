# Host tools (Python desktop)

The original reference implementation the web app mirrors — handy if you'd rather
drive the rig from a terminal, script batch runs, or work fully offline. The web
app ([../docs/](../docs/)) does everything these do, so you don't need both.

Flash the firmware ([../firmware/README.md](../firmware/README.md)) and wire the
rig ([../hardware/README.md](../hardware/README.md)) first.

## Setup
```bash
pip install -r requirements.txt   # pyserial, numpy, matplotlib
```

## Bring-up (first time on a new rig)
```bash
python bring-up.py
```
Guided and interactive: link → `PINTEST` harness check → DAC comms + `SAVEZERO` →
no-DUT self-test → jumper-wire functional tests (High–Low Ohm's-law current,
Gate–Low Igs scale, Gate–High A1/A2 agreement) → DMM calibration, with PASS/FAIL
and wiring hints per step.

## Scan
```bash
python scan_arduino.py --repl       # talk to the firmware directly (bring-up/debug)
python scan_arduino.py              # full 3-phase cycle, default steps -> CSVs + PNGs
python scan_arduino.py --selftest   # rerun just the no-DUT electrical self-test
```
Useful flags: `--h-step` / `--g-step` (defaults 0.1 / 0.25 V → ~2–3 min/device;
go finer for interesting parts), `--rlow 10000` for low-current DUTs, `--phases
23` to skip the leakage check, `--port COMx` if auto-detect fails, `--cal-vdd <DMM
volts>` to automate the bandgap calibration.

> Keep the Arduino IDE **Serial Monitor closed** while these run — it holds the
> COM port and they can't open the board.

Outputs: per-phase `scan-arduino-<timestamp>_phase{1,2,3}.csv` + `.png`. Column
schema and signs are in [../hardware/README.md](../hardware/README.md).
