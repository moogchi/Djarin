# Djarin — USB-C Security/Protection Board

## Design Documentation

### Project Overview

Djarin is a USB-C inline board that protects, monitors, and (eventually) selectively passes through power and data. It combines power-path protection (ESD, reverse-voltage, overvoltage/surge) with PD negotiation monitoring and a security-oriented data disconnect feature, split across four physical PCB faces connected by FPC jumpers.

---

## Face 1 — Power Protection

### J1 — USB-C Receptacle

- Full-pin `USB_C_Receptacle` symbol (KiCad default library)
- Only VBUS, CC1, CC2, GND, SHIELD are used on this face
- D+/D-, RX1/RX2, TX1/TX2, SBU1/SBU2 reserved for Face 4 (SuperSpeed passthrough)

### U1 — USBLC6-2SC6 (TVS array)

- STMicroelectronics ESD protection array, 2 independent channels
- Channel 1 (pins 1/6) protects CC1; Channel 2 (pins 3/4) protects CC2
- Pin 5 = VBUS, Pin 2 = GND
- **Placement rule (from datasheet):** must sit as close to J1 as physically possible — minimizes trace length for effective ESD clamping
- Physical order in the power path: **J1 → U1 (TVS) → surge protection → reverse-voltage protection → rest of board**
  - TVS goes first because it's the fastest-reacting protection (nanosecond-scale ESD events)

![Face 1 TVS](./images/TVS.png)

### Surge Detection — U3 (LM397) + Q2

**Why needed:** reverse-voltage protection (below) only blocks current flowing the wrong _direction_ — it does nothing against an overvoltage event (e.g., bad charger, USB-Killer-style attack) where current is still flowing the correct direction, just at a dangerous magnitude.

**Topology:** Inverting comparator with hysteresis (LM397 datasheet Figure 8/9)

