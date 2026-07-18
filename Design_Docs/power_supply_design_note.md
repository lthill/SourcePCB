# Power Supply Design Note
## Distillation Controller — 12V → 5V / 3.3V Rail

**Revision:** A  
**Date:** 2026-06-15  
**Volume:** ~500–1,000 units  
**Build cost context:** ~$80/unit at 500 units

---

## 1. System Requirements

| Parameter | Specification |
|-----------|---------------|
| Input | 12V DC from onboard 35W mains-connected AC/DC converter |
| Output 1 | 5V @ 1.5A — sensor supply and future loads |
| Output 2 | 3.3V — MCU and quiet peripherals |
| Sensor | Infineon KP212 absolute pressure sensor (4.5–5.5V, ratiometric analog output) |
| Noise sensitivity | 3.3V and sensor supply must be quiet |
| Efficiency | Not a priority — 100W mains heater dominates system power consumption |
| Regulatory | FCC Part 15B (US, required); CE (EU, limited volume) |

**Other loads on the 12V rail:**
- 12V brushed DC motors (low-side FET switched, with suppression capacitors, common-mode choke, and twisted-pair wiring)
- SSR-controlled mains heater (zero-crossing, 100W resistive load)
- Three 5V relay coils being replaced with low-side FET drives (relay loads eliminated from 5V budget)

---

## 2. Architecture Decision: Custom PCB, Not Modules

Off-the-shelf buck converter modules were evaluated and rejected for this application.

At 500–1,000 units, the BOM cost of a quality module (Pololu D24V22F5: ~$7–10/unit) compared to an on-board circuit (~$1.35/unit) represents a $3,000–9,000 cost difference for the run. Additionally, modules require mechanical mounting and wire harnesses, add assembly failure modes, are not characterised for this noise environment, and introduce supply chain inconsistency across a production run.

A pre-certified module does not certify the appliance. System-level FCC and CE testing is required regardless of what sub-components carry their own marks. The on-board circuit is certified as part of the system.

**Decision: buck converter integrated onto the main PCB.**

---

## 3. Power Architecture

```
Mains ── [AC/DC converter, 35W, integrated mains filter] ── 12V rail
                                                                 │
                              ┌──────────────────────────────────┤
                              │                                  │
                    [470µF bulk cap]                     [SMAJ15CA TVS]
                         [12V rail]
                              │
                    ┌─────────┴──────────────┐
                    │                        │
             [TPS54331DDAR]          [Motor FET drivers]
             [12V → 5V, 700kHz]      [suppression in place]
                    │
               [5V rail]
               /        \
    [KP212 sensor]    [AP2112K-3.3 LDO]
    [5V supply]       [5V → 3.3V]
                            │
                     [MCU + peripherals]
```

The 5V rail serves the KP212 sensor directly and provides the input to the 3.3V LDO. Future non-sensitive 5V loads can tap the rail directly. A point-of-load LC filter (10µH + 47µF) can be added for any future analog-sensitive 5V loads; it is not required for the KP212 (see Section 5).

---

## 4. Buck Converter: TPS54331DDAR

### Why TPS54331

The TPS54331DDAR was chosen over the initial candidate (TPS563201DDCR) specifically for its **fixed-frequency voltage-mode PWM** operation at 700kHz.

The TPS563201 uses D-CAP2 (constant on-time) control. Although it runs at approximately 580kHz under load, it enters Eco-mode pulse-skipping at light load — producing variable-frequency switching that can generate audio-band noise from capacitors and is harder to filter because the ripple frequency is not predictable. With the relay loads eliminated from the 5V rail, the minimum 5V load is low enough (~10–30mA) that pulse-skipping would occur regularly.

The TPS54331 switches at exactly 700kHz regardless of load. The output ripple is always at one predictable frequency; the LC filter is optimised for one target; and the conducted emissions spectrum shows a known spike at 700kHz plus harmonics, which is straightforward to characterise and attenuate.

Note: the TPS54331 is **non-synchronous** — it requires an external Schottky catch diode (D1). This is a single additional component and is not a meaningful cost or complexity concern.

### Key Specifications

| Parameter | Value |
|-----------|-------|
| Input voltage range | 3.5–28V |
| Output voltage | 5.0V (set by resistor divider) |
| Maximum output current | 3A |
| Switching frequency | 700kHz, fixed |
| Control mode | Current-mode PWM |
| Package | SOIC-8 with PowerPAD |
| IC price @ 1,000 units | ~$0.45 |

