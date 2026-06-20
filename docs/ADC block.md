# ADC Block — Technical Design Document

## OPA333 — Current Source Control

OPA333 was selected for its near-zero input offset voltage (zero-drift),
rail-to-rail input/output, and wide input voltage range.
These characteristics are critical for precise reference voltage
reproduction at the op-amp input, which directly controls
the test current stability.

## PWM-to-Reference Conversion (RC Filter)

MCU generates PWM signal which is converted to a stable DC reference
voltage for the op-amp via a two-stage RC filter.

**Filter parameters (each stage):**
- R = 10 kΩ, C = 100 nF
- Cutoff frequency: f = 1/(2π·R·C) ≈ 159 Hz

Two identical cascades do not shift the cutoff frequency —
it remains at 159 Hz — but increase the roll-off slope
from -20 dB/dec to -40 dB/dec, significantly improving
PWM ripple suppression at 100 kHz.

**Settling time:**
- Single stage: 5τ = 5·RC = 5 ms
- Two stages: approximately 12–13 ms (empirical ~2.5× factor)

PWM frequency of 100 kHz ensures fast RC settling,
allowing pulsed measurement duration to be reduced if needed
by increasing PWM frequency — shorter settling time means
shorter minimum pulse width.

## Voltage Divider (Post-Filter)

A 10:1 voltage divider is placed after the two-stage RC filter.

This serves two purposes:
- Scales the reference voltage to the required input range of OPA333
- Reduces residual PWM ripple by an additional factor of 10,
  providing a clean reference for the op-amp

A stable reference at OPA333 input ensures the op-amp correctly
drives the MOSFET gate, keeping the test current stable
during pulsed measurements.

## OPA333 + IRLR7843 — Feedback Stability

IRLR7843 gate capacitance (Ciss = 4380 pF) combined with OPA333
open-loop output impedance (~5 kΩ) and unity-gain bandwidth (350 kHz)
would cause high-frequency oscillation without compensation.

**Isolation resistor R9:**
Placed between op-amp output and MOSFET gate to isolate
capacitive load from the feedback loop.

R9 = 1/(2π·F·C) = 1/(2π·350kHz·4380pF) ≈ 103 Ω → selected 120 Ω
(within recommended range 20–120 Ω)

**Feedback filter R11 + C14:**
R11 (feedback resistor) selected at 10 kΩ to avoid loading the op-amp
(recommended range: 10 kΩ – 100 kΩ).

C14 calculated for optimal damping:
C = Ciss × (Ro + Riso) / Rf = 4380 × (5000 + 120) / 10000 ≈ 2243 pF
→ selected nearest standard value: 2.2 nF (C0G/NP0 for stability)

Notes: 
-R11 value on schematic to be updated to 10 kΩ.
-The OPA333 and IRLR7843 combination itself works very slowly and is only suitable for linear modes, which is exactly what we are talking about.

## INA226 — Voltage Measurement

INA226 measures voltage drop directly across the DUT.
Test current is fixed by the OPA333 control loop (via PWM reference),
so resistance is calculated as R = U / I.

INA226 supports 4-wire (Kelvin) connection — separate force and sense
lines eliminate contact and lead resistance from the measurement,
which is essential for µΩ-level accuracy.

Differential input traces (VIN+ and VIN−) are routed as short as
possible following differential pair rules (tight coupling)
for common-mode noise rejection.

## INA226 — Input Filter (R8, R22, C21)

MT3608 DC-DC converter operates at 1.2 MHz. The measurement shunt
sits on this supply line, making INA226 inputs susceptible to
switching noise at 1.2 MHz — which coincides with harmonics of
INA226 internal ADC sample rate (500 kHz ±30%, harmonics at
1 MHz, 1.5 MHz...).

Per INA226 datasheet recommendation:
- R8, R22 = 10 Ω (series input resistors)
- C21 = 1 µF (differential filter capacitor)

This filter attenuates switching noise above 1 MHz before it
reaches INA226 inputs, preventing ADC corruption during measurement.

## LP2985 — Analog Power Supply

LP2985 (5V version) provides a clean, isolated supply for OPA333
and analog circuitry, separated from the noisy DC-DC boost converter rail.

Selected for:
- PSRR: ~75 dB at low frequencies
- Output noise: ~30 µV RMS
- Dropout voltage: ~30 mV at 150 mA
- Output accuracy: ±0.5%
- Minimal external components

5V version selected to match IRLR7843 gate drive requirements.
OPA333 supply quality directly affects output signal integrity —
any supply noise appears as reference error and impacts
current source stability.

## Pulsed Measurement Mode

Continuous current flow causes resistive heating of the DUT,
leading to resistance drift during measurement.

Pulsed mode applies current for ~100 ms per measurement cycle.
This allows multiple consecutive measurements without thermal drift
of the DUT and PCB traces.

## 4-Layer PCB

4-layer stackup was chosen for compactness, routing simplicity,
and noise immunity.

Stackup: L1 (signal) / L2 (solid GND) / L3 (power) / L4 (signal/GND)

Dedicated GND and power planes isolate signal traces from supply
currents. This is critical during pulsed measurements — high di/dt
events on power layers do not couple into the measurement signal
path, reducing transient-induced errors.
