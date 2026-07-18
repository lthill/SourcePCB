# Buck Regulator Circuit: 12V → 5V @ 1.5A

**IC:** TPS54331DDAR (Texas Instruments)  
**Topology:** Non-synchronous, current-mode PWM, fixed 700kHz  
**Input:** 12V nominal (3.5–28V range)  
**Output:** 5.0V @ 1.5A continuous  

---

## Schematic

```
12V_IN ────┬────────────────────────────────┬──[R_EN: 100k]──┐
           │                                │                  │
      [C_BULK:      [TVS1:            ┌─────┴─────────────┐   │
      100µF/25V]   SMAJ15CA]         │  TPS54331DDAR     │   │
           │            │             │                   │   │
          GND          GND       VIN─┤1          EN ─────┼───┘
                                     │                   │
                  ┌──[C_BOOT:100nF]──┤BOOT        SS ───┤──[C_SS: 10nF]── GND
                  │                  │                   │
                  │         ┌────────┤SW         VFB ───┼──┬── GND
                  │         │        │                   │  │
                  │        [D1:      │       COMP ───────┼──┤
                  │       SS34]      └───────────────────┘  │
                  │         │ (anode GND, cathode SW)       │
                  │        GND                    ┌─────────┘
                  │                               │
                  └──[L1: 10µH/3A]──┬── 5V_OUT   │   COMP node
                                    │             ├──[C_HF: 47pF]── GND
                               [C_OUT:           │
                               2×22µF/10V]   [R_COMP: 100k]
                                    │             │
                                   GND        [C_COMP: 22nF]
                                                  │
                         5V_OUT ──[R_FB1: 52.3k]─┤ VFB
                                                  │
                                        [R_FB2: 10.0k]
                                                  │
                                                 GND

     Input cap: C_IN: 2×22µF/25V X5R placed at VIN and GND pins
```

---

## Component List

### Buck Converter

| Ref | Value / Part | Description |
|-----|-------------|-------------|
| U1 | TPS54331DDAR | Buck regulator IC, SOIC-8 PowerPAD |
| L1 | 10µH, 3A shielded | See inductor selection below |
| D1 | SS34 | Schottky catch diode, 3A / 40V, SMA. Anode → GND, Cathode → SW |
| C_IN | 2× 22µF / 25V X5R, 1210 | Input decoupling, close to VIN pin |
| C_BULK | 100µF / 25V electrolytic | 12V rail bulk cap, handles motor transients |
| TVS1 | SMAJ15CA | Bidirectional TVS, 15V clamp, 400W peak. Across 12V rail |
| C_OUT | 2× 22µF / 10V X5R, 1210 | Output filter caps |
| D1 catch diode → SW pin, SMA package. |  |  |
| R_FB1 | 52.3kΩ 1% | Feedback top — sets VOUT = 5.0V (VREF = 0.8V) |
| R_FB2 | 10.0kΩ 1% | Feedback bottom |
| C_BOOT | 100nF / 10V | Bootstrap cap, between BOOT and SW pins |
| R_COMP | 100kΩ | Compensation — see note below |
| C_COMP | 22nF | Compensation zero cap, in series with R_COMP |
| C_HF | 47pF | HF pole cap, in parallel with R_COMP + C_COMP |
| C_SS | 10nF | Soft-start — sets ~2ms startup ramp |
| R_EN | 100kΩ | EN pullup to VIN for always-on; replace with MCU GPIO drive if soft-enable needed |

**VOUT check:** VOUT = 0.8V × (1 + 52.3k / 10.0k) = 0.8 × 6.23 = **4.98V ≈ 5.0V** ✓

---

### Inductor Selection (L1)

**Value derivation**

Target ripple ratio 30% (ΔIL = 0.45A) — sufficient for stable current-mode control with acceptable peak current stress:

```
D = VOUT / VIN = 5 / 12 = 0.417
L = (VIN − VOUT) × D / (ΔIL × fSW)
  = 7V × 0.417 / (0.45A × 700,000Hz)
  = 9.3µH  →  10µH (nearest standard value)
```

**Current ratings required**

Peak current = IOUT + ΔIL/2 = 1.5 + 0.225 = **1.73A peak**

- Saturation current (Isat): must exceed 1.73A — below Isat the inductance collapses and current control is lost. Specify ≥ 2.5A for margin.
- DC current rating: thermal rating; must exceed 1.5A continuous. Lower DCR is better but efficiency is not a priority.

**Construction: shielded mandatory**

An unshielded inductor radiates from its winding at 700kHz. This will fail radiated emissions. Shielded drum-core construction is required.

**Candidate parts**

| Part | Isat | IDC | DCR | Size | ~Cost @ 1ku |
|------|------|-----|-----|------|-------------|
| **Bourns SRR6038-100Y** | 2.6A | 3.0A | 62mΩ | 6×6×3.8mm | ~$0.25 |
| Würth 744066100 | 3.0A | 3.0A | 65mΩ | 6×6×4.0mm | ~$0.30 |
| TDK VLS6045EX-100M | 2.4A | 2.3A | 88mΩ | 6×6×4.5mm | ~$0.20 |

The TDK VLS6045EX has only 0.67A headroom above the 1.73A peak — too tight. The Bourns SRR6038-100Y is the primary recommendation (0.87A headroom, well-characterised, widely stocked). The Würth 744066100 is the drop-in alternative with greater saturation margin.

The inductor cost ($0.25) is included in the $1.47 total BOM estimate.

