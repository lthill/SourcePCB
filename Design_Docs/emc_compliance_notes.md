# EMC and Safety Compliance Notes

**Product:** Distillation Controller  
**Markets:** US primary (UL/FCC), EU limited (CE)  
**Relevant attributes:** Mains-connected, 100W heater (SSR), 12V brushed motors, MCU, 700kHz buck converter

---

## Applicable Standards

### CE (EU)

| Standard | Scope |
|----------|-------|
| **EN 55014-1** | Emissions — household and similar appliances with motors and heating elements |
| **EN 55014-2** | Immunity — same product category |
| **EN 61000-3-2** | Harmonic current emissions into mains (applies — load > 75W) |
| **EN 61000-3-3** | Voltage fluctuations and flicker (applies — switching 100W heater load) |
| **IEC/EN 62368-1** | Safety — replaces 60950-1 and 60065; covers audio/video/IT equipment |
| **2014/30/EU** | EMC Directive — requires DoC and CE marking |
| **2014/35/EU** | Low Voltage Directive — requires DoC and CE marking |

> If positioned and sold as a **laboratory instrument** rather than a household appliance,
> substitute EN 55011 (ISM equipment) or EN 61000-6-4 (industrial emissions) for EN 55014-1.
> EN 55014-1 is appropriate if sold to end users for home/small-scale use.

### US

| Standard | Scope |
|----------|-------|
| **FCC Part 15 Subpart B** | Unintentional radiator — required for any digital device sold in the US |
| **UL 61010-1** | Safety — measurement, control, and laboratory equipment |
| **ANSI C63.4** | US emissions measurement method (used with FCC Part 15B) |

FCC Part 15B can be satisfied by Supplier's Declaration of Conformity (SDoC) with in-house
or third-party testing — no FCC filing required for Class B devices.
UL listing requires testing by a Nationally Recognised Testing Laboratory (NRTL): UL, ETL (Intertek), or CSA.

---

## Emissions Limits

### Conducted Emissions (measured on mains cable, 150kHz–30MHz)

| Frequency band | CE Class B (EN 55014-1) | FCC Part 15B Class B |
|---------------|------------------------|----------------------|
| 150–500 kHz | 66 dBµV QP / 56 dBµV AVG | — |
| 500 kHz–5 MHz | 56 dBµV QP / 46 dBµV AVG | ~56 dBµV QP |
| 5–30 MHz | 60 dBµV QP / 50 dBµV AVG | ~60 dBµV QP |

**The 700kHz buck converter** switching frequency (and its harmonics at 1.4MHz, 2.1MHz…)
falls in the 500kHz–5MHz band — limit is **56dBµV quasi-peak**.

EN 55014-1 has relaxed limits in some motor noise bands compared to EN 55032, which helps
since the brushed motor commutation noise is broadband.

### Radiated Emissions (measured at 10m for CE, 3m for FCC)

| Frequency band | CE Class B (10m) | FCC Class B (3m) |
|----------------|-----------------|-----------------|
| 30–88 MHz | 30 dBµV/m | 40 dBµV/m |
| 88–230 MHz | 30 dBµV/m | ~43 dBµV/m |
| 230–1000 MHz | 37 dBµV/m | ~46 dBµV/m |

---

## EMI Sources: Priority Order

| Source | Severity | Status |
|--------|----------|--------|
| Brushed motor commutation | ~~High~~ Mitigated | Suppression caps + choke + twisted pair in place |
| SSR heater switching | ~~High~~ Low | Zero-crossing SSR on resistive load — no HF content in 150kHz–30MHz band |
| AC/DC converter (12V PSU) | Mitigated | Integrated mains filter on the converter handles its own switching emissions |
| Buck converter (700kHz) | **Medium** | **Primary remaining EMC design action — layout and shielded inductor** |
| MCU clock and switching | Low | Standard layout practice sufficient |

The mains-side picture is well-controlled. The buck converter at 700kHz is now the
dominant remaining conducted emissions contributor. Its 700kHz fundamental and harmonics
(1.4MHz, 2.1MHz…) must pass the 56dBµV quasi-peak limit (500kHz–5MHz band). With the
tight layout described in `buck_converter_circuit.md` and a shielded inductor, this is
routinely achievable.

---

## Required Design Elements

### 1. Mains Input Filter

**Status: handled by the AC/DC converter's integrated filter.**

The 12V AC/DC converter has an onboard mains filter. This handles conducted emissions from
the converter's own switching. Buck converter noise at 700kHz on the 12V rail must traverse
the AC/DC converter's power stage to reach the mains — the converter's input capacitance and
filter provide significant attenuation in this direction.

No additional external mains filter module is required unless pre-compliance testing reveals
a deficit. If one is needed, a Schaffner FN2060 or Würth 7427 series module (X2 cap + Y2
caps + common-mode choke, ~$2–5 at 1ku) can be added in series with the mains input.

