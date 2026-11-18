# Test Plan: Scenario 1 – MCP Measurement Test Board

Characterisation of the board in [PCB.fbrd](PCB.fbrd) / [Schematic.fsch](Schematic.fsch). The test points are in [scenario1_opamp_board.json](scenario1_opamp_board.json) (load with `robot_board(operation='load', file='scenario1_opamp_board')`).

## 1. Device under test

```
J3.2 IN ──R3──┬──R1──┬───────┐
       TP1    │ TP2  │ TP3   │\
              C1     └──R2──┐│−\___ OUT ── J2.1, TP4
              │             ││+/   │
             GND            ││/    │
                            └───────┘     +IN = GND
Supply: JP1.1 = VCC+ (TP5), JP1.2 = GND (TP6), JP1.3 = VCC- (TP7)
```

An RC low-pass filter (R3, C1) feeds an LM741 inverting amplifier (R1 input, R2 feedback). The component values are **not in the design files**, so this plan finds them empirically. With the op-amp input at virtual ground:

| Quantity | Expression | How it is measured |
|---|---|---|
| Passband gain A0 | −R2 / (R1 + R3) | TP4 / TP1 at low frequency (T3) |
| RC divider ratio D | V(TP2) / V(TP1) at low frequency = R1 / (R1 + R3) | T5 |
| Amplifier gain | V(TP4) / V(TP2) = −R2 / R1 | T5 |
| Cutoff fc | 1 / (2π · C1 · (R1 ∥ R3)) | −3 dB point (T4) |

From these the plan derives **R2/R1**, **R1/R3** and **C1·(R1∥R3)**. Absolute values need one known part, or the values read off the fitted parts.

Expected limits of the LM741 at ±15 V: output swing about ±13 V (12–14 V), slew rate about 0.5 V/µs, gain-bandwidth about 1 MHz, input offset ≤ 5 mV.

## 2. Equipment and connections

| Item | Connection | Settings |
|---|---|---|
| Dual bench supply | JP1: pin 1 = +15 V, pin 2 = GND (common), pin 3 = −15 V | Current limit 50 mA per rail. Switch both rails on together. |
| External signal generator | J3: pin 2 = signal, pin 1 = ground | Load setting **High-Z**: the input is R3, not 50 Ω, so a 50 Ω setting shows half the real amplitude. Output **off** until a test asks for it. DC offset 0 V. |
| Scope CH2 (fixed reference, 10:1 probe) | Tip on J3 pin 2 (= TP1 net), ground clip on J3 pin 1 | DC coupling, probe 10:1. Trigger source. |
| Scope CH1 (robot probe, 10:1 probe) | Held by the UR3e, moved between TP1–TP7 | DC coupling, probe 10:1 |
| CH1 probe ground | Clip to GND at J2 pin 2 (or TP6), routed so the robot can reach all points | Check that the lead does not snag during travel. |

The scope's MSO digital channels are not used.

**Operator actions** are the steps marked **[OPERATOR]**: setting the generator or switching the supply. Whoever runs the plan (for example Claude) stops at each one, states the exact setting, and waits for confirmation.

## 3. Preparation (power off)

| Step | Action | Pass |
|---|---|---|
| P1 | Connect the scope (`scope_device connect`) and the robot. Load `scenario1_opamp_board`. | Both connected |
| P2 | Teach the board frame with `robot_teach`. Measure the real header pin height and set `z_mm` (default 8.5 mm) if it differs. | Frame saved |
| P3 | For every TP1–TP7: run a `dry_run`, then `above_point`, and check visually that the probe is centred over the pin. | All aligned within ±0.5 mm |
| P4 | Probe TP6 with power off and check the probe touches the pin (no slipping off). Correct with `nudge` if needed. | Stable contact |
| P5 | Scope: CH1 and CH2 probe ratio 10:1, DC coupling, trigger on CH2 edge, acquire mode High-Res or Average (8) for small signals. | — |

## 4. Tests

Unless a step says otherwise, the CH1 robot probe goes to the point named, and amplitudes are peak-to-peak at **TP1 (CH2)**, not the generator display.

### T1 – Supply rails

**[OPERATOR]** Generator output off. Switch on the ±15 V supply and note the current on each rail.

| Point | Measure | Expected / pass |
|---|---|---|
| TP5 | DC mean, AC ripple Vpp | +15 V ±0.5 V, ripple < 50 mVpp |
| TP7 | DC mean, AC ripple Vpp | −15 V ±0.5 V, ripple < 50 mVpp |
| TP6 | DC mean | 0 V ±20 mV, which confirms the probe ground is good |
| Supply display | Quiescent current per rail | about 1.7 mA typical, < 3 mA (LM741) |

Stop if any rail is outside its limit or a rail current is above 10 mA.

### T2 – DC operating point and offset

**[OPERATOR]** Generator connected, output **on**, 0 V DC, amplitude at minimum (or set DC mode to 0 V).

| Point | Measure | Expected / pass |
|---|---|---|
| TP1 | DC mean | 0 V ±10 mV |
| TP2 | DC mean | 0 V ±10 mV |
| TP3 | DC mean | 0 V ±10 mV (virtual ground; shows input offset) |
| TP4 | DC mean | \|Voffset\| < 100 mV (offset × noise gain) |

### T3 – Coarse frequency survey (find the passband and the cutoff)

