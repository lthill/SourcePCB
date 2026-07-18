# Buzzer Selection and Drive Circuit

## Requirements
- Audible from ~20ft away or an adjacent room with doors open
- Not objectionable in volume when standing at the machine
- Buzzer is inside an enclosed housing (attenuation expected, amount unknown)
- Driven from the existing 5V rail

## Selected Buzzer
**CUI/Same Sky CEM-1206S**
- 12 mm, through hole, sealed, magnetic audio transducer
- 5V rated, ~85 dB @ 10cm
- Native 5V operation matches existing logic rail — no separate supply needed

Note: SPL ratings are quoted at a reference voltage for comparison purposes. Actual
in-enclosure SPL will be attenuated (potentially 15-20 dB for a sealed plastic
housing), and final loudness should be confirmed empirically once the enclosure
is available.

## Drive Circuit
Do not drive the buzzer directly from a GPIO — use a low-side transistor switch.
This protects the GPIO, allows PWM control for tone/volume tuning, and handles
the buzzer's current draw (~30-40 mA typical for magnetic buzzers).

```
5V Rail ---- Buzzer (+)
              |
           Buzzer (-)
              |
         Transistor (collector/drain)
              |
GPIO --[1k]-- Transistor (base/gate)
              |
         Transistor (emitter/source) ---- GND

Flyback diode (1N4148) across buzzer terminals,
cathode to 5V, anode to buzzer (-)
```

- Transistor: 2N7000 (MOSFET) or 2N2222 (NPN BJT) — either works at 5V
- Flyback diode recommended for magnetic buzzers due to internal coil
  (inductive kickback on switch-off)
- PWM duty cycle can be used in firmware to tune perceived loudness without
  changing hardware, once enclosure attenuation is known

## Approximate BOM Cost (single-unit pricing)
| Component | Cost |
|---|---|
| Buzzer (CEM-1206S) | $0.50 - $1.00 |
| Transistor (2N7000 / 2N2222) | $0.10 - $0.20 |
| Flyback diode (1N4148) | $0.02 - $0.05 |
| **Total** | **$0.60 - $1.25** |

Costs likely drop to ~$0.30-$0.60 per unit at production volumes (1k+).
Resistor costs not included (negligible).

## Open Items
- Confirm CEM-1206S SPL in actual enclosure once prototype housing is available
- If too quiet/loud, adjust via PWM duty cycle before considering a different
  SPL-rated part
- Verify CEM-1206S current sourcing/availability before finalizing BOM