### Output Voltage Setting

VOUT = VREF × (1 + R_FB1 / R_FB2) = 0.8V × (1 + 52.3kΩ / 10.0kΩ) = **4.98V ≈ 5.0V**

---

## 5. Component List

### Buck Converter (12V → 5V)

| Ref | Value / Part | Notes |
|-----|-------------|-------|
| U1 | TPS54331DDAR | IC, SOIC-8 PowerPAD |
| L1 | 10µH, 3A shielded | Bourns SRR6038-100Y or Würth 744066100. Shielded mandatory for EMC. |
| D1 | SS34 | Schottky catch diode, 3A/40V, SMA. Anode → GND, Cathode → SW pin. |
| C_IN | 2× 22µF / 25V X5R, 1210 | Input decoupling, close to VIN pin |
| C_BULK | 100µF / 25V electrolytic | 12V rail bulk — placed at board power entry |
| TVS1 | SMAJ15CA | Bidirectional TVS, 15V clamp, 400W peak. Across 12V rail. |
| C_OUT | 2× 22µF / 10V X5R, 1210 | Output filter |
| R_FB1 | 52.3kΩ 1% | Feedback top — sets VOUT = 5.0V |
| R_FB2 | 10.0kΩ 1% | Feedback bottom |
| C_BOOT | 100nF / 10V | Bootstrap, between BOOT and SW pins |
| R_COMP | 100kΩ | Compensation — verify with WEBENCH |
| C_COMP | 22nF | Compensation zero |
| C_HF | 47pF | Compensation HF pole |
| C_SS | 10nF | Soft-start (~2ms ramp) |
| R_EN | 100kΩ | EN pullup to VIN; replace with MCU GPIO for controlled enable |

### 3.3V LDO

| Ref | Value / Part | Notes |
|-----|-------------|-------|
| U2 | AP2112K-3.3TRG1 | 3.3V, 600mA, SOT-23-5 |
| C_LDO_IN | 10µF / 10V X5R | LDO input bypass |
| C_LDO_OUT | 10µF / 10V X5R | LDO output (ceramic required for stability) |

Power loss at 200mA MCU load: (5.0 − 3.3) × 0.2A = **0.34W** — no heatsink needed.  
Upgrade to MIC5209-3.3YS if 3.3V load exceeds 500mA.

### KP212 Sensor Interface

| Ref | Value | Notes |
|-----|-------|-------|
| C_KP | 100nF / 10V | VCC bypass at KP212 supply pin |
| R_SIG1 | 36.5kΩ 1% | Signal divider top — scales 4.5V max output → 3.3V ADC |
| R_SIG2 | 100kΩ 1% | Signal divider bottom |
| R_VREF1 | 51.1kΩ 1% | VREF divider top — scales 5V → 3.3V MCU VREF pin |
| R_VREF2 | 100kΩ 1% | VREF divider bottom |
| C_VREF | 100nF | VREF bypass |

**Total: 24 components. Estimated BOM cost at 1,000 units: ~$1.35/unit.**

---

## 6. KP212 Pressure Sensor Supply and Interface

The KP212 requires 4.5–5.5V and cannot be powered from the 3.3V LDO. It is powered from the 5V rail directly with a 100nF bypass cap at its VCC pin.

### Noise Sensitivity

The KP212 measures pressure — a slow-moving physical quantity. Its useful signal bandwidth is DC to a few Hz. The buck converter's 700kHz switching ripple is orders of magnitude above this bandwidth and averages to zero over any reasonable ADC sampling period. The sensor does not require a point-of-load LC filter; a single 100nF decoupling cap at the supply pin is sufficient.

### Ratiometric Accuracy

The KP212 output is ratiometric — it scales proportionally with its supply voltage. The output range is 0.5–4.5V for 15–115kPa (at 5V supply). The MCU ADC uses a 3.3V reference, which is on an independent rail (the LDO). With a simple voltage divider to scale the sensor output into the ADC range, 5V supply variations directly produce reading errors — the ratiometric benefit is broken.

**Solution:** Derive the MCU VREF from the same 5V rail at the same divider ratio. Both the signal path and the reference then scale identically with the 5V rail, and supply variations cancel in the ADC result.

