# Source Turbo Control PCB

## PN 203, Rev A1

This is a new version of the Source Turbo control PCB SRC_REV_Q

### Major changes
1. Replacement of the MXP5100DP with a 0-100Kpa absolute pressure sensor, 3.3V
2. Depopulated RN4020 BLE Module
3. Additional test point pads for use with Keystone 5010
4. New analog signal chains forced by sensor change
5. PCB layout changes driven by the sensor
6. Additional of test points

Output signal chain is the same from the Rev_Q Board.
The orginal, difficult to read schematic, remains the same. :-(

Removal of the BLE frees the hardware UART and the bulk of the processor resources.
Processing time was already generous *with the removal of blocking code* even with BLE 
Bi-Directional terminal access may be a very useful development option.

### Software Requirements
1. Removal of the BLE will require significant software changes
2. Patch for absolute pressure sensor

## Implementaion
Removal of the BLE will require patches or a re-write.
The absolute pressure sensor should be an object with private variables and allow for different pressure ranges.
Suggest looking are ratiometric sensor for characterizaton. https://github.com/lthill/ratiometric-sensor


## TODO
- [ ] Basic verification of the new hardware
- [ ] Test pressure sensor analog signal chain
- [ ] Test thermistor signal chain
- [ ] Sensor is unvalidated for use with with ethanol
- [ ] Softwre patch to remove BLE code
- [ ] Software patch for the new sensor


## Demo Sensors
- XGZP6858A15115KPAL0, 15-115kPa, 5V
- XGZP6858A15115KPAL033, 15-115kPa, 3.3V

Note the different pressure ranges

## Fallback Options
1. If validation doesn't pass use MPXH6115AC6U
2. Use OEM, CFSensor XGZP6869A100KPGL033, HBP1311A, HBP1501D, Senoch GZP6858A15115KPA50KL0


