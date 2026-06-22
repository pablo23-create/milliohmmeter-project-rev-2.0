# DC-DC Block — MT3608 Boost Converter

## Overview

The DC-DC block steps up the Li-ion battery voltage to a regulated 5.5 V rail,
which feeds downstream LDO regulators:

- **LDO 3.3 V** — digital supply (STM32, SSD1306)
- **LDO 5 V (LP2985)** — clean analog supply (OPA333, INA226 reference)

Using 5.5 V as the boost target instead of 5.0 V provides headroom for the
LP2985 LDO (dropout ~30 mV) and ensures stable output across the full Li-ion
discharge range (3.0 V – 4.2 V).

---

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

### Inductor — Bourns SRP5030T-4R7M

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

Ripple current estimate at $V_{in} = 3.7\ V$, $D = 1 - V_{in}/V_{out}$:

$$D = 1 - \frac{3.7}{5.5} = 0.327$$

$$\Delta I_L = \frac{V_{in} \times D}{L \times f_{sw}} = \frac{3.7 \times 0.327}{4.7\mu \times 1.2M} = \frac{1.21}{5.64} \approx 0.215\ A$$

Ripple ΔI_L ≈ 215 mA — well within acceptable range (~30% of DC current).

**Why shielded:**
The SRP5030T series uses a closed magnetic core. This suppresses fringing
flux that would otherwise couple into the L2 solid GND plane and propagate
switching noise across the board — critical given the µΩ-level measurements
in the ADC block.

**Peak current margin:**

At maximum load, estimated $I_{out} = 200\ mA$ (conservative, all blocks active):

$$I_{peak} = I_{out} \times \frac{V_{out}}{V_{in} \times \eta} + \frac{\Delta I_L}{2}$$

$$I_{peak} = 0.2 \times \frac{5.5}{3.7 \times 0.85} + 0.108 \approx 0.35 + 0.11 = 0.46\ A$$

I_sat = 6 A provides a safety margin of >10× — the inductor will not saturate
under any realistic operating condition.

### Feedback Divider

Output voltage is set by a resistive divider on the FB pin:

$$V_{out} = V_{ref} \times \left(1 + \frac{R_{upper}}{R_{lower}}\right)$$

With $V_{ref} = 0.6\ V$ and $V_{out} = 5.5\ V$:

$$\frac{R_{upper}}{R_{lower}} = \frac{V_{out}}{V_{ref}} - 1 = \frac{5.5}{0.6} - 1 = 8.167$$

Choosing $R_{lower} = 100\ kΩ$:

$$R_{upper} = 816.7\ kΩ \rightarrow \textbf{820\ kΩ}\ (E24\ series)$$

Verification:

$$V_{out} = 0.6 \times \left(1 + \frac{820k}{100k}\right) = 0.6 \times 9.2 = \textbf{5.52\ V}$$

Error: +0.4% — acceptable. The 5.5 V target is a headroom rail, not a
precision reference.

> **Note:** Use 1% tolerance resistors for the feedback divider.
> 5% resistors introduce up to ±0.5 V error on V_out — sufficient to
> drop below LP2985 minimum input or exceed absolute maximum ratings
> of downstream components.

### Rectifier Diode — SS36

| Parameter       | Value      |
|-----------------|------------|
| Type            | Schottky   |
| V_R (reverse)   | 60 V       |
| I_F (avg)       | 3 A        |
| V_F at 1 A      | ~0.5 V     |
| Package         | DO-214AB   |

SS36 handles the full switched current with margin. V_R = 60 V exceeds
the worst-case voltage spike ($V_{out}$ + ringing) with headroom.
Low forward voltage (~0.5 V) keeps conduction losses minimal.

### Output Capacitor — 22 µF Ceramic (X8R)

| Parameter     | Value                      |
|---------------|----------------------------|
| Capacitance   | 22 µF (nominal)            |
| Dielectric    | X8R                        |
| Voltage rating| ≥ 10 V recommended         |

**Why ceramic over electrolytic:**
Ceramic capacitors have very low ESR, which reduces output voltage ripple
at 1.2 MHz. Electrolytic capacitors have significantly higher ESR at this
frequency and are less effective at filtering switching noise.