> **Y-capacitors require a protective earth (PE) connection.** If the enclosure is plastic
> with no earth pin, a double-insulated design is required — no Y-caps, tighter creepage.

### 2. SSR Heater Switching

**Confirmed quiet for HF conducted emissions (150kHz–30MHz).**

A zero-crossing SSR turns on at zero voltage and turns off at zero current. There is no
high-dV/dt switching edge — the transition follows the mains sinewave. This produces no
significant content in the 150kHz–30MHz conducted emissions band.

The SSR's internal RC snubber passes a small leakage current (~mA) when off; this is
characterised in the SSR datasheet and is negligible for emissions testing.

EN 61000-3-2 (harmonic current limits, 50/60Hz domain) still applies because the 100W
load is above the 75W threshold. A resistive heater at unity power factor passes the
Class A harmonic limits by a wide margin — no corrective action needed; just document it.

An RC snubber (10Ω + 100nF in series) across the SSR output terminals is worthwhile to
damp inductive kick from heater wiring, but is not an EMC requirement for a resistive load.

### 3. Brushed Motor Noise Mitigation

**Status: complete.** Motors have suppression capacitors, a choke, and twisted-pair
wiring. No additional motor-side EMI work required.

Verify that flyback diodes are present on each FET output (drain to 12V supply, Schottky
SS34 or equivalent) for back-EMF clamping — this is a reliability requirement independent
of EMC.

### 4. Buck Converter Layout (700kHz)

See `buck_converter_circuit.md` for full layout guidance. Key requirements for compliance:

- Minimise the high-current switching loop: SW → D1 → GND → C_OUT → L1 → SW. Keep this
  loop area < 1cm² on the PCB.
- Use a shielded inductor (SRR-series or equivalent). An unshielded inductor radiates at
  700kHz and harmonics — this will fail radiated emissions.
- Keep SW-node copper area small. The SW node is the primary radiated noise antenna.
- Input decoupling (C_IN: 2×22µF) must be placed directly at the IC VIN pin.
- The 100µF bulk electrolytic on the 12V rail helps prevent motor switching transients from
  appearing as common-mode conducted emissions on the mains.

### 5. PCB Zoning and Ground Planes

```
┌───────────────────────────────────────────────────────────────┐
│  MAINS ZONE         │  DC POWER ZONE      │  LOGIC ZONE       │
│  (L, N, PE)         │  (12V, motors,      │  (5V, 3.3V,       │
│  [Mains filter]     │   heater SSR drive) │   MCU, sensor)    │
│  [AC/DC converter]  │  [Buck converter]   │   [KP212]         │
│                     │  [Motor FETs]       │                   │
│  ← 8mm min creepage →│                    │                   │
└───────────────────────────────────────────────────────────────┘
```

- **Single solid GND plane** on one PCB layer.
- DC power return currents (motors, SSR drive) must not route under the logic/sensor zone.
- Connect chassis ground (PE) to signal GND at **one point only** near the mains input
  connector. Multiple PE-to-signal connections create ground loops.
- Star-topology ground routing from the AC/DC converter GND output to all subsections.

---

## Safety: Creepage and Clearance (IEC/EN 62368-1)

For a PCB with 250VAC mains traces (L, N) and SELV traces (12V, 5V, 3.3V) on the same board:

**Assumptions:** Pollution Degree 2 (indoor, some condensation), Overvoltage Category II,
FR4 substrate (Material Group IIIa, CTI 175–249).

| Insulation type | Clearance | Creepage on FR4 |
|----------------|-----------|----------------|
| Basic (one layer of protection) | 1.5 mm | 4.0 mm |
| Reinforced (equivalent to double) | **3.2 mm** | **8.0 mm** |

**Use reinforced insulation distances** (8mm creepage, 3.2mm clearance) for all
primary-to-secondary boundaries. Basic insulation alone is not sufficient for a consumer-
accessible product without a second independent layer of physical protection.

**Practical rules:**
- 8mm minimum spacing between any mains trace and any SELV trace, measured along the PCB
  surface (creepage) and through air (clearance must be ≥ 3.2mm).
- Add a PCB **slot** (routed gap) at the mains-to-SELV boundary if 8mm cannot be maintained
  due to layout constraints. A slot eliminates the surface creepage path entirely.
- No traces or vias in the 8mm keep-out zone between mains and SELV domains.
- Mark the isolation boundary in silkscreen for assembly inspection.

> If a conformal coating is applied to the PCB, it may qualify as a barrier that shifts
> the Pollution Degree to PD1, reducing minimum distances. Verify with your coating supplier
> and test lab before relying on this.

---

## Immunity Requirements (CE — EN 55014-2)

| Test | Level | Standard |
|------|-------|----------|
| ESD contact discharge | ±4kV | IEC 61000-4-2 |
| ESD air discharge | ±8kV | IEC 61000-4-2 |
| Radiated RF immunity | 3 V/m, 80–1000MHz | IEC 61000-4-3 |
| EFT/Burst on mains | ±2kV | IEC 61000-4-4 |
| Surge on mains | 1kV differential, 2kV CM | IEC 61000-4-5 |
| Conducted RF immunity | 3 V(emf), 0.15–80MHz | IEC 61000-4-6 |
| Voltage dips/interruptions | Various levels | IEC 61000-4-11 |

