# DC-DC Block — MT3608 Boost Converter

## Overview

The DC-DC block steps up the Li-ion battery voltage to a regulated 5.5 V rail,
which feeds downstream LDO regulators:

- **LP5907 3.3 V** — digital supply (STM32, SSD1306)
- **LP2985 5 V** — clean analog supply (OPA333)

Using 5.5 V as the boost target instead of 5.0 V provides 520 mV of headroom
for the LP2985 LDO (dropout ~30 mV) and ensures stable output across the full
Li-ion discharge range (3.0 V – 4.2 V).

The boost output is enabled via Q1 (AO3401A, P-channel MOSFET) controlled
by the EN signal from LTC2954 power-on controller.


## Component Selection

### MT3608 — Boost Controller

| Parameter         | Value                  |
|-------------------|------------------------|
| Topology          | Boost (PWM)            |
| Switching freq.   | 1.2 MHz (internal osc) |
| V_ref (FB)        | 0.6 V                  |
| SW current limit  | 4 A                    |
| V_in range        | 2.0 V – 24 V           |
| Package           | SOT-23-6               |

1.2 MHz switching frequency is the key design constraint for this project.
All analog measurements (INA226, OPA333 loop) must be protected from
switching noise at this frequency — addressed in the ADC block via
LP2985 isolation and RC filters on INA226 inputs.

### Power Switch — AO3401A (Q1)

P-channel MOSFET controlled by EN signal from LTC2954. Connects VBAT to
the MT3608 input when EN is asserted. R29/R30 (100 kΩ / 100 kΩ) form a
gate voltage divider to ensure reliable turn-on/turn-off.

### Inductor — Bourns SRP5030T-4R7M (L1)

| Parameter     | Value       |
|---------------|-------------|
| Inductance    | 4.7 µH      |
| I_sat         | 6 A         |
| I_rms         | 5 A         |
| DCR           | ~27 mΩ      |
| Package       | 5.0×5.0 mm  |
| Shielding     | Yes         |

**Why 4.7 µH:**
At 1.2 MHz switching frequency, a smaller inductance is sufficient to
maintain continuous conduction mode (CCM) without excessive ripple current.

Duty cycle at V_in = 3.7 V:

    D = 1 - V_in / V_out = 1 - 3.7 / 5.5 = 0.327

Inductor ripple current:

    dI_L = (V_in x D) / (L x f_sw)
         = (3.7 x 0.327) / (4.7uH x 1.2MHz)
         = 1.21 / 5.64
         = 0.215 A

Ripple dI_L = 215 mA — well within acceptable range.

**Peak current at maximum load (1.2 A):**

    I_peak = I_out x (V_out / (V_in x eta)) + dI_L / 2
           = 1.2 x (5.5 / (3.7 x 0.85)) + 0.108
           = 2.10 + 0.11
           = 2.21 A

I_sat = 6 A provides a safety margin of 2.7x at full load.

**MT3608 current limit verification:**

    I_out_max = (I_sw_max - dI_L / 2) x V_in x eta / V_out
              = (4 - 0.108) x 3.7 x 0.85 / 5.5
              = 2.25 A

1.2 A load is within MT3608 capability with ~1.9x margin.

**Why shielded:**
The SRP5030T series uses a closed magnetic core. This suppresses fringing
flux that would otherwise couple into the L2 solid GND plane and propagate
switching noise across the board — critical given the µΩ-level measurements
in the ADC block.

### Feedback Divider (R2 / R3)

Output voltage is set by a resistive divider on the FB pin:

    V_out = V_ref x (1 + R2 / R3)

With V_ref = 0.6 V and target V_out = 5.5 V:

    R2 / R3 = V_out / V_ref - 1 = 5.5 / 0.6 - 1 = 8.167

Choosing R3 = 100 kΩ:

    R2 = 100k x 8.167 = 816.7 kΩ  →  820 kΩ (E24 series)

Verification:

    V_out = 0.6 x (1 + 820k / 100k) = 0.6 x 9.2 = 5.52 V

Error: +0.4% — acceptable. The 5.5 V target is a headroom rail, not a
precision reference.

> **Note:** Use 1% tolerance resistors for the feedback divider.
> 5% resistors introduce up to ±0.5 V error on V_out — sufficient to
> drop LP2985 input below its minimum or exceed downstream component
> absolute maximum ratings.

### Rectifier Diode — SS36AHL (D1)

| Parameter       | Value      |
|-----------------|------------|
| Type            | Schottky   |
| V_R (reverse)   | 60 V       |
| I_F (avg)       | 3 A        |
| V_F at 1 A      | ~0.5 V     |
| Package         | DO-214AB   |

SS36AHL is an improved variant of SS36 with lower reverse leakage current
and better thermal stability. V_R = 60 V exceeds worst-case voltage spikes
(V_out + switching ringing) with margin. Handles full load current of
1.2 A with 2.5x headroom.

