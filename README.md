# Milliohmmeter

A precision resistance measurement device based on STM32 and INA226,
designed for measuring low-resistance components in the range from µΩ to ~5 Ω.

**Status:** Hardware design complete — battery management circuit under revision



## Overview

This device measures ultra-low resistances using a stabilized pulsed current source.
Pulsed measurement mode prevents resistance drift caused by self-heating of the DUT.

**Typical applications:**
- Transformer and motor winding resistance
- Low-resistance shunts
- PCB via and trace resistance
- Connector and contact resistance



## Key Features

- Measurement range: µΩ — 5 Ω
- Test current: 100 mA / 500 mA / 1000 mA (auto-range)
- Pulsed measurement mode (thermal drift elimination)
- OLED display SSD1306 (128×64)
- Li-ion battery powered with 5V USB charging
- Flexible current and reference voltage control via PWM + RC filter + voltage divider



## Hardware

| Component | Description |
|-----------|-------------|
| STM32 | Main microcontroller — PWM generation, I2C, measurement control |
| INA226 | High-precision differential current/voltage sensor (I2C) |
| Op-Amp + MOSFET | Stabilized current source |
| SSD1306 | 128×64 OLED display |
| Li-ion cell | Main power source |
| TP4056 / BMS | Battery charging and protection |

**PCB:** 4-layer, designed in Altium Designer



## Measurement Principle

1. MCU generates PWM signal → RC filter converts to DC → voltage divider sets reference
2. Op-amp + MOSFET maintain stable test current through DUT
3. INA226 measures voltage drop across DUT via I2C
4. Resistance is calculated: R = U / I
5. Pulsed mode: current applied briefly, measurement taken at peak, then off
   → eliminates thermal resistance drift



## Specifications (target)

| Parameter | Value |
|-----------|-------|
| Measurement range | µΩ — 5 Ω |
| Test current | 100 / 500 / 1000 mA |
| Accuracy | < 1% |
| Display | OLED SSD1306 128×64 |
| Power supply | Li-ion, USB-C 5V charging |
| PCB layers | 4 |



## Project Status

- [x] Schematic design
- [x] PCB layout (4-layer)
- [x] Gerber files generated
- [ ] Battery management circuit revision
- [ ] PCB manufacturing
- [ ] Firmware development
- [ ] Device calibration and testing
- [ ] Enclosure design (SolidWorks)


## Schematic & PCB

[Schematic and PCB layers (PDF)](hardware/fabrication/Fabrication.pdf)
[Bill of Materials](hardware/fabrication/Bill%20of%20Materials-111.xlsx)
[PCB 3D renders](photos/)



## Author

**runaway**
Learning hardware engineering — this is my first complete PCB project.



## License

MIT License