**Compensation note:** Values above (R_COMP = 100kΩ, C_COMP = 22nF, C_HF = 47pF) are
starting-point estimates based on the TPS54331 datasheet example scaled for this operating
point. Verify with [TI WEBENCH](https://www.ti.com/design-resources/design-tools-simulation/webench-power-designer.html)
using VIN=12V, VOUT=5V, IOUT=1.5A, L=10µH, COUT=44µF before finalising.

---

### 3.3V LDO (from filtered 5V)

| Ref | Value / Part | Description |
|-----|-------------|-------------|
| U2 | AP2112K-3.3TRG1 | 3.3V, 600mA LDO, SOT-23-5 |
| C_LDO_IN | 10µF / 10V X5R | LDO input bypass |
| C_LDO_OUT | 10µF / 10V X5R | LDO output bypass (ceramic required for stability) |

Input: 5V_OUT. Output: 3.3V for MCU and quiet peripherals.  
P_loss at 200mA: (5 − 3.3) × 0.2 = **0.34W** — no heatsink needed.  
If 3.3V load > 500mA, substitute with a higher-current LDO (e.g., MIC5209-3.3YS).

---

### KP212 Pressure Sensor Interface

The KP212 is ratiometric: its output is proportional to its supply voltage. To preserve
ratiometric cancellation when the ADC reference (3.3V from LDO) is independent of the
sensor supply (5V), both the signal and ADC VREF must be derived from the same 5V rail
at the same ratio. This eliminates 5V regulation errors from the pressure reading.

```
5V_OUT ─────────────────────────┬── KP212 VCC
                                 │
                           [C_KP: 100nF]── GND   (KP212 supply bypass)

KP212 OUT ──[R_SIG1: 36.5kΩ]───┬──── ADC_IN (to MCU analog input)
                                 │
                          [R_SIG2: 100kΩ]
                                 │
                                GND

5V_OUT ──[R_VREF1: 51.1kΩ]─────┬──── MCU VREF+ pin
                                 │
                         [R_VREF2: 100kΩ]
                                 │
                         [C_VREF: 100nF]── GND
```

| Ref | Value | Description |
|-----|-------|-------------|
| C_KP | 100nF / 10V | KP212 VCC bypass, place at sensor pin |
| R_SIG1 | 36.5kΩ 1% | Signal divider top — scales 4.5V_max → 3.3V ADC |
| R_SIG2 | 100kΩ 1% | Signal divider bottom |
| R_VREF1 | 51.1kΩ 1% | VREF divider top — scales 5V → 3.3V |
| R_VREF2 | 100kΩ 1% | VREF divider bottom |
| C_VREF | 100nF | VREF bypass — filter noise from divider |

**Ratiometric check:**  
Signal at max pressure: 5V × 0.9 (sensor ratio) × 100k/(36.5k+100k) = 5 × 0.9 × 0.733 = **3.30V** ✓  
VREF: 5V × 100k/(51.1k+100k) = 5 × 0.662 = **3.31V** ✓  
Both scale identically with 5V → supply variation cancels in ADC reading. ✓

---

## 5V Distribution

```
U1 (buck) → 5V_OUT ──┬── [5V direct]  → future non-sensitive 5V loads
                      │
                      └── KP212 VCC
                      └── U2 (LDO) input → 3.3V → MCU
```

No separate LC point-of-load filter is needed for the KP212 — the sensor is insensitive
to 700kHz switching ripple (well above any pressure signal bandwidth), and the VREF
technique handles DC accuracy. A 100nF at the KP212 VCC pin is sufficient.

Add an LC filter (10µH + 47µF, f_c ≈ 7kHz) on the 5V rail only if a future sensitive
analog load requires it.

---

## PCB Layout Notes

1. **Switching loop:** Keep the path IC SW pin → D1 cathode → D1 anode (GND) → C_OUT → L1 → SW
   as small and tight as possible. This loop carries pulsed current at 700kHz and is the
   primary EMI source.

2. **Catch diode D1:** Place as close to the SW pin as possible. SMA footprint, short trace to GND.

3. **Inductor:** Place close to SW pin output, before C_OUT. Keep the SW node (IC to inductor)
   copper area small to reduce radiated EMI.

4. **Input caps C_IN:** Directly at VIN and AGND pins of IC.

5. **Feedback divider:** Place R_FB1/R_FB2 close to VFB pin. Route the VFB trace from the output
   capacitor side (quiet point), not the switch node side.

6. **Bootstrap cap C_BOOT:** Directly between BOOT and SW pins. Short trace.

7. **PowerPAD:** Connect the exposed pad on the SOIC-8 to the GND plane with multiple vias.
   Not needed for thermal (only ~0.5W dissipation), but provides good GND reference.

8. **TVS1 and C_BULK:** Place on the 12V input near the board power entry point, before
   the motor driver section and the buck converter input.

---

## Component Count Summary

| Section | Components |
|---------|-----------|
| Buck converter (U1 + passives + TVS) | 15 |
| 3.3V LDO (U2 + caps) | 3 |
| KP212 sensor interface | 6 |
| **Total** | **24** |

---

## Estimated BOM Cost at 1,000 Units

| Item | Est. cost/unit |
|------|---------------|
| TPS54331DDAR (U1) | $0.45 |
| AP2112K-3.3 (U2) | $0.10 |
| SS34 catch diode | $0.10 |
| SMAJ15CA TVS | $0.10 |
| Inductor L1 | $0.25 |
| Capacitors (all) | $0.30 |
| Resistors (all) | $0.05 |
| **Total** | **~$1.35** |
