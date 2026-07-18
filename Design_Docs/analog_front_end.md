# 3. Sensors / Analog Front End

## Purpose

Provide reliable measurement of process conditions while maintaining flexibility during Rev A development. The architecture shall support future sensor changes without major PCB redesign.

---

## Requirements

### Processor Independence

Sensor accuracy shall not depend on the microcontroller's internal ADC performance.

An external ADC shall be used for all critical analog measurements.

### Analog Architecture

The design shall include:

* External serial ADC (I²C or SPI)
* Quiet analog supply/reference strategy
* Ratiometric measurement where applicable
* Input filtering
* Calibration constant storage in nonvolatile memory

### Grounding

The PCB shall provide an analog measurement island with careful grounding practices.

Requirements:

* Separate analog and noisy power returns
* Star connection to system ground
* Minimize heater, pump, and switching current flow through sensor return paths

---

## Temperature Measurement

### Process Temperature

The system shall provide process temperature measurement for control and diagnostics.

Current implementation uses NTC thermistors.

Requirements:

* ±0.5 °C repeatability target
* ±1.0 °C absolute accuracy target
* 1% thermistor minimum
* Precision divider resistor
* Firmware filtering

### Heater Temperature

A heater-adjacent temperature sensor shall be provided.

Requirements:

* Temperature sensor package rated for at least 150 °C
* Preference for 200 °C-capable sensor packages where practical
* Used for control, diagnostics, and fault detection

The heater temperature sensor is not considered the primary safety device.

### Ambient Temperature

An ambient air temperature sensor shall be provided.

Purpose:

* Detect operation in excessively warm rooms
* Detect operation in excessively cold rooms
* Improve diagnostic capability
* Improve interpretation of condenser performance
* Support future user warnings and recommendations

---

## Pressure Measurement

### Architecture

Pressure shall be measured through the existing pump-line connection to the vacuum chamber.

This connection method is fixed by system architecture.

### Sensor Selection

Pressure sensor selection remains open pending testing.

The specification shall not require:

* A specific pressure sensor family
* Analog versus digital output
* Absolute versus differential measurement

Final selection shall be based on:

* Process usefulness
* Accuracy
* Long-term stability
* Ethanol and condensate exposure tolerance
* Pump-line measurement behavior
* Cost and availability

### PCB Flexibility

Rev A hardware should preserve flexibility for pressure sensor evaluation where practical.

---

## Calibration

Calibration data shall be stored in nonvolatile memory.

Examples:

* Serial number
* Calibration revision
* Pressure calibration constants
* Temperature offsets
* Manufacturing calibration data

---

## Open Items

### TBD During Rev A Testing

1. Pressure sensor selection
2. ADC part selection
3. ADC resolution requirement
4. Pressure measurement strategy
5. Additional optional temperature locations
6. Calibration procedure

---

## Notes

The sensor subsystem is considered design-ready.

Pressure sensor selection remains intentionally open until testing determines the most suitable solution.
