# Buck Converter Trade Study: 12V → 5V @ 1.5A

**Project:** Distillation Controller  
**Date:** 2026-06-14  
**Volume:** 1,000 units  
**Decision:** Off-the-shelf module vs. custom PCB design

---

## System Context

| Parameter | Detail |
|-----------|--------|
| Input to board | 12V DC from mains-connected AC/DC converter |
| Required output | 5V @ 1.5A (sensor + future loads; headroom is generous — see below) |
| Sensor | Infineon KP212 absolute pressure sensor (4.5–5.5V supply, ratiometric analog output) |
| 5V load note | Original design had 3× 5V relays (180–300mA coil load). These are being replaced with low-side N-channel FET drives from 3.3V MCU GPIOs. Gate drive current is negligible. Minimum 5V load drops to ~10–30mA (sensor + LDO quiescent); 1.5A budget is headroom for future loads. |
| Other voltages | 3.3V for MCU and peripherals (must be quiet) |
| Noise environment | **High** — mains AC switching on PCB; 12V brushed DC motors switched on PCB |
| Application | Appliance (mains-connected → regulatory certification required) |
| Volume | ~500–1,000 units (current build quote: ~$80/unit at 500) |
| Efficiency priority | **Low** — board also switches a 100W mains heater; buck converter losses (~0.5W) are insignificant |
| Regulatory | CE required only for EU; minimal EU volume. Pre-certified AC/DC modules do not certify the appliance — system-level EMC testing required regardless. Dominant EMC sources are the triac/SSR and brushed motors, not the buck converter. |

**Off-the-shelf modules are eliminated at this volume.** The analysis below documents why, then focuses on the custom-PCB IC choice.

---

## Off-the-Shelf Modules: Why Not at 1,000 Units

| Factor | Impact |
|--------|--------|
| Unit cost at 1ku | $5.75–$10 (Pololu) vs. ~$1.00 BOM for custom — ~$4,750–$9,000 more for the run |
| Assembly | Each module requires mechanical mounting, connectors, and wire harness → adds labor and failure modes |
| Regulatory certification | Appliance certification requires traceable, documented components; generic modules introduce risk |
| Noise environment | Modules are not characterized for use with brushed motor commutation noise on their input rail |
| Supply chain | Module availability and consistency cannot be guaranteed across a 1ku production run |

**Conclusion: Build the buck converter directly onto the main PCB.**

---

## Noise Sources and Priority

**12V rail:** Motor spikes are handled by the motor driver circuit (low-side FET with flyback protection). The 12V rail is therefore reasonably clean; no separate TVS is required for the buck converter input. Standard bulk capacitance near the motor drivers handles the remaining transient current.

**5V output rail:** This is the primary noise concern. The buck converter's switching frequency (400–580kHz depending on IC choice) produces output voltage ripple. Certain downstream loads — particularly the sensor — require a clean supply. The architecture must provide a clean 5V branch for sensitive loads while allowing non-sensitive loads to tap the raw 5V bus directly.

**3.3V rail:** Derived from 5V via LDO. The LDO's PSRR (typically 60–80dB) provides additional attenuation of any residual 5V ripple before it reaches the MCU and quiet peripherals.

---

## IC Candidate Comparison

### Candidate 1: TPS54331DDAR (Texas Instruments) — **Recommended**

| Parameter | Value |
|-----------|-------|
| Input voltage max | 28V |
| Output current | 3A |
| Switching frequency | 700 kHz — **truly fixed frequency (voltage-mode PWM)** |
| Package | SOIC-8 |
| IC price @ 1 unit | ~$0.56 |
| IC price @ 1,000 units | **~$0.40–$0.50** |
| External components | 1 inductor + 4 caps + 2 resistors + compensation RC |
| Approx. full BOM cost @ 1ku | ~$1.00–$1.20 |
| Control mode | Voltage-mode PWM — fixed frequency at all load levels |

**Why fixed frequency matters here:** The TPS54331 switches at exactly 700kHz regardless of
load. The output ripple is always at one predictable frequency, so the LC point-of-load filter
can be precisely tuned to attenuate it. Harmonic content in the conducted emissions spectrum is
also predictable, which helps pre-compliance EMC testing.

