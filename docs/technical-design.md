# Technical Design Document

## INA226

Selected for its flexibility and precision:
- VBUS line enables software auto-range in future revisions
- Maximum gain error: 0.01%
- Input offset voltage: 10 µV

INA226 supports 4-wire (Kelvin) connection — separate force and sense
lines eliminate contact and lead resistance from the measurement,
which is essential for µΩ-level accuracy.

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
