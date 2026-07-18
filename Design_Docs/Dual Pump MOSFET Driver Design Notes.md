# Dual Pump MOSFET Driver Design Notes

## Requirements

* Supply: 12 VDC
* Load: Brushed DC piston pump
* Normal current: ~3 A
* Startup / stall current: >5 A
* MCU output: 3.3 V GPIO
* Quantity: ~500 units
* Goal: Replace TE OJ/OJE relay with lower-cost MOSFET solution
* Optional PWM used only for soft-start / inrush limiting

---

# Recommended Architecture

One low-side N-channel MOSFET per pump.

## Components

### Power MOSFET

* AOD4184A
* TO-252 / DPAK
* 40 V rated
* Low RDS(on) at 4.5 V gate drive

### Flyback Diode

Replace S1G.

Recommended:

* SS56 (5 A, 60 V Schottky)
* SS510 (5 A, 100 V Schottky)
* STPS10L45 (10 A, 45 V Schottky)

S1G is acceptable for relay flyback but not ideal for motor PWM recirculation.

### Transistor

* MMBT4401
* Used as gate pull-down transistor
* Driven directly from MCU

---

# Gate Drive Circuit

```text
MCU_CH1
   |
 4.7k
   |
 Base
MMBT4401
   |
Emitter -> DGND

Collector
   |
Gate Drive Node
   |
 1k
   |
 +5V

Gate Drive Node
   |
 22R
   |
MOSFET Gate

Gate Drive Node
   |
10k
   |
MGND
```

## Component Values

| Component     | Value        |
| ------------- | ------------ |
| Base resistor | 4.7 kΩ       |
| Base pulldown | 47 kΩ–100 kΩ |
| Gate pullup   | 1 kΩ         |
| Gate resistor | 22 Ω         |
| Gate pulldown | 10 kΩ        |

### Logic

This configuration is active-low.

| MCU Output | MOSFET |
| ---------- | ------ |
| LOW        | ON     |
| HIGH       | OFF    |

Firmware may invert the signal if desired.

---

# Power Stage

```text
+12V
  |
 Fuse
  |
 Pump
  |
Drain
 MOSFET
Source
  |
 MGND
```

---

# Flyback Diode

```text
         +12V
           |
           |
         Pump
           |
           +------+
           |      |
         Drain   Diode
           |      |
           +------+
```

Connections:

* Cathode → +12V
* Anode → MOSFET drain

Keep the diode physically close to the MOSFET and pump connection.

---

# Grounding

Separate grounds in schematic:

* DGND = MCU and logic
* MGND = pumps, MOSFETs, power returns

Join at a single point using:

* NetTie_2 (preferred)
* 0 Ω resistor

Example:

```text
DGND ---- NetTie ---- MGND
```

Place the star point near:

* Supply entry
* Bulk capacitor negative terminal

Do not allow motor return current to flow through MCU ground paths.

---

# Bulk Capacitance

Place near MOSFETs:

* 1000 µF to 2200 µF
* 25 V rating minimum

Additional:

* 100 nF ceramic near each MOSFET
* 100 nF ceramic across motor terminals if EMC becomes an issue

---

# PWM Soft-Start

PWM is not intended for speed control.

Purpose:

* Reduce startup current
* Reduce supply droop
* Reduce connector stress

Example ramp:

```text
20% duty
40% duty
60% duty
80% duty
100% duty
```

over approximately 0.5–2 seconds.

Suggested PWM frequency:

* 500 Hz to 2 kHz

No dedicated gate driver is required for this soft-start approach.

---

# Cost Optimization

At 500 pcs:

Approximate per-channel BOM:

| Component | Cost       |
| --------- | ---------- |
| MMBT4401  | $0.01–0.02 |
| AOD4184A  | $0.15–0.35 |
| SS56      | $0.03–0.06 |
| Resistors | <$0.01     |

Estimated total:

* ~$0.20–0.45 per channel

This is substantially lower than a TE OJ/OJE relay implementation.

---

# Final Recommendation

For lowest cost while retaining robustness:

* MMBT4401
* AOD4184A
* SS56
* 4.7 kΩ base resistor
* 22 Ω gate resistor
* 1 kΩ gate pullup
* 10 kΩ gate pulldown
* NetTie between DGND and MGND
* 1000–2200 µF bulk capacitor

This supports relay replacement and optional soft-start PWM without requiring a dedicated MOSFET gate driver.
