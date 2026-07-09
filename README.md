# Milliohmmeter

A precision resistance measurement device based on STM32 and INA226,
designed for measuring low-resistance components in the range from µΩ to ~5 Ω.

**Status:** PCB manufacturing

## Overview

This device measures ultra-low resistances using a stabilized pulsed current source.
Pulsed measurement mode prevents resistance drift caused by self-heating of the DUT.

**Typical applications:**
- Transformer and motor winding resistance
- Low-resistance shunts
- PCB via and trace resistance
- Connector and contact resistance



## Hardware

| Component | Description |
|-----------|-------------|
| STM32F103C8T6 | Main microcontroller - PWM generation, I2C, measurement control |
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
6. 4-wire (Kelvin) connection — separate current injection and voltage
   sensing paths eliminate lead and contact resistance.



## Specifications (target)

| Parameter | Value |
|-----------|-------|
| Measurement range | µΩ - 5 Ω |
| Test current | 100 / 500 / 1000 mA |
| Accuracy | < 1% |
| Display | OLED SSD1306 128×64 |
| Power supply | Li-ion, USB-C 5V charging |
| PCB layers | 4 |


## Status Legend

- [x] Complete
- [~] In progress
- [ ] Planned

## Project Status

- [x] Schematic design
- [x] PCB layout (4-layer)
- [~] PCB manufacturing
- [ ] Firmware development
- [ ] Device calibration and testing
- [ ] Enclosure design (SolidWorks)



## Firmware Architecture

**Toolchain:** STM32CubeIDE, HAL, bare-metal (no RTOS)

**Module structure:**

| File | Responsibility |
|------|----------------|
| `ina226.c/.h` | INA226 init, register config, voltage read via I2C |
| `ssd1306.c/.h` | OLED display driver |
| `current_source.c/.h` | PWM control, current range selection |
| `measurement.c/.h` | Measurement cycle, averaging, zero calibration |
| `main.c` | Init, superloop |

**Measurement cycle (single reading):**
1. Select current range (100 / 500 / 1000 mA) — auto-selected by result
2. Enable PWM → wait settling time (RC filter + OPA333 loop)
3. Read INA226 voltage across DUT
4. Disable PWM
5. R = U / I

**Zero calibration:**
On startup with probes shorted — measure residual offset, store in Flash, subtract from all subsequent readings.

**Firmware development order:**
1. INA226 driver — verify I2C communication and register reads
2. SSD1306 driver — needed for all subsequent debugging
3. PWM configuration — verify current flow with oscilloscope
4. Measurement cycle assembly
5. Auto-range and zero calibration

## Schematic & PCB

[Schematic and PCB layers (PDF)](hardware/Fabrication.pdf)
[Bill of Materials](hardware/fabrication/Bill%20of%20Materials-111.xlsx)
[PCB 3D renders](photos/)



## Author

**runaway**



## License

MIT License
