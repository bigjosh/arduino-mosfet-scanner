# Theory of Operation

How the Arduino MOSFET Scanner actually works, end to end — the analog signal
chain, why each resistor is the value it is, how a 10-bit AVR reaches < 5 %
accuracy, the firmware protocol, and the web/Android software stack. For the
build sequence see [README.md](README.md); for design history and the v2 roadmap
see [plan.md](plan.md).

## Contents
1. [Design philosophy](#1-design-philosophy)
2. [Naming and the signal chain](#2-naming-and-the-signal-chain)
3. [Generating the scan voltages](#3-generating-the-scan-voltages)
4. [Measuring current](#4-measuring-current)
5. [The ADC: < 5 % from a 10-bit converter](#5-the-adc--5-from-a-10-bit-converter)
6. [Dual-reference selection: crossover, cutoff, and resolution](#6-dual-reference-selection-crossover-cutoff-and-resolution)
7. [Why each resistor is what it is](#7-why-each-resistor-is-what-it-is)
8. [LOW_IO and the three-phase cycle](#8-low_io-and-the-three-phase-cycle)
9. [Firmware: protocol and robustness](#9-firmware-protocol-and-robustness)
10. [The software stack and why Android needs a wrapper](#10-the-software-stack-and-why-android-needs-a-wrapper)
11. [One measurement, end to end](#11-one-measurement-end-to-end)
12. [Limits of this version (v1)](#12-limits-of-this-version-v1)

---

## 1. Design philosophy

Two principles drive every decision:

- **Dumb firmware, smart host.** The Arduino only *sets DAC codes, drives one
  GPIO, and reports raw averaged ADC counts*. Every volt, amp, calibration, and
  curve is computed on the host (the web app or the Python tools). The firmware
  has no notion of "current" or "MOSFET." This means the analysis can evolve
  forever without ever reflashing a board, and the same firmware backs the web
  app, the Android app, and Python identically.
- **Viability, not metrology.** The rig answers *"can this MOSFET be cascaded
  with digital logic?"* — gate tightness and turn-on behavior — so the accuracy
  target is **< 5 %**, not lab-grade. That budget is what makes a $30 Uno-based
  design possible: burden resistors instead of precision current sources, a
  10-bit ADC instead of a 16-bit front end.

## 2. Naming and the signal chain

Pins are **High / Gate / Low**, never drain/source, so the same rig works for
N- and P-channel parts. Channel current flows **High ↔ Low**; the Gate controls
it; **Low is the grounded current return** in v1.

```
        MCP4725 #1 (SDA=D4, SCL=D5)
soft-I2C ─►[ DAC_H ]─[ 100 Ω ]──────────────────► HIGH pin ──┐
                                          A2 ◄────┘ (readback) │
        MCP4725 #2 (SDA=D4, SCL=D6)                          [ DUT ]
soft-I2C ─►[ DAC_G ]─[ R_gate 1 MΩ ]──┬──────────► GATE pin   │
                              A1 ◄─────┘ (+5 nF→GND)           │
                                                              ▼
                                       LOW pin ──┬─[ R_low 1 kΩ ]─┬─► LOW_IO (D3)
                                          A0 ◄───┘          A3 ◄──┘
```

Four ADC inputs sense four nodes; two DACs drive two pins; one GPIO (D3) is the
current return. Everything shares one ground (Uno + DACs + DUT).

## 3. Generating the scan voltages

### MCP4725 DACs — ratiometric 12-bit
Each MCP4725 is a 12-bit I²C DAC whose output is **ratiometric to its supply**:

```
V_out = (code / 4096) × VDD          code ∈ [0, 4095]
```

Because the DAC is referenced to the same 5 V rail as everything else, its output
tracks the rail — which we exploit rather than fight (see VDD compensation below).
Writes use the **fast-mode** 2-byte form (`firmware … dacWrite()`): the high
nibble of the code (with the power-down bits = 00) then the low byte.

### VDD compensation — accurate volts from a drifting rail
A USB 5 V rail is really 4.7–5.1 V and drifts with load. A naive `code =
V/5 × 4096` would inherit that error directly. Instead the firmware solves for
the code using the **measured** rail (`vddMv`, from §5):

```
code = round( V_target × 4096 / VDD )          (clamped to [0, 4095], V to [0, 5])
```

Since `V_out = code/4096 × VDD`, the VDD cancels and the commanded voltage lands
on target regardless of where the rail actually sits. `SETH`/`SETG` return the
`CODE=` used so the host knows the exact value requested.

### Bit-banged I²C — why not the hardware TWI
I²C is **bit-banged in software** (`D4 = SDA` shared, `D5 = SCL→DAC_H`,
`D6 = SCL→DAC_G`) as open-drain emulation: a line is only ever **driven low** or
**released** to the pull-ups, never driven high (the PORT bit is cleared before
switching to OUTPUT and set after switching to INPUT, so there are no
driven-high glitches). Clock stretching is honored with a bounded wait.

This buys three things:
- **One SCL per DAC** means two *identical, unstrapped* MCP4725 modules coexist
  — each can sit at address 0x60 because only its own SCL clocks it; the idle DAC
  sees SDA wiggle without clock edges and latches nothing. A 3rd DAC later is
  just one more SCL pin.
- It sidesteps a real bring-up failure: the hardware TWI on A4/A5 never ACKed on
  the bench (and a drive-low test browned out the board), so the design moved to
  bit-banged I²C on D4–D6.
- It's slow (~30–50 kHz; a DAC write ≈ 1 ms) but that's negligible next to a
  measurement, and frees A4/A5 for future ADC use.

At boot the firmware probes **0x60–0x67** on each bus and takes the first ACK;
**role is physical** — whatever D5 clocks is DAC_H. `SCAN?`, `RESCAN`, and
`PINTEST` exist for diagnosing a bus that won't ACK.

## 4. Measuring current

There is no current sensor. Current is inferred from the voltage drop across a
known **burden resistor**, read by the ADC.

### Channel current — burden resistor + GPIO-resistance cancellation
The channel current returns through **R_low** to the LOW_IO GPIO (D3). Two ADC
taps straddle R_low: **A0** at the DUT (Low) side, **A3** at the GPIO side:

```
I_ds = (V_A0 − V_A3) / R_low          (positive = High → Low)
```

Why two taps instead of assuming the GPIO is at 0 V? An AVR output pin has
~25–40 Ω of internal resistance. At 1 mA through a 1 kΩ R_low that's up to ~4 %
error if you treat D3 as a perfect ground. Measuring **both ends** of R_low and
subtracting cancels the GPIO resistance entirely — A3 is the trick that keeps the
current reading honest.

### Gate current — the "is it leaky?" sense
The gate is driven through a deliberately huge **R_gate = 1 MΩ**, with **A1**
reading the gate-side node:

```
I_gs = (V_cmd_G − V_A1) / R_gate
```

`V_cmd_G` is the commanded DAC voltage; `V_A1` is what the gate node actually
sits at. If the gate doesn't leak, no current flows through R_gate and
`V_A1 ≈ V_cmd_G` → I_gs ≈ 0. If it leaks, the node sags and the sag *is* the
current. (Limits in §7.)

### Bias readback — burden-corrected, not assumed
Because the Low node lifts off ground under load, Vgs/Vds are computed from
**measured** node voltages, not commanded ones:

```
V_ds = V_A2 − V_A0       (High pin after the 100 Ω, minus the Low node)
V_gs = V_A1 − V_A0       (Gate node minus the Low node)
```

This is why A2 sits *after* the 100 Ω series resistor (it reads the real High-pin
voltage under load) and why A0 is used as the bias reference rather than 0 V.

## 5. The ADC: < 5 % from a 10-bit converter

The ATmega328P has a single 10-bit ADC (0–1023 counts). Three techniques stretch
it to the accuracy target.

### Dual reference: resolution *and* range, auto-selected
Every `MEAS?` reads **all four pins under both references**:
- **INTERNAL 1.1 V** bandgap reference → ~1.07 mV/LSB — high resolution for
  small signals (low-current channel drops, gate sag).
- **DEFAULT ≈ 5 V (AVcc)** reference → ~4.9 mV/LSB — full 0–5 V range.

The host then picks, **per node**, whichever reading is in range (`convert.js
nodeVolts()`): use the 1.1 V reading while it's below ~1010 counts (≈ 1.08 V),
otherwise fall back to the 5 V reading. If even the 5 V reading rails (≥ ~1010
counts ≈ 4.93 V) the point is tagged `flag=clip`. No range modes, no firmware
state, no contradictions — a node near GND gets ~1 µA/LSB resolution while a node
near the rail is simply read on the coarse reference. The exact crossover, the
per-reference resolution, and the reason the cutoff sits below full scale are
detailed in [§6](#6-dual-reference-selection-crossover-cutoff-and-resolution).

### AREF settling — the subtle part
Only one reference is active at a time, so `MEAS?` reads all four pins on AVcc,
switches to INTERNAL, then reads all four again. The Uno has a 100 nF cap on
AREF; the internal bandgap charges it through ~32 kΩ, so switching **into**
INTERNAL needs real settling time. The firmware discards a couple of conversions
and waits **~10 ms for INTERNAL** (only ~3 ms for the low-impedance AVcc), plus a
throwaway read after every mux change, before averaging `avgN` samples (default
32).

### VDD self-measurement via the bandgap — the keystone
The same 1.1 V bandgap can be read *as an ADC input* against the AVcc reference.
That inverts to give the actual rail:

```
VDD_mV = V_bandgap_mV × 1024 / counts
```

`MEAS?` reports `VDD_MV` every time, so both the DAC scale (§3) and the 5 V-ref
conversions track the live rail.

### Calibration — one number in EEPROM
Everything above hinges on knowing the *true* bandgap voltage, which varies
1.0–1.2 V part to part. `CALBG <mV>` stores the measured value in MCU EEPROM
(default 1100, persisted across resets). This single calibration — done once
against a DMM during bring-up — is what pulls the whole chain inside 5 %, because
the bandgap is the reference that the rail measurement, the DAC scale, and the
5 V-ref readings all depend on.

## 6. Dual-reference selection: crossover, cutoff, and resolution

§5 covered *why* every node is read under both references; this is the exact
selection rule, the numbers behind it, and the reason the cutoff sits where it
does.

### Resolution at each reference
The ADC is 10-bit (0–1023 counts), so each reference's LSB is its full scale ÷ 1024:

| Reference | LSB (voltage) | LSB current @ R_low = 1 kΩ | @ 10 kΩ |
| --- | --- | --- | --- |
| INTERNAL 1.1 V | 1.1 / 1024 ≈ **1.07 mV** | ≈ **1.07 µA** | ≈ 0.107 µA |
| AVcc ≈ 5 V | 5.0 / 1024 ≈ **4.9 mV** | ≈ **4.9 µA** | ≈ 0.49 µA |

The 1.1 V reference is **~4.6× finer** (5.0 / 1.1). That factor is the whole
point of carrying it: small signals resolve far better on the internal reference
than the 5 V reference could ever manage.

### The selection rule (per node)
`convert.js nodeVolts()` decides independently for each node from its two counts,
`c_1V1` and `c_5V`:

```
if  c_1V1 < 1010 :  V = (c_1V1 / 1024) × V_ref_int     // fine reading
else             :  V = (c_5V  / 1024) × VDD           // coarse reading
                    clip = (c_5V ≥ 1010)
```

- **Crossover.** 1010 counts on the 1.1 V reference = 1010/1024 × 1.1 ≈
  **1.085 V**. Below it a node uses the fine reference; at/above it, the coarse
  one. In channel-current terms (R_low = 1 kΩ) the crossover is ≈ **1.085 mA** —
  below it you resolve to ~1.07 µA, above it to ~4.9 µA.
- **Per-node and independent.** A0 and A3 (the two ends of R_low) are selected
  separately, so `Ids = (V_A0 − V_A3)/R_low` may combine a fine reading on one end
  with a coarse reading on the other. In a forward scan both ends sit near ground
  → both fine; in the reverse scan (LOW_IO = 5 V) both sit near the rail → both
  coarse (~4.9 µA/LSB).
- **The `clip` flag.** When even the coarse reading is pinned (`c_5V ≥ 1010`, ≈ ≥
  4.93 V) the node has railed on *both* references — the reported value is a
  floor/ceiling, not a real measurement, so the point is tagged `flag=clip`.

### Why the cutoff is 1010, not 1023
Full scale is 1023, yet the fine reading is abandoned ~13 counts (~14 mV) early,
at a node voltage of ~1.085 V rather than the ~1.10 V reference ceiling. Three
reasons, in order of importance:

1. **Saturation ambiguity.** Above the 1.1 V reference the ADC pins at 1023, and a
   pinned reading can't tell 1.10 V from 5 V apart. Only a value *strictly inside*
   the linear region is trustworthy, which already rules out the top codes.
2. **One-sided clipping bias on the average.** `MEAS?` averages 32 samples
   (`avgN`). Near full scale, per-sample noise pushes some samples past 1023,
   where the hardware *clamps* them to 1023. Truncating only the high tail drags
   the 32-sample mean **below** the true value, so a node genuinely at ~1.09 V
   reads low. Keeping the average out of that regime is the main reason the cutoff
   sits well back from 1023, not just one count below it.
3. **Top-code nonlinearity.** The ATmega328P ADC's INL/DNL degrade near both
   rails; the highest codes are its least accurate. Trimming them keeps the fine
   reading honest.

So 1010 (≈ 1.085 V) is where the fine reading is still unambiguous, unbiased, and
linear. The handful of counts between there and the ~1.10 V ceiling are exactly
the untrustworthy ones — which is why pushing the crossover higher buys almost
nothing real.

### Considered alternative (not used)
You could instead decide from the full-range 5 V reading — it never saturates, so
it could authorize the fine reading right up to the true ~1.10 V ceiling, gaining
~1 % more high-resolution range. We keep the simple count threshold because that
recovered ~1 % *is* the clipping-biased, nonlinear top-of-range — not worth the
extra logic. The real levers for more high-resolution current range are R_low (the
socketed 10 kΩ option) and the v2 current-sense front end.

## 7. Why each resistor is what it is

| Part | Value | Reasoning |
| --- | --- | --- |
| **R_low** | **1 kΩ** (socketed) | Sizes the *expected* max channel current (1 mA) to ~1.0 V — just under the 1.1 V reference, so it stays on the high-resolution ref. Gives ~1.07 µA/LSB. Swap to **10 kΩ** for sub-100 µA DUTs (100 µA → 1.0 V, ~0.1 µA/LSB) at the cost of clipping above ~110 µA — hence the socket. |
| **R_gate** | **1 MΩ** | Two jobs at once. (a) Sensitivity: 1 µA of leakage → 1 V of sag, easily measured. (b) A built-in current limit: at 5 V the gate can deliver only ≈ 5 µA before its node collapses — the "device is leaky, don't care how leaky" regime. Consequence: clean Igs only to ~1 µA, ~5 µA hard ceiling (deliberate v1 limit). |
| **100 Ω** (High) | **100 Ω** | Short protection: if the High pin shorts to Low, current is limited to ≈ 5 V/(100 Ω + R_low) ≈ 4.5 mA — safe for the DAC and board. A2 senses *after* it (at the pin), so it reads true High-pin voltage under load and costs no accuracy. |
| **Cap on A1** | **~5 nF** (5–10 nF) | The 1 MΩ gate node has far too high a source impedance for the AVR's sample-and-hold (which wants < 10 kΩ); without a charge reservoir the ADC would under-sample it. 5 nF gives τ = R_gate·C = 5 ms — much faster than the 200 ms gate-settle, so it adds no measurement lag. |

## 8. LOW_IO and the three-phase cycle

**LOW_IO (D3)** is an ordinary GPIO used as a switchable current return:
`LOWIO 0` drives it low (ground return — normal), `LOWIO 1` drives it to 5 V
(a **source pedestal**), `LOWIO Z` floats it. Driving it to 5 V lifts the Low
node up so the channel can be biased *negative* relative to it — the v1 trick for
reaching negative Vgs/Vds without a second drive DAC.

A full device cycle is three phases (`scan.js`, mirroring `scan_arduino.py`):

1. **Gate leakage.** Forward: `DAC_G=5 V, DAC_H=0 V, LOW_IO=0` → measure the
   forward drop across R_gate. Reverse: `DAC_G=0 V, DAC_H=5 V, LOW_IO=5 V` →
   reverse drop. Reported against the ~1 µA "leaky" ceiling.
2. **Positive-Vgs map.** `LOW_IO=0` (Low node near GND → the 1.1 V-ref readings
   are in range). Sweep `DAC_G` 0→5 V in 250 mV steps; at each gate step sweep
   `DAC_H` 0→5 V in 100 mV steps. Yields Ids-vs-Vds curve families, one line per
   commanded Vgs.
3. **Negative-Vgs map.** `LOW_IO=5 V` pedestal (Low node near 5 V → the 5 V-ref
   readings are the in-range ones, ~4.9 µA/LSB). Same sweep. On an N-channel part
   this mostly resolves the **body diode** (conducts below Vds ≈ −0.6 V); the
   current is bounded by R_low to ~4.3 mA worst case — safe for the GPIO sink and
   the DAC.

Defaults (100 mV / 250 mV steps) give a ~2–3 min quick-look at ~65 ms/point,
dominated by the dual-reference `MEAS?`.

## 9. Firmware: protocol and robustness

- **Line protocol, 115200 8N1.** One ASCII command per line, exactly one reply
  line per command — trivial to drive from any language and to reason about (the
  host runs one command in flight at a time). Command reference in
  [firmware/README.md](firmware/README.md).
- **Boot banner + auto-reset.** Opening the serial port asserts DTR, which resets
  the Uno; `setup()` prints `ArduinoMosfetScanner v1 …`. Hosts wait for that
  banner (or poll `IDN?`, which returns the same line) to confirm the link.
- **Glitch immunity.** Non-printable RX bytes are dropped, so a USB-serial bridge
  hiccup can't corrupt a command. Replies are pure printable ASCII.
- **Safe cold boot.** MCP4725s ship with **mid-scale (~2.5 V) in their EEPROM**
  and replay it at power-on — which would dump 2.5 V onto a DUT before the host
  connects. `setup()` zeroes both DACs immediately, and `SAVEZERO` writes 0 V
  into the DAC EEPROMs so even a cold power-up (no host) starts at 0 V.

## 10. The software stack and why Android needs a wrapper

The same web app (`docs/`, a static PWA on GitHub Pages) runs everywhere. It
talks to the rig through one of three interchangeable **transports**
(`docs/js/transport.js`), chosen automatically by `pickTransport()`:

| Transport | Used on | Mechanism |
| --- | --- | --- |
| **Web Serial** | Desktop Chrome/Edge | `navigator.serial` — the browser owns the serial port directly. |
| **WebUSB-CDC** | (fallback) | Raw `navigator.usb`, speaking the CDC-ACM protocol in JavaScript. |
| **Android bridge** | The Android shell app | `window.AndroidSerial`, injected natively (below). |

### Why the wrapper is necessary on Android
Plain Chrome on Android **cannot reach a wired CDC serial device by any web API**:

- **Web Serial** (`navigator.serial`) was desktop-only for years; Android support
  only began arriving in 2026 and is device-gated, so it can't be relied on.
- **WebUSB** (`navigator.usb`) *exists* on Android, but Chrome **fences the
  CDC-data interface** — a page is not allowed to `claimInterface()` the
  communications-class interface a USB-serial adapter exposes (there's no Android
  serial API behind it), so the claim fails with a SecurityError. The raw
  WebUSB-CDC transport that works on some platforms is blocked here.

So the browser is a dead end on Android — which is exactly what you observed on
the Pixel: a native serial terminal sees the Uno, but the web page can't.

**Native apps don't have this problem.** They use the **Android USB Host API**
(`UsbManager`/`UsbDeviceConnection`) in userspace, and the
[usb-serial-for-android](https://github.com/mik3y/usb-serial-for-android) library
implements CDC-ACM (and CH340/FTDI/CP210x) on top of it — no kernel driver, no
browser fence.

### What the wrapper actually is
The Android app (`android/`) is a ~300-line shell:

1. A fullscreen **WebView** loads the *same* GitHub Pages app — so all UI/feature
   code stays in the shared web app and updates by `git push`, no APK rebuild.
2. It injects **`window.AndroidSerial`**, a bridge backed by
   usb-serial-for-android exposing `list / connect / write / close / saveFile`,
   with bytes base64-encoded across the JS boundary and device events
   (`connect`/`data`/`error`/`disconnect`) pushed back into the page.
3. The web app's `pickTransport()` **auto-prefers** that bridge when present.

The APK only ever needs rebuilding when the bridge or the Pages URL changes. It
also adds niceties a browser can't: the `USB_DEVICE_ATTACHED` intent grants USB
permission on plug-in, keep-screen-on, CSV export into the system Downloads
folder, and no background-tab timer throttling. (Desktop, by contrast, uses Web
Serial because desktop WebUSB can't claim a CDC interface the OS serial driver
already owns — the mirror image of the Android situation.)

### The connect handshake
Opening the link DTR-resets the board, so the apps wait for the boot banner
**after** the port is open — crucially *not* starting the timeout until the
permission dialog is dismissed — and fall back to polling `IDN?` if the
spontaneous banner is missed. This makes connection robust to a slow permission
dialog and to an auto-reset that didn't fire. A failed attempt closes the
transport so a retry starts from a clean handle.

### The rest of the app
Vanilla ES modules, no build step. Hand-rolled canvas charts (viridis curve
families, live per-point). Scans persist to **IndexedDB**; CSVs download on
demand with the *same columns and filenames* as the Python tools
([hardware/README.md](hardware/README.md) has the schema). It's an installable
**PWA** with a network-first service worker (always fresh online, fully usable
offline after first load) and holds a **wake lock** during scans. `?demo` swaps
in a `MockTransport` that simulates the firmware *and* a synthetic FET, so the
entire app — scan, abort, history, CSV, the full bring-up wizard — runs and is
testable with no hardware.

## 11. One measurement, end to end

Tracing a single point ties it all together:

1. Host: `SETG 2.500` → firmware computes `code = 2.5×4096/VDD`, bit-bangs it to
   DAC_G; gate node settles through R_gate + the A1 cap.
2. Host: `SETH 1.000` → DAC_H code → 100 Ω → High pin.
3. Host: `MEAS?` → firmware measures VDD via the bandgap, reads A0–A3 on AVcc,
   settles into the 1.1 V ref, reads A0–A3 again, replies with eight counts +
   `VDD_MV`.
4. Host (`convert.js`): for each node pick the in-range reference →
   `V_A0…V_A3`; then `I_ds=(V_A0−V_A3)/R_low`, `I_gs=(V_cmd_G−V_A1)/R_gate`,
   `V_ds=V_A2−V_A0`, `V_gs=V_A1−V_A0`, with a `clip` flag if any node railed.
5. Plot the point, append a CSV row, repeat across the sweep grid.

The firmware never knew a MOSFET was involved.

## 12. Limits of this version (v1)

These are deliberate scope cuts to get the full chain working cheaply; each has a
v2 path in [plan.md](plan.md).

- **0–5 V only.** Drives come straight from the MCP4725s with no gain stage. v2
  adds op-amps (force-sense at the terminal) for 0–15 V.
- **No independent Low drive / true negative bias.** Negative Vgs/Vds is faked
  with the `LOW_IO = 5 V` pedestal trick, which mostly exposes the body diode. v2
  adds a Low DAC + differential sense for a real source pedestal.
- **Gate current to ~1 µA only.** The 1 MΩ R_gate trades range for sensitivity
  and self-limits at ~5 µA. Fine for "is it leaky?"; v2 would use a smaller
  R_gate or a proper current-sense (e.g. INA219/226) for real Igs range.
- **Channel current ~1 µA/LSB to ~1 mA** at R_low = 1 kΩ (10 kΩ for low-current
  parts, clipping above ~110 µA). Body-diode current is bounded to ~4.3 mA.
- **10-bit ADC, ~5 % target.** Dual-reference reading and the bandgap-based VDD
  calibration are what make that achievable; it is a viability screen, not a
  curve tracer.
- **One device, one host at a time.** Bit-banged I²C is ~30–50 kHz; throughput is
  set by the dual-reference `MEAS?` (~65 ms/point), not the I²C.
- **Keep the UI foregrounded** during web scans — background tabs get
  timer-throttled by the browser (the wake lock handles this on phones).
