# Lessons Learned — Milliohmmeter Project

A running log of mistakes, wrong turns, and how they were fixed.
Format: what went wrong → why → what was done instead.


## Rev 1 → Rev 2

### LTC2954 — Replaced with Discrete Latch

**What went wrong:**
LTC2954 was used as a push-button power-on controller. The IC works,
but it is expensive, requires careful external component selection
(PDT capacitor for timing), and adds unnecessary complexity for a
simple latched power path.

**Why it was a problem:**
- Higher cost and lower availability compared to discrete solution
- PDT timing capacitor requires calculation and tuning
- Overkill for a device that only needs: press-to-on, firmware-controlled-off

**What was done instead:**
Replaced with a discrete latch using three common components:
- AO3401A — P-channel MOSFET, main power switch (same as before)
- BSS138 — N-channel MOSFET, holds AO3401A gate low after boot
- BAT54C — dual Schottky diode assembly, OR-logic between SW1 and BSS138

MCU drives PB3 HIGH after boot to engage the latch.
Power-off: firmware drives PB3 LOW → system shuts down.

**Trade-off accepted:**
No hardware kill timer (LTC2954 had one). IWDG is now the only
fault recovery path. On IWDG timeout the reset handler must drive
PB3 LOW explicitly, otherwise the system reboots in a loop.

---