The CH1 robot probe stays on **TP4** for the whole sweep. Only the generator changes.

**[OPERATOR]** Sine, start at 100 mVpp, 1 kHz. Look at TP4. Adjust the amplitude so that **TP4 is about 1 Vpp**, or as large as possible if the gain is < 1. Keep that generator amplitude for the whole sweep. 1 Vpp at the output stays inside the 741's slew limit up to about 150 kHz.

At each frequency **[OPERATOR sets the frequency]**, record Vpp(CH2), Vpp(CH1), gain = Vpp(CH1)/Vpp(CH2) in dB, and phase CH1 relative to CH2:

10 Hz, 30 Hz, 100 Hz, 300 Hz, 1 kHz, 3 kHz, 10 kHz, 30 kHz, 100 kHz, 300 kHz, 1 MHz

Results:
- **A0** is the flat low-frequency gain. Expected phase there is ±180° (inverting).
- **fc (estimate)** is the frequency where the gain is 3 dB below A0.
- **Pass:** a flat passband below fc, then roll-off. Phase moves from −180° towards −270° as frequency rises (shown on the scope as +180° → +90°).
- If the gain falls faster than 20 dB/decade, or extra phase shift appears well above fc, record the frequency. It may be the op-amp's own bandwidth (GBW ≈ 1 MHz / noise gain).

### T4 – Fine sweep around the cutoff

Robot probe still on TP4, same amplitude as T3.

**[OPERATOR]** Set the frequencies at about 10 points per decade, from fc/10 to 10·fc (with fc from T3). Add points close to the −3 dB crossing until it is known to ±2 %.

| Result | Pass |
|---|---|
| fc (−3 dB) | Recorded to ±2 % |
| Phase at fc | −225° ±10° (shown as +135°), as for a single-pole low-pass |
| Slope from 3·fc to 10·fc | −20 dB/decade ±3 dB |

### T5 – Internal nodes (gain split between the RC filter and the amplifier)

**[OPERATOR]** Same sine amplitude. Run at three frequencies: **fc/10**, **fc**, **10·fc**.

At each frequency move the robot probe through **TP1 → TP2 → TP3 → TP4** and record Vpp and phase relative to CH2.

| Point | Expected |
|---|---|
| TP1 | Same as CH2 (checks that both probes agree; within 2 %, 2°) |
| TP2 | At fc/10: D = R1/(R1+R3), phase about 0°. At fc: −3 dB from D, −45°. |
| TP3 | Very small: virtual ground, rising with frequency. Report Vpp, and note any signal above 10 mVpp at fc/10. |
| TP4 | Equal to the T3/T4 result at the same frequency |

Derived: **R2/R1 = Vpp(TP4)/Vpp(TP2)** at fc/10, and **R1/R3 = D/(1−D)**.

### T6 – Output swing and linearity

Robot probe on TP4. **[OPERATOR]** Sine at **fc/10**, but no higher than 1 kHz.

Raise the generator amplitude in steps so that TP4 is about 1, 2, 5, 10, 20 Vpp, then up to the point of clipping. Stop at the generator's maximum if the gain is too low to reach clipping.

| Result | Pass |
|---|---|
| Gain at each step | Within 1 % of A0 up to 20 Vpp |
| Maximum output before clipping (+ and − peak) | About +13 V / −13 V (≥ ±12 V) |
| THD at 10 Vpp out (scope FFT) | < 1 % |

**[OPERATOR]** Return the generator to the T3 amplitude afterwards.

### T7 – Step response

**[OPERATOR]** Square wave, frequency **fc/20** (so each half-period is long enough to settle). Amplitude such that TP4 is about 1 Vpp.

| Point | Measure | Expected |
|---|---|---|
| TP2 | 10–90 % rise time, overshoot | Rise ≈ 0.35/fc, no overshoot (first-order RC) |
| TP4 | 10–90 % rise and fall time, overshoot, settling | Rise ≈ 0.35/fc (same as TP2), overshoot < 5 %, edges inverted |

If the TP4 edges are clearly slower than TP2's, raise the amplitude to TP4 = 10 Vpp and measure the slew rate (expected about 0.5 V/µs). This is visible only if fc is above about 10 kHz.

### T8 – Supply rails under load (optional)

**[OPERATOR]** Sine at fc/10, TP4 at 20 Vpp (from T6).

| Point | Measure | Pass |
|---|---|---|
| TP5, TP7 | AC ripple Vpp | < 50 mVpp, with no signal-frequency component above 20 mV |

## 5. Shutdown

1. **[OPERATOR]** Generator output off.
2. `robot_move(operation='home')`.
3. **[OPERATOR]** Switch off the ±15 V supply.

## 6. Results to report

- A table per test with measured values and pass/fail.
- A Bode plot (gain dB and phase against log frequency) from T3 and T4.
- Derived values: A0, fc, R2/R1, R1/R3, C1·(R1∥R3). Add absolute R and C values if one part value is known.
- Anything unexpected: offset, oscillation, noise, extra poles.

## 7. Open items

- **Signal generator limits:** the plan assumes sine and square output to at least 1 MHz, about 20 Vpp maximum into High-Z, and a High-Z load setting. Lower the T3 upper frequency or the T6 maximum to suit the generator.
- **Probe attenuation:** assumed 10:1 on both channels.
- **Header pin height:** assumed 8.5 mm until it is measured in P2.