**Comparison to TPS563201DDCR ($0.21/unit @ 1ku):** The TPS563201 uses D-CAP2 — a
constant on-time architecture that runs at *approximately* 580kHz under normal load but
**pulse-skips at light load (Eco-mode)**. Pulse skipping produces variable-frequency ripple
that can include audio-band components (<20kHz) — audible from ceramic capacitors and harder
to filter. If the 5V rail always carries a meaningful minimum load (sensor + LDO quiescent
≥ ~50mA), the TPS563201 stays out of Eco-mode and behaves as approximately fixed-frequency.
If loads could drop to near-zero (sleep mode, sensor off), fixed-frequency PWM is preferable.

The TPS54331 requires one additional passive for the external compensation network (an RC);
TI WEBENCH generates the values automatically.

**28V input** also provides comfortable headroom above the 12V rail, even with motor driver
protection in circuit.

---

### Candidate 2: LMR33630ADDAR (Texas Instruments)

| Parameter | Value |
|-----------|-------|
| Input voltage max | 36V |
| Output current | 3A |
| Switching frequency | ~400 kHz |
| Package | SOIC-8 |
| IC price @ 1 unit | $2.99 |
| IC price @ 1,000 units | **~$1.85** (Arrow) |
| External components | 1 inductor + 3–4 caps + 2 resistors |
| Approx. full BOM cost @ 1ku | ~$2.50–$2.75 |
| EMC documentation | TI App Note SNVA859A (CISPR-22 compliance) |

The 36V input tolerance is the main selling point. With motor spikes handled at the driver, this
advantage is reduced — the $1,640 premium for the run is harder to justify purely on input
headroom. The SNVA859A EMC application note remains useful for certification, but the
TPS563201 has similar EMI characteristics with proper layout and output filtering.

Choose the LMR33630 if: the motor driver protection design cannot be fully verified before
PCB fabrication, or if future motor variants with higher inductance are planned.

---

### Candidate 3: LM2596 — Eliminated

| Reason | Detail |
|--------|--------|
| Switching frequency | 150 kHz — large, expensive inductors; frequency falls in conducted emissions band |
| Non-synchronous | Requires external Schottky diode → more BOM, worse output ripple |
| IC cost at 1ku (TI) | ~$3+ — expensive for what it delivers |
| Counterfeiting risk | Widely cloned; real TI parts expensive, Chinese alternatives unreliable |

**Do not use for new designs.**

---

## Cost Comparison at 1,000 Units

| Approach | IC cost @ 1ku | Total BOM/unit | 1ku total BOM | Control mode | Notes |
|----------|--------------|----------------|---------------|--------------|-------|
| **TPS54331DDAR + TVS** | **~$0.45** | **~$1.20** | **~$1,200** | **Fixed-freq PWM** | **Recommended** |
| TPS563201DDCR + TVS | $0.21 | ~$1.00 | ~$1,000 | D-CAP2 (pseudo-fixed) | Valid if min load ≥ 50mA |
| LMR33630ADDAR | $1.85 | ~$2.65 | ~$2,650 | Fixed-freq PWM | If motor protection unverified |
| LM2596 (TI) | ~$3.00+ | ~$4.00+ | ~$4,000+ | Fixed-freq PWM | Eliminated — cost, EMC, BOM |

**TPS54331 vs TPS563201:** ~$200 more for the run buys truly fixed-frequency operation and
28V input headroom. For a filterable, predictable 5V supply in a sensor application, that is
worth the extra $0.20/unit. Use TPS563201 if minimum load is guaranteed non-zero and every
cent matters.

---

## Recommended Power Architecture

```
Mains → AC/DC PSU → 12V rail
                        │
          ┌─────────────┼──────────────────────┐
          │             │                      │
    [470µF bulk]  [SMAJ15CA TVS]      [Motor drivers]
          │             │              [SS34 flyback diodes]
          └─────────────┘
                        │
               [TPS563201DDCR]
               [12V → 5V @ 1.5A]
                        │
               ─────────────────── 5V raw bus
               │                 │
    [Non-sensitive 5V loads]  [L_filter + C_filter]   ← point-of-load filter
                               │
                    ─────────────────── 5V filtered
                    │                 │
              [Sensor]       [AP2112K-3.3 LDO]
                             [5V → 3.3V]
                                   │
                            [MCU + peripherals]
```

### 5V Noise Filtering and the KP212 Ratiometric Issue

The KP212 is a ratiometric pressure sensor (4.5–5.5V supply, output range 0.5–4.5V).
Its output voltage is proportional to its supply voltage — this is the ratiometric property.