### Output Capacitor — C3, 22 µF Ceramic (X8R)

| Parameter      | Value               |
|----------------|---------------------|
| Capacitance    | 22 µF (nominal)     |
| Dielectric     | X8R                 |
| Voltage rating | ≥ 10 V recommended  |

**Why ceramic over electrolytic:**
Ceramic capacitors have very low ESR, which reduces output voltage ripple
at 1.2 MHz. Electrolytic capacitors have significantly higher ESR at this
frequency and are less effective at filtering switching noise.

**DC bias derating:**
Ceramic capacitors lose effective capacitance under DC bias voltage.
The MT3608 datasheet reference design specifies 22 µF nominal but does
not account for derating at operating voltage.

At 5.5 V DC bias, the 22 µF X8R capacitor degrades to approximately
~4.4 µF effective capacitance (~80% loss).

Impact on output voltage ripple at maximum load (1.2 A):

    dV_out = (I_out x D) / (C_eff x f_sw)
           = (1.2 x 0.327) / (4.4uF x 1.2MHz)
           = 0.392 / 5.28
           = 74.8 mV

For comparison, at nominal 22 µF without derating:

    dV_out = (1.2 x 0.327) / (22uF x 1.2MHz) = 14.9 mV

Derating increases ripple by 5x. At 1.2 A load this is significant —
74.8 mV ripple on the 5.5 V rail reduces the effective headroom for
LP2985 from 520 mV to ~445 mV under worst case. LP2985 (PSRR = 75 dB
at 1.2 MHz) attenuates this ripple by ~5600x before it reaches the
analog supply, so measurement accuracy is not compromised. However,
adding a second output capacitor is recommended.

**Recommended fix — add second 100 µF X8R in parallel:**

    C_eff_total = 4.4 + 19 = 23.4 uF
    dV_out = (1.2 x 0.327) / (23.4uF x 1.2MHz) = 13,9 mV

Ripple reduced by 2x with minimal board area impact (same footprint).

**INA226 averaging as supplementary ripple mitigation:**
The INA226 ADC supports hardware averaging (up to 1024 samples).
This is used as a supplementary measure alongside output capacitor
filtering — averaging reduces the effect of residual switching noise
on voltage measurements, improving effective resolution at µΩ levels.
Averaging does not replace output filtering but complements it.

X8R was selected over X5R/X7R for better DC bias stability.

### Input Capacitor — C1, 22 µF Ceramic

C1 provides local decoupling at the MT3608 input, stabilizing V_in
during switching transients and reducing input ripple current drawn
from the battery.


## PCB Layout Considerations

The MT3608 switching loop carries high di/dt currents at 1.2 MHz.
Poor layout directly causes noise coupling into analog measurements.

**Critical loop:** MT3608 SW pin → L1 → D1 → C3 → GND → MT3608 GND

- Keep this loop as small as possible on L1 signal layer
- Place C3 immediately adjacent to D1 cathode and MT3608 output
- SW node trace must be short and wide — it carries full inductor current
- Do not route sensitive analog signals parallel to the SW node trace

**Component placement:**
The DC-DC block (Q1, U2, L1, D1) is located in the top-left corner of
the board. The analog measurement block (OPA333, INA226) is located on
the right edge, with STM32 between them. This physical separation
minimizes direct coupling of switching noise into the analog section.

**GND plane (L2):**
The board uses a single solid GND plane on L2 without a separate analog
ground pour. This is acceptable given the physical separation between
the DC-DC and analog blocks — switching return currents follow the
path of least impedance back to Q1/U2 and do not pass through the
analog measurement zone.

The shielded inductor (SRP5030T) prevents magnetic flux from coupling
into the GND plane beneath it.

**LP2985 placement:**
LP2985 sits between the 5.5 V boost rail and the OPA333 analog supply.
Its 75 dB PSRR at 1.2 MHz is the primary isolation barrier for the
analog block. Placement close to OPA333 minimizes post-regulation
trace length and reduces pickup area.


## Power Budget

| Rail           | Consumer             | Estimated Current |
|----------------|----------------------|-------------------|
| 3.3 V (LP5907) | STM32                | ~50 mA            |
| 3.3 V (LP5907) | SSD1306 OLED         | ~20 mA            |
| 5 V (LP2985)   | OPA333               | ~5 mA             |
| 5.5 V          | LDO quiescent + misc | ~5 mA             |
| 5.5 V          | Current source (DUT) | ~1100 mA (peak)   |
| **Total**      |                      | **~1180 mA**      |

Peak load occurs only during measurement pulses (~100 ms). Average
current draw is significantly lower due to the pulsed measurement mode.

MT3608 maximum deliverable output current at V_in = 3.7 V:

    I_out_max = (I_sw_max - dI_L / 2) x V_in x eta / V_out
              = (4 - 0.108) x 3.7 x 0.85 / 5.5
              = 2.25 A

1.2 A peak load leaves ~1.05 A margin.