- U3 is powered from a stable **+5V** rail (Face 2's MP2393 output) — NOT from VBUS directly, since VBUS is the signal being measured and can't also be the stable reference
- **VBUS sense divider:** R6 (86kΩ) + R7 (10kΩ) from VBUS to GND, tap feeds U3 pin 1 (VIN−)
  - At VBUS = 24V, divider output = 24 × (10k/96k) ≈ 2.5V
- **Hysteresis network (off +5V):** R1 (10kΩ) from +5V to node; R10 (1MΩ) from node to GND; R8 (10.2kΩ) from node to OUTPUT; node feeds U3 pin 3 (VIN+)
  - Calculated to trip (VT2) at ≈ 24V on VBUS, ~25mV hysteresis band (VT1 − VT2)
- **R_pullup (10kΩ):** U3 pin 4 (OUTPUT, open-collector) to +5V
- **Q2 (N-MOSFET, Q_NMOS_GDS):** cutoff switch in series with VBUS
  - Drain ← VBUS (from TVS output)
  - Source → onward to Q1 (reverse-voltage protection stage)
  - Gate ← U3 OUTPUT node
  - LM397 output is LOW when VIN > VT (per datasheet) → normal operation = output HIGH (Q2 on, power flows) → surge trips = output LOW (Q2 off, power cut). Fail-safe behavior.
- Surge stage placed **before** reverse-voltage protection stage so Q1/U2 are themselves protected from surge damage.

![Face 1 Surge Protection](./images/Surge_Protection.png)

### Reverse-Voltage Protection — U2 (LM74610QDGKRQ1) + Q1

- "Ideal diode controller" — near-zero voltage drop vs. a passive diode, no GND pin needed (self-referenced across Anode/Cathode, functions like a diode electrically)
- Symbol/footprint imported from SnapMagic (not in KiCad default library)
- **Topology (datasheet "Smart Diode Configuration"):**
  - Anode → Q1 Source (VIN side)
  - Cathode → Q1 Drain (VOUT/protected side)
  - Gate Drive / Gate Pull Down → Q1 Gate
  - VCAPH/VCAPL → C1 (charge pump cap)
- **C1 (charge pump cap):** 1µF, X7R/COG, 16V+ rating (datasheet range: 220nF–4.7µF)
  - Keep physically away from Q1 — thermal isolation affects capacitance value
- **C2 (VIN bypass cap):** 10µF, X7R, low-ESR ceramic — wired in **parallel** (VBUS to GND), not in series
- **Q1 (N-MOSFET, Q_NMOS_GDS):** the actual reverse-blocking switch
- **Layout rules to carry into PCB stage (from datasheet Section 10):**
  - VIN-to-Source: thick trace/polygon (high current path)
  - Anode/Cathode sense traces: short as possible
  - Gate Drive/Gate Pull Down: no vias, short/thick traces, keep close to MOSFET gate

## ![Face 1 Reverse Polarity Protection ](./images/Reverse_Polarity.png)

## Face 2 — Brain (Power Regulation + PD Negotiation)

### U4 — MP2393 (Synchronous Buck, VBUS → 5V)

- Replaces originally-considered MP2307 (marked "Not Recommended for New Designs" by MPS — refer to MP2393)
- Input range 4.2V–24V covers full VBUS range (5–20V) with margin
- 650kHz switching, up to 3A output, integrated PG (Power Good) output
- Circuit follows datasheet Figure 6 (VIN=19V, VOUT=5V/3A) almost exactly:
  - C1 (22µF) + C1A (0.1µF) input caps
  - R5 (604kΩ) EN pull-up to VIN (auto-start)
  - R4 (20Ω) + C3 (1µF) bootstrap network on BST
  - L1 (~3.3–3.74µH, sized via MPS inductor calculator at 650kHz/5V-out/2A)
  - C2 + C2A (22µF each) output caps
  - Feedback divider: R3 (15kΩ), R1 (40.2kΩ), R2 (7.68kΩ), C5 (10pF) — sets 5V output per datasheet Fig. 6
  - C7 (1nF) PG decoupling, C4 (6.8nF) soft-start timing on SS
- Output distributed via global `+5V` power symbol (used on both Face 1 and Face 2 — KiCad treats same-named power symbols as one global net, no hierarchical labels needed for power rails)

## ![Face 2 Buck Converter](./images/Buck_Converter.png)

### U6 — AP2112K-3.3 (LDO, 5V → 3.3V)

- Powers STM32 + FUSB302 digital logic
- EN tied directly to VIN (always-on)
- C6 (1µF) input, C8 (1µF) output ceramic caps

### Still to do on Face 2:

- FUSB302 (PD negotiation reader, I2C)
- STM32 (reads FUSB302, drives OLED/LED, handles button, controls Face 4 mux)
- SWD programming header (or UART bootloader + BOOT0 — decision pending)

---

## Face 3 — Display / Control (planned, not yet built)

- SSD1306 OLED — displays PD negotiation status + Face 4 data-connection state
- Bicolor status LED — fault/normal indicator
- Single tactile button — toggles Face 4's data passthrough on/off

---

## Face 4 — SuperSpeed Passthrough + Security Mux (planned, not yet built)

**Rationale:** project scope expanded beyond basic power protection (considered "trivial"/commonly done) to include actual USB 3.x data passthrough with a security-oriented disconnect feature — a stronger differentiator for HWE recruiting.

- Low-capacitance ESD protection IC rated for SuperSpeed lines (e.g., TI TPD4S014/016) — USBLC6-2SC6 is NOT suitable for these speeds
- USB3-capable mux/switch IC (e.g., TUSB546-class part), enable pin driven by STM32
- Lets user toggle "Charge + Data" vs. "Charge Only" mode via Face 3's button
- MCU cannot process RX/TX SuperSpeed signals directly (needs dedicated PHY/SerDes, beyond STM32 capability) — this is pure passive passthrough through the mux, no MCU signal processing of the data itself
- **Explicitly out of scope:** Thunderbolt (requires active redriver/retimer silicon + protocol stack, not realistic for this project)
- **Known difficulty/risk:** differential pair length-matching + controlled impedance routing at PCB layout stage — new skill area, budgeting real time for this rather than treating as trivial

---

## Key Component Decisions Log

| Part           | Role                         | Why chosen                                                                                     |
| -------------- | ---------------------------- | ---------------------------------------------------------------------------------------------- |
| USBLC6-2SC6    | CC1/CC2/VBUS ESD             | Standard USB-C TVS array, cheap, well-documented                                               |
| LM74610QDGKRQ1 | Reverse-voltage protection   | Near-zero drop "ideal diode," no GND pin needed                                                |
| LM397          | Surge/overvoltage comparator | Pin-compatible with TL331, general-purpose, fast enough since TVS handles true fast transients |
| MP2393         | VBUS → 5V buck               | Active/recommended replacement for MP2307, wider input range, PG output                        |
| AP2112K-3.3    | 5V → 3.3V LDO                | Simple, standard logic supply for STM32/FUSB302                                                |

---

_Document reflects design state as of the most recent working session. Face 2 (FUSB302/STM32 detail), Face 3, and Face 4 schematics still to be completed._