**DC bias derating:**
Ceramic capacitors lose effective capacitance under DC bias voltage.
This is a known characteristic of all ceramic dielectrics — the MT3608
datasheet reference design specifies 22 µF nominal but does not account
for derating of a specific component at operating voltage.

At 5.5 V DC bias, the 22 µF X8R capacitor degrades to approximately
**~4.4 µF effective capacitance** (~80% loss).

Impact on output voltage ripple:

$$\Delta V_{out} = \frac{I_{out} \times D}{C_{eff} \times f_{sw}}$$

With $C_{eff} = 4.4\ µF$, $I_{out} = 100\ mA$, $D = 0.327$:

$$\Delta V_{out} = \frac{0.1 \times 0.327}{4.4\mu \times 1.2M} \approx \textbf{6.2\ mV}$$

For comparison, at nominal 22 µF without derating:

$$\Delta V_{out} = \frac{0.1 \times 0.327}{22\mu \times 1.2M} \approx \textbf{1.4\ mV}$$

The derating increases ripple by ~4.5×. However, the LP2985 LDO
(PSRR = 75 dB at 1.2 MHz) attenuates this ripple by a factor of ~5600
before it reaches the analog supply rail. The 6.2 mV ripple on the
5.5 V boost rail does not compromise analog measurement accuracy.

**Mitigation options (for future revision):**
- Add a second 22 µF X8R in parallel → effective capacitance ~8.8 µF,
  ripple reduced to ~3.1 mV
- Replace with 100 µF X8R (10 V rating) → effective capacitance ~20+ µF
  after derating

> **Note for bring-up:** Verify output ripple with an oscilloscope at the
> 5.5 V rail and at the LP2985 output (5 V analog rail). Expected ripple
> after LP2985: < 2 µV. If measured ripple exceeds this, add a second
> output capacitor in parallel.

X8R was selected over X5R/X7R for its superior DC bias stability —
derating behaviour is better across the same voltage range.

---

## PCB Layout Considerations

The MT3608 switching loop carries high di/dt currents at 1.2 MHz.
Poor layout directly causes noise coupling into analog measurements.

**Critical loop:** MT3608 SW pin → Inductor → Output capacitor → GND → MT3608 GND

- Keep this loop as small as possible on L1
- Place output capacitor immediately adjacent to the inductor and SS36 cathode
- SW node trace must be short and wide — it switches the full inductor current
- Do not route sensitive analog signals parallel to the SW node trace

**GND plane (L2):**
- L2 solid GND plane provides a low-impedance return path and shields
  L3 power traces from L1 switching noise
- The shielded inductor (SRP5030T) prevents flux coupling into L2
- Analog GND (INA226, OPA333) and power GND are connected at a single
  point — star ground topology — to prevent switching return currents
  from flowing through the analog ground path

**LP2985 placement:**
The LP2985 LDO sits between the 5.5 V boost rail and the analog supply.
Its 75 dB PSRR at 1.2 MHz provides the primary isolation of the analog
block from MT3608 switching noise. Placement close to the analog block
minimizes the length of the post-regulation trace, reducing pickup area.

---

## Power Budget

| Rail              | Consumer                  | Estimated Current |
|-------------------|---------------------------|-------------------|
| 3.3 V             | STM32F103                 | ~50 mA            |
| 3.3 V             | SSD1306 OLED              | ~20 mA            |
| 5 V (LP2985)      | OPA333 + INA226           | ~5 mA             |
| 5.5 V             | LDO quiescent currents    | ~5 mA             |
| **Total**         |                           | **~80–100 mA**    |

At $I_{out} = 100\ mA$ and $V_{in} = 3.7\ V$, MT3608 operates well within
its current limit. Efficiency at this light load is typically 85–90%.

---

## Summary

| Component         | Part                  | Role                                       |
|-------------------|-----------------------|--------------------------------------------|
| Boost IC          | MT3608                | 1.2 MHz PWM boost controller               |
| Inductor          | Bourns SRP5030T-4R7M  | 4.7 µH, shielded, Isat = 6 A              |
| Rectifier         | SS36                  | Schottky, 3 A, 60 V                        |
| Output cap        | 22 µF X8R ceramic     | Low-ESR output filter (eff. ~4.4 µF @ 5.5V)|
| FB divider        | 820 kΩ / 100 kΩ       | Sets V_out = 5.52 V                        |
| Post-reg (analog) | LP2985                | Isolates analog block from switching noise  |