**Design implications:**
- **ESD:** TVS diodes on all user-accessible connectors and sensor inputs. Adequate creepage
  around connectors. Metal enclosure improves ESD immunity significantly.
- **EFT/Burst:** The mains input filter handles most of this. The MOV on the 12V rail
  (TVS1: SMAJ15CA in the buck circuit) handles residual 12V rail transients.
- **Surge:** The mains filter MOV provides primary surge protection. The AC/DC converter's
  internal protection covers residual.
- **Voltage dips:** Firmware must handle graceful shutdown or hold-up during brief mains
  interruptions. A bulk capacitor on the 12V rail provides ~10–50ms hold-up depending on
  load.

---

## Harmonic Emissions: EN 61000-3-2

Applies because the **100W heater exceeds the 75W threshold**.

The standard limits harmonic currents injected into the mains (2nd through 40th harmonic).
A purely resistive heating element has very low harmonic content and is typically compliant
by default. The SSR (thyristor-based) introduces some phase-angle modulation but zero-crossing
switching minimises this.

**Action:** Confirm the AC/DC converter's input power factor. Switching power supplies
below ~75W are exempt from 61000-3-2. At 35W input, the AC/DC converter is below the
threshold. The heater is resistive. The combined harmonic profile should pass without
additional PFC circuitry.

---

## Testing Path and Budget

### Pre-Compliance (before formal submission)

| Test | Where | Est. cost |
|------|-------|----------|
| Conducted emissions scan | Pre-comp lab or in-house with spectrum analyser + LISN | $500–$2,000 |
| Radiated emissions scan | Pre-comp lab (not full anechoic needed) | $500–$2,000 |
| ESD spot check | Manual gun testing, identify weaknesses | $200–$500 |

Run pre-compliance before committing to the production PCB. Most compliance failures can
be corrected with component changes (filter values, ferrite specifications) that do not
require a new board spin.

### Formal Testing

| Test suite | Est. cost | Notes |
|------------|----------|-------|
| FCC Part 15B (conducted + radiated) | $2,500–$5,000 | Required for US market. SDoC path — no filing fee. |
| CE Emissions (EN 55014-1) | $2,000–$4,000 | Can be combined with FCC at same lab |
| CE Immunity (EN 55014-2) | $3,000–$5,000 | Required for CE DoC |
| CE Safety (EN 62368-1 / LVD) | $3,000–$8,000 | Required for CE DoC |
| UL 61010-1 listing | $5,000–$15,000 | Required for UL mark in US |
| **CE only (FCC + emissions + immunity + safety)** | **$10,000–$22,000** | Ballpark first-pass |
| **Full CE + UL** | **$15,000–$37,000** | Strongly dependent on first-pass success |

**First-pass failure is expensive** — retesting costs the same as the first test. Pre-compliance
testing pays for itself if it catches one failure.

### Practical Path for This Product

Given limited EU volume:

1. **Do FCC Part 15B.** Required regardless. Self-declaration (SDoC) is accepted — no FCC
   approval letter needed. Test at an accredited lab, keep the test report on file.

2. **Do CE EMC (EN 55014-1 + EN 55014-2).** Even for a few EU units, a DoC is a legal
   requirement if product is placed on the EU market. Can be combined with FCC testing at
   the same lab session, marginal cost to add.

3. **Defer UL listing** unless a large US customer specifically requires it or you enter
   retail. UL listing is expensive and typically required only by retailers, large
   commercial buyers, or certain regulated markets. Most small-volume industrial/lab
   sales do not require it — only FCC compliance is legally mandated in the US.

4. **CE Safety (LVD)** — required for the CE mark. If budget is constrained, prioritise
   this for the EU units. Some test houses offer combined EMC + safety packages.

---

## Sources

- [TI: Conducted Emissions Review (SSZT673)](https://www.ti.com/lit/pdf/sszt673)
- [TI: Creepage and Clearance for IEC 62368-1 (SLUP419)](https://www.ti.com/lit/pdf/slup419)
- [EN 55032 / CISPR 32 summary (Electronics Notes)](https://www.electronics-notes.com/articles/analogue_circuits/emc-emi-electromagnetic-interference-compatibility/cispr22-en55022-standard.php)
- [FCC Part 15B testing overview (MiCOM Labs)](https://micomlabs.com/deep-dive-into-fcc-unintentional-radiator-testing/)
- [UL 61010-1 compliance guide (Keystone Compliance)](https://keystonecompliance.com/ul-61010/)
- [IEC 62368-1 design changes (Perimattic)](https://perimattic.com/iec-62368-1-design-guide/)
- [EMC testing cost guide (Celectronics)](https://www.celectronics.com/Resources/EMC-Testing-Cost-Guide)