```
5V ── [R_VREF1: 51.1kΩ] ──┬── MCU VREF+ pin
                            │
                   [R_VREF2: 100kΩ]  [C_VREF: 100nF]
                            │
                           GND

KP212 OUT ── [R_SIG1: 36.5kΩ] ──┬── ADC_IN
                                  │
                         [R_SIG2: 100kΩ]
                                  │
                                 GND
```

Both dividers reduce the 5V rail to 3.3V (ratios 0.662 and 0.733 respectively — the signal is scaled to fill the ADC range at full pressure). Supply variation on the 5V rail appears identically in both signal and reference, cancelling in the ADC reading. The pressure measurement is then independent of 5V regulation accuracy.

---

## 7. PCB Layout Requirements

### Switching Loop (critical for EMC and performance)

The high-current switching loop — **SW pin → D1 (catch diode) → GND → C_OUT → L1 → SW** — must be as small and compact as possible. This loop carries pulsed current at 700kHz and is the primary EMI source on the board. Target loop area < 1cm².

- Place D1 (SS34) immediately adjacent to the SW pin, SMA footprint, short trace to GND plane
- Place L1 output side directly at the output capacitors
- Keep the SW node copper area small — it is an antenna at 700kHz
- Place C_IN directly at the VIN and GND pins of U1
- Place C_BOOT between BOOT and SW pins with minimum trace length
- Place feedback resistors R_FB1/R_FB2 close to the VFB pin; route VFB from the quiet side of C_OUT, not the switch node

### Ground Plane

- Solid GND copper pour on one layer across the entire board
- Motor drive return currents must not pass under the sensor or MCU sections
- Connect chassis PE to signal GND at one point only, near the mains input connector
- Thermal vias under the U1 PowerPAD to GND plane (3×3 array, 0.3mm drill)

### Board Zoning

```
┌─────────────────┬──────────────────────┬─────────────────────┐
│  MAINS ZONE     │  DC POWER ZONE       │  LOGIC / SENSOR     │
│  AC/DC conv.    │  12V rail            │  5V / 3.3V          │
│  Mains filter   │  Buck converter      │  MCU                │
│  SSR heater     │  Motor FET drivers   │  KP212 sensor       │
└─────────────────┴──────────────────────┴─────────────────────┘
         ← 8mm creepage boundary →
```

---

## 8. Safety: Creepage and Clearance

Applicable standard: **IEC/EN 62368-1**, Pollution Degree 2, Overvoltage Category II, 250VAC working voltage, FR4 substrate (Material Group IIIa).

| Insulation class | Clearance | Creepage on FR4 |
|-----------------|-----------|----------------|
| Reinforced (mains to SELV) | **3.2mm** | **8.0mm** |

All mains traces (L, N) must maintain 8mm minimum spacing along the PCB surface and 3.2mm through air to any SELV conductor (12V, 5V, 3.3V, signal).

Add a **PCB slot** (routed gap) at the mains-to-SELV isolation boundary if the 8mm creepage distance cannot be maintained in the layout. Mark the isolation boundary in silkscreen.

No vias, traces, or copper within the 8mm keep-out zone between mains and SELV domains.

---

## 9. EMC Status

### Conducted and Radiated Emissions Sources

| Source | Status | Action required |
|--------|--------|-----------------|
| Brushed motors | ✅ Mitigated | Suppression caps + choke + twisted pair in place |
| AC/DC converter | ✅ Mitigated | Integrated mains filter on the converter |
| SSR heater (zero-crossing, resistive) | ✅ Clean | No HF content in 150kHz–30MHz band |
| **Buck converter 700kHz** | ⚠️ Open | Shielded inductor + tight SW loop layout |
| MCU/clock | ✅ Low risk | Standard layout practice |

The buck converter at 700kHz is the primary open EMC item. Its fundamental and harmonics (1.4MHz, 2.1MHz…) fall in the 500kHz–5MHz conducted emissions band (limit: 56dBµV quasi-peak for Class B). A shielded inductor and tight switching loop layout are the principal controls. Fixed-frequency operation means the emissions spectrum is predictable and repeatable — pre-compliance measurement directly shows whether the design passes.

### Applicable Emissions Limits

