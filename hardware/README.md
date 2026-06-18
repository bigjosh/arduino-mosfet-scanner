# Hardware

![Wiring schematic](wiring.png)
*(regenerate with `python wiring_diagram.py` after wiring changes)*

Pins are named **High / Gate / Low** rather than drain/source/gate, so the same
rig characterizes both N- and P-channel parts. Channel current flows High↔Low;
the Gate controls it. **Low is the grounded current return in v1.**

## Bill of materials (~$30)

- **Arduino Uno R3** (or clone), ATmega328P.
- **2× MCP4725** I2C DAC breakouts — any vendor; **no address straps needed**
  (each DAC gets its own SCL line, so both can sit at 0x60).
- **R_low = 1 kΩ** 0.1 %, in a socket/header (swap to 10 kΩ for low-current DUTs
  ≤ 100 µA). Sets the channel-current scale.
- **R_gate = 1 MΩ** (1 % fine) — gate-leakage sense.
- **100 Ω** in series with High (short protection; sensed after it, so no
  accuracy cost).
- **~5–10 nF** ceramic from A1 to GND (the 1 MΩ gate node needs it to drive the
  ADC sample cap; 5 nF fitted).
- Breadboard, jumpers, a DUT socket. Optional clamp diodes.

## Wiring

I2C is **bit-banged** with a **shared SDA and one SCL per DAC** — identical
modules need no address straps.

| From | To |
| --- | --- |
| Uno **D4** | SDA of **both** MCP4725s (shared) |
| Uno **D5** | SCL of the **DAC_H** module |
| Uno **D6** | SCL of the **DAC_G** module |
| MCP4725 VDD / GND | Uno 5V / GND (common with the DUT) |
| **DAC_H OUT** | 100 Ω → **High** pin of DUT |
| **DAC_G OUT** | 1 MΩ (R_gate) → **Gate** pin |
| DUT **Low** pin | R_low (1 kΩ, socketed) → **D3** (LOW_IO) |
| **A0** | Low node (DUT side of R_low) |
| **A1** | Gate-pin node, plus the 5–10 nF cap to GND |
| **A2** | High pin (DUT side of the 100 Ω) |
| **A3** | D3 / LOW_IO side of R_low |

Roles are physical: whichever module's SCL is on **D5 is DAC_H** — swap the two
SCL wires to swap roles. A 3rd DAC later is just one more SCL pin (D2/D7).

If a DAC doesn't ACK, the firmware has diagnostics: `PINTEST` (per-line
pull-up/short/bridge check), `SCAN?` (probe both buses), `RESCAN` (after
rewiring). The bring-up wizard runs these for you.

## How current & voltage are derived

The firmware reports raw ADC counts; the host computes (details in
[../plan.md](../plan.md)):
- **Channel current:** `I_low = (V_A0 − V_A3) / R_low` (positive = High→Low; A3
  cancels the GPIO's output resistance).
- **Gate current:** `I_gate = (V_cmd_G − V_A1) / R_gate`.
- **Bias:** `Vgs = V_A1 − V_A0`, `Vds = V_A2 − V_A0`.

Each `MEAS?` reads every pin under **both** the 1.1 V internal reference (for
resolution) and the ~5 V rail reference (for range); the host auto-picks the
in-range reading per node. The board also self-measures its VDD each reading, so
the scale tracks the (USB-drifting) rail.

## CSV output schema

Sweep CSVs start with the legacy grid columns `Vds (V), Vgs (V), Ids (uA)`
(commanded values, so existing pivot tools work), then measured extras:
`Igs (uA), Vds_meas, Vgs_meas, Vhigh, Vlow, Vgate, Vlowio, flag`
(`flag=clip` → even the 5 V-ref reading railed). Gate leakage is meaningful only
to ~1 µA in v1 — a deliberate limit from the 1 MΩ R_gate.