**What "quiet" means for this sensor:**

- **High-frequency ripple (700kHz from the buck):** Irrelevant. The pressure ADC samples at
  Hz-to-kHz rates; hundreds of thousands of switching cycles average out in every reading.
  A 100nF decoupling cap directly at the KP212 supply pin handles the rest.

- **DC regulation accuracy of 5V:** This is what matters. The KP212 output is proportional
  to its 5V supply, but the MCU ADC reference is 3.3V (from the LDO) — an independent rail.
  A 1% droop on 5V produces a ~1% error in the pressure reading. The ratiometric cancellation
  that makes this sensor insensitive to supply variation is broken when the signal crosses
  voltage domains.

**The correct fix — restore ratiometric cancellation in the ADC:**

If the MCU has a dedicated VREF pin (common on STM32, RP2350, etc.), derive it from the
**same 5V rail at the same divider ratio** used to scale the sensor output:

```
5V rail ──┬── KP212 supply
          │
          ├── R_top ──┬── KP212 output ── R1 ──┬── ADC input
          │           │                         │
          │           └── (sensor signal)    R2 ┤
          │                                     └── GND
          │
          └── R_vref_top ──┬── MCU VREF+
                           │
                        R_vref_bot
                           │
                          GND
```

Set R_vref_top / R_vref_bot = R1 / R2 (same ratio as the signal divider). Both the
signal and the reference then scale identically with the 5V supply — variations cancel
completely in the ADC result, and the measurement is independent of 5V regulation accuracy.

If the MCU has no separate VREF pin and uses VDD (3.3V) as its reference, ensure the
5V buck DC regulation is ≤1% (standard for any quality synchronous buck) and budget the
corresponding pressure accuracy accordingly.

**The point-of-load LC filter** (10µH + 47µF, f_c ≈ 7kHz, ~40dB at 700kHz) is still
included as good practice and to serve future 5V loads, but it is not critical for the KP212.

### Why LDO for 3.3V

- LDO PSRR (60–80 dB) provides a second stage of filtering for the MCU rail — no design effort required
- Power loss at 200mA MCU load: (5V − 3.3V) × 0.2A = **0.34W** — no heatsink needed
- AP2112K-3.3 (~$0.10/unit at 1ku) or MCP1700-3302 (~$0.07/unit) — both are very low-noise LDOs
- If 3.3V load exceeds ~400mA, use a second small synchronous buck (e.g., TPS562201) instead

---

## LMR33630 Design Parameters (12V → 5V @ 1.5A)