| Band | CE Class B (QP / AVG) | FCC Part 15B Class B |
|------|-----------------------|----------------------|
| 150–500kHz | 66 / 56 dBµV | — |
| 500kHz–5MHz | 56 / 46 dBµV | ~56 dBµV QP |
| 5–30MHz | 60 / 50 dBµV | ~60 dBµV QP |

### Harmonic Current Emissions (EN 61000-3-2)

Applies: 100W heater exceeds the 75W threshold. Zero-crossing SSR on a resistive heater produces near-unity power factor with negligible harmonic current. Expected to pass Class A limits by a large margin without corrective action.

---

## 10. Compliance Path

### United States — FCC Part 15 Subpart B (required)

- Class B device (intended for residential/commercial use)
- **Supplier's Declaration of Conformity (SDoC)** — no FCC filing required
- Test at an accredited lab; keep test report on file
- Covers conducted emissions (150kHz–30MHz on mains) and radiated emissions (30MHz–1GHz)
- Estimated test cost: $2,500–$5,000

### European Union — CE Marking

Applicable directives: **EMC Directive 2014/30/EU** + **Low Voltage Directive 2014/35/EU**

| Standard | Scope | Notes |
|----------|-------|-------|
| EN 55014-1 | Emissions | Household/similar appliances with motors and heating — appropriate category |
| EN 55014-2 | Immunity | ESD, EFT, surge, conducted immunity |
| EN 61000-3-2 | Harmonic currents | Applies >75W; expected to pass by default |
| EN 61000-3-3 | Voltage fluctuations | Heater switching at reasonable cycle rates |
| IEC/EN 62368-1 | Safety (LVD) | Creepage, clearance, insulation, fusing |

CE can be self-declared (Declaration of Conformity) supported by test reports. A notified body is required only for specific product categories that include explicit notified body mandates — this product does not appear to be in that category, but confirm with the test lab.

**Recommended path:** Combine FCC and CE emissions testing in one lab session — marginal cost to add CE testing when already set up for FCC. This is cost-effective even for limited EU volume since the legal requirement exists for any unit placed on the EU market.

### Testing Budget Estimate

| Test | Estimated cost |
|------|---------------|
| Pre-compliance scan (conducted + radiated) | $500–$2,000 |
| FCC Part 15B (conducted + radiated, formal) | $2,500–$5,000 |
| CE Emissions EN 55014-1 (combined with FCC) | +$1,000–$2,000 incremental |
| CE Immunity EN 55014-2 | $3,000–$5,000 |
| CE Safety EN 62368-1 (LVD) | $3,000–$8,000 |
| UL 61010-1 (if required by customer) | $5,000–$15,000 |
| **Minimum viable (FCC + CE EMC only)** | **~$6,000–$12,000** |
| **Full CE + FCC (all above excl. UL)** | **~$10,000–$20,000** |

First-pass failure is as expensive as the first test. Pre-compliance testing before committing to the final board spin is strongly recommended.

---

## 11. Open Items

| Item | Action |
|------|--------|
| Compensation values | Run TI WEBENCH (VIN=12V, VOUT=5V, IOUT=1.5A, L=10µH, COUT=44µF) — verify R_COMP, C_COMP, C_HF before finalising |
| MCU VREF pin | Confirm MCU has dedicated VREF+ input; if not, assess 5V regulation accuracy vs. required pressure measurement tolerance |
| 3.3V load current | Confirm total 3.3V load ≤ 500mA for AP2112K; if higher, upsize LDO |
| Pump voltage | Confirm pumps are 12V DC (FET low-side drive appropriate) or mains AC (SSR required) |
| Flyback diodes on FETs | Confirm SS34 or equivalent on all motor FET drain-to-12V supply |
| PCB slot at mains boundary | Include routed slot in PCB layout to enforce 8mm creepage |
| SSR snubber | Add 10Ω + 100nF in series across SSR output terminals (heater wiring inductance) |
| Pre-compliance testing | Schedule before production PCB order; correct any conducted emissions issues at 700kHz before locking layout |

---

## 12. Reference Documents

| Document | Contents |
|----------|----------|
| `buck_converter_trade_study.md` | Full IC comparison, cost analysis, volume rationale |
| `buck_converter_circuit.md` | Schematic description, complete component list, layout notes |
| `emc_compliance_notes.md` | Detailed standards reference, immunity requirements, testing guidance |