Use [TI WEBENCH Power Designer](https://www.ti.com/design-resources/design-tools-simulation/webench-power-designer.html) to generate and verify the complete schematic and component values.

| Component | Recommended value | Notes |
|-----------|------------------|-------|
| L1 | 15µH, 2A, shielded | Lower freq → larger inductor vs. TPS563201 |
| C_IN | 22µF / 50V ceramic + 10µF | High voltage rating for 12V rail transients |
| C_OUT | 22µF + 10µF ceramic | X7R, 10V+ rating |
| R_TOP | 100kΩ | Feedback divider top |
| R_BOT | 28.7kΩ | Sets 5.0V output (V_REF = 1.0V) |
| C_BOOT | 100nF | Bootstrap capacitor |

**Thermal check:**
At any realistic efficiency, P_loss < 1W. Negligible in SOIC-8 on copper pads. No heatsink needed.
The 100W heater load dominates system power consumption; buck converter losses are not a design concern.

---

## PCB Layout and EMC Guidance

This board combines mains switching, motor drives, and a sensitive sensor — layout discipline is critical for both regulatory compliance and reliable operation.

**Zoning:**
- Divide the PCB into three physical zones: (1) AC mains / heater control, (2) motor drives + 12V power, (3) DC logic / sensor / analog
- Zone separations should be enforced by physical distance and PCB slot cuts where clearance is required by safety standards (IEC 60950 / IEC 62368)

**Buck converter layout:**
- Keep the switching loop tight: LMR33630 SW pin → inductor → output cap → GND → back to IC GND
- Place input cap directly at VIN and GND pins
- Do not route sensitive analog signals under or near the switch node copper

**Output filtering for sensor:**
- Add a ferrite bead (e.g., BLM31PG601SN1L, 600Ω @ 100MHz) in series with the 5V sensor supply
- Follow with a 10µF ceramic cap to ground after the ferrite bead
- This provides a low-pass filter against any residual 400kHz switching ripple

**Ground plane:**
- Use a solid ground plane on one layer across the board
- Motor driver and AC switch current returns should not pass under the analog/sensor section
- Return currents from motors and AC switch flow on the bottom copper layer; logic and sensor returns on a separate segment tied at one star point near the AC/DC input connector

**AC mains section:**
- Enforce creepage and clearance per IEC 62368-1 (typically 6mm across isolation boundary for 230V mains)
- Use a PCB slot or routed gap at the isolation boundary
- Mark the isolation boundary clearly in silkscreen for assembly inspection

---

## Regulatory Considerations

At 1,000 units, appliance certification is required:

| Standard | Requirement |
|----------|-------------|
| IEC/EN 62368-1 | Safety for audio/video and IT equipment (likely applicable) |
| CISPR 32 / EN 55032 | Conducted and radiated emissions (Class B limits) |
| IEC 61000-4-2 | ESD immunity |
| IEC 61000-4-4 | EFT/Burst immunity — particularly relevant with mains switching on board |
| IEC 61000-4-5 | Surge immunity — 1kV/2kV on AC mains |

The LMR33630's TI app note (SNVA859A) provides a documented path to CISPR-22/32 conducted emissions compliance, which directly supports pre-compliance testing. This is a non-trivial advantage over undocumented alternatives when preparing for a notified-body EMC test.

The AC switching section (triacs/SSRs) and brushed motor drives are the dominant EMC sources on this board. Budget time and PCB area for adequate filtering and isolation at the schematic stage.

---

## Summary Recommendation

| Decision point | Choice |
|----------------|--------|
| Off-the-shelf vs. custom PCB | **Custom PCB — required at 1ku** |
| Primary IC | **TPS54331DDAR** — fixed-frequency PWM, 28V input, ~$0.45/unit @ 1ku |
| If min 5V load is always ≥50mA | TPS563201DDCR saves ~$200 on the run; D-CAP2 stays at fixed freq under load |
| Fallback IC | LMR33630ADDAR if motor driver protection cannot be verified before first build |
| 5V noise filtering | **Point-of-load LC filter** (10µH + 47µF) on the filtered branch; ~40dB at 700kHz |
| 3.3V generation | **AP2112K-3.3 LDO** from filtered 5V rail — quiet, cheap (~$0.10/unit) |
| 12V rail protection | **470µF bulk + SS34 flyback diodes on motor drivers + SMAJ15CA TVS** |

---

## Sources

- [TPS563201DDCR DigiKey](https://www.digikey.com/en/products/detail/texas-instruments/TPS563201DDCR/5808204) — $0.209/unit @ 1ku
- [LMR33630ADDAR Arrow](https://www.arrow.com/en/products/lmr33630addar/texas-instruments) — ~$1.85/unit @ 2.5ku
- [LMR33630ADDAR DigiKey](https://www.digikey.com/en/products/detail/texas-instruments/LMR33630ADDAR/8554847)
- [LMR33630 Datasheet (TI)](https://www.ti.com/lit/ds/symlink/lmr33630.pdf) — 3.8–36V, 3A, >95% peak efficiency
- [TI App Note SNVA859A — LMR33630 CISPR-22 Compliance](https://www.ti.com/lit/an/snva859a/snva859a.pdf)
- [TPS563201 Product Page (TI)](https://www.ti.com/product/TPS563201) — 580 kHz, D-CAP2, 4.5–17V
- [LM2596 Lab Test (element14)](https://community.element14.com/members-area/personalblogs/b/rachael-s-blog/posts/lab-experiments---performance-evaluation-and-testing-of-lm2596-dc-dc-converter-board) — poor conducted EMI at 150kHz switching
- [Motor back-EMF protection guide (Zbotic)](https://zbotic.in/dc-motor-back-emf-protection-flyback-diode-circuit-guide/)
- [TVS for motor transients — TI E2E Forum](https://e2e.ti.com/support/motor-drivers-group/motor-drivers/f/motor-drivers-forum/1056279/help-regarding-back-emf-protection-during-free-wheeling-of-motor-manually-rotating-it)
- [Pros and Cons of DC/DC Converter Modules (Cadence)](https://resources.pcb.cadence.com/blog/pros-and-cons-of-dc-dc-converter-modules)
