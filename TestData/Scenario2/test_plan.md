# Test Plan: Scenario 2 – Field Return, "broken"

Fault-finding plan for a field-returned unit of the Scenario 1 design. [PCB.fbrd](PCB.fbrd) and [Schematic.fsch](Schematic.fsch) are identical to [Scenario 1](../Scenario1/), so the circuit, test points and connections are the same. The only fault description is "broken".

**Measurement equipment:** only the **oscilloscope** (CH1 on the robot, CH2 fixed by hand) and the **UR3e robot**. There is no multimeter and no camera. The external signal generator and bench supply provide stimulus only, and their settings are made by the operator. Every conclusion comes from voltages captured on the scope.

**Approach:**
1. With the power **off**, drive a small signal through the passive RC network. This checks R3, C1 and the input path without any risk from the op-amp.
2. Power up with current limiting, then check the supply rails and the DC state.
3. Trace a small signal through TP1 → TP2 → TP3 → TP4. The first point that departs from normal shows the faulty section. The signature table in section 6 then narrows it down to likely parts.

If the Scenario 1 board has been characterised with [../Scenario1/test_plan.md](../Scenario1/test_plan.md), its results are the reference ("golden board"). Otherwise use the design expectations given here.

**Board file:** `robot_board(operation='load', file='scenario2_opamp_board')`.

## 1. Device under test (reminder)

```
J3.2 IN ──R3──┬──R1──┬───────┐
       TP1    │ TP2  │ TP3   │\
              C1     └──R2──┐│−\___ OUT ── J2.1, TP4
              │             ││+/   │
             GND            ││/    │
                            └───────┘     +IN = GND
Supply: JP1.1 = VCC+ (TP5), JP1.2 = GND (TP6), JP1.3 = VCC- (TP7)
```

The component values are not in the design files and cannot be measured directly here. The plan therefore works with **ratios and frequencies**, which the scope can measure:

| Quantity | Powered (op-amp working) | Unpowered (op-amp off) |
|---|---|---|
| TP2 / TP1 at low frequency | D = R1 / (R1 + R3) (R1 goes to virtual ground) | ≈ 1 (R1 goes to an unpowered, high-impedance input) |
| Corner frequency at TP2 | fc = 1 / (2π · C1 · (R1 ∥ R3)) | fu = 1 / (2π · R3 · C1) |
| TP4 / TP2 | −R2 / R1 | — |
| TP3 | ≈ 0 V (virtual ground) | — |

Two useful cross-checks:
- **fc / fu = 1 + R3/R1 = 1 / D.** The powered and unpowered corner frequencies must agree with the powered divider ratio. A mismatch points to a failed part.
- **Gain ratios:** the measurements give R2/R1, R1/R3 and the time constants R3·C1 and C1·(R1∥R3).

## 2. Equipment and connections

| Item | Connection | Settings |
|---|---|---|
| Dual bench supply | JP1: pin 1 = +, pin 2 = GND, pin 3 = − | Off for F1. Then **±5 V, 20 mA limit** (F2). ±15 V, 50 mA limit, only once F2 passes. |
| Signal generator (stimulus) | J3: pin 2 = signal, pin 1 = GND | **High-Z** load setting, output off until asked, 0 V DC offset |
| Scope CH2 (fixed, 10:1, placed by hand) | J3 pin 2, ground on J3 pin 1. Moved by the operator only where a step says so. | DC coupling, trigger source |
| Scope CH1 (robot probe, 10:1) | TP1–TP7; ground clip on J2 pin 2 or TP6 | DC coupling |

**[OPERATOR]** marks a step for the operator: setting the generator or supply, moving the CH2 clip, connecting a load, or flexing or heating the board. Whoever runs the plan (for example Claude) states the exact setting and waits for confirmation. The operator makes **no measurements**; every reading is taken on the scope.

**Stop rule:** stop and switch off if a supply rail collapses at TP5/TP7 (the supply is in current limit), or if the operator notices heat, smell or smoke.

## 3. Preparation (power off)

| Step | Action | Pass |
|---|---|---|
| P1 | Connect the scope (`scope_device connect`) and the robot. Load `scenario2_opamp_board`. | Both connected |
| P2 | Teach the board frame with `robot_teach`. Find the header pin height by jogging the probe onto TP6 and set `z_mm` if it differs from 8.5 mm. | Frame saved |
| P3 | For each of TP1–TP7: `dry_run`, then `above_point`. The operator checks the probe is centred over the pin. A bent or missing pin (field damage) is recorded and that point skipped. | All usable points aligned within ±0.5 mm |
| P4 | Scope: CH1 and CH2 probe ratio 10:1, DC coupling, trigger on CH2 edge, High-Res or Average (8) acquisition for small signals. | — |

## 4. Tests

### F1 – Passive network test, power off

The op-amp is unpowered, so only R3, C1 and the input path are active.

**[OPERATOR]** Supply **off** (JP1 connected, output disabled). Generator: sine, **100 mVpp**, 0 V offset. Keep the level this small so no junction inside the unpowered LM741 conducts.

**F1a – Input path and loading at 100 Hz:**

| Point | Measure | Healthy | Fault indicated |
|---|---|---|---|
| CH2 (J3.2) | Vpp | About 100 mVpp, the set level | Much lower → input shorted to GND (J3, R3 + C1 both shorted, or board short) |
| TP1 | Vpp, phase relative to CH2 | = CH2 (within 2 %, 2°) | Lower or 0 → J3.2 solder joint or input track open |
| TP6 | Vpp, DC mean | ≈ 0 | Signal on TP6 → ground path from J3.1 broken (board ground floating) |

**F1b – RC network sweep:** the robot probe stays on **TP2**. **[OPERATOR]** Sine 100 mVpp at 10 Hz, 30 Hz, 100 Hz, 300 Hz, 1 kHz, 3 kHz, 10 kHz, 30 kHz, 100 kHz. At each frequency record Vpp(TP2)/Vpp(CH2) in dB and the phase.

| Observation | Fault indicated |
|---|---|
| Low-pass from about 0 dB with a single corner **fu**, −45° at fu | Healthy R3/C1. Record fu. |
| TP2 ≈ 0 at all frequencies | **R3 open** or **C1 shorted** (see F1a: a short also loads the input) |
| TP2 = TP1 at all frequencies, no roll-off | **C1 open** (or **R3 shorted**) |
| Corner well above the reference fu | **C1 lost capacitance** (aged electrolytic), or R3 lower in value |
| Low-frequency level well below 0 dB | Leakage path from TP2 to GND (leaky C1, contamination), or R1 path conducting into the unpowered IC |

**F1c – Spot check at the TP3 and TP4 nodes at 100 Hz:**

| Point | Healthy (unpowered) | Fault indicated |
|---|---|---|
| TP3 | ≈ TP2 (R1 into a high-impedance input) | ≈ 0 → **R1 open**, or IC1 input shorted to GND or a rail |
| TP4 | Small, ≈ TP3 through R2 (exact level depends on the unpowered IC) | Much larger than TP3 → impossible unpowered, so check probe contact. ≈ 0 with TP3 ≈ TP2 → R2 open, or output shorted to GND. |

**[OPERATOR]** Generator output off.

### F2 – Current-limited power-up

**[OPERATOR]** Generator output off. Supply at **±5 V, 20 mA limit**. Switch on both rails together.

| Point | Healthy | Fault indicated |
|---|---|---|
| TP5 | +5 V ±0.2 V | Well below +5 V → **rail in current limit**: short from V+ (IC1, board), or IC1 latched up. Switch off at once. |
| TP7 | −5 V ±0.2 V | Same for V− |
| TP6 | 0 V ±20 mV | Ground path problem (JP1.2 joint) |
| TP5, TP7 AC (AC coupling, 1 ms/div) | Ripple < 50 mVpp | Oscillation or a heavy pulsed load → IC1 damaged |

If it passes: **[OPERATOR]** raise the supply to **±15 V, 50 mA limit** and repeat. Pass: TP5 = +15 V ±0.5 V and TP7 = −15 V ±0.5 V, with no sag.

TP5 and TP7 sit on the JP1 side of the supply tracks. A track or joint that is open between TP5/TP7 and the IC1 pins is not seen here; F3 and F4 show it as "IC1 not powered".

### F3 – DC state with no signal

**[OPERATOR]** Generator connected, output on, 0 V DC (sine at minimum amplitude, or DC mode 0 V).

The robot probes TP1, TP2, TP3 and TP4 and records the DC mean of each. It also takes an AC capture of TP4 at 100 ns/div and 10 µs/div to look for oscillation.

| Point | Healthy | Fault indicated |
|---|---|---|
| TP1, TP2 | 0 V ±10 mV | DC present with 0 V input → leakage from a rail (contamination, damaged board) |
| TP3 | 0 V ±10 mV | Several mV to volts → IC1 input stage damaged, or the loop is not closed (R2 open, IC1 unpowered) |
| TP4 | \|V\| < 100 mV | **Near +13 V or −13 V (saturated)** → R2 open, or IC1 damaged. **0 V exactly and TP3 floating** → IC1 not powered (open supply track/joint) or dead. |
| TP4 AC | Noise only | Oscillation (kHz–MHz) → IC1 damaged/unstable, or capacitive load on J2 |

### F4 – Powered signal trace

**[OPERATOR]** Sine, **100 mVpp at TP1**, at a frequency of **fu/10** from F1b (use 100 Hz if F1b found no corner). The level is small so that even a high-gain fault does not clip.

The robot probes TP1 → TP2 → TP3 → TP4 in turn. At each point record Vpp, phase relative to CH2 and DC mean, and keep a scope screenshot (`scope_capture` / screenshot tool).

| Point | Healthy | First failing point means |
|---|---|---|
| TP1 | = CH2 (within 2 %, 2°) | Input path: J3.2 joint or track (should already show in F1a) |
| TP2 | D × TP1, about 0°, with **D < 1** (lower than in F1 because R1 now goes to virtual ground) | **TP2 unchanged from F1 (D ≈ 1)** → no virtual ground: IC1 dead or unpowered, or R1 open (F1c). ≈ 0 → R3 open or C1 shorted (F1b). |
| TP3 | ≪ TP2 (a few mV) | ≈ TP2 → op-amp not controlling the loop: IC1 dead or not powered at its pins, or **R2 open** |
| TP4 | (R2/R1) × TP2, inverted (≈ 180°) | ≈ 0 with TP3 ≈ 0 → **R2 shorted** or output shorted to GND. Gain wrong → R1/R2 value drift. Clipped/saturated → R2 open or IC1 fault. Not inverted → assembly error. |

**F4b – Output connector:** **[OPERATOR]** move the CH2 probe tip from J3 pin 2 to **J2 pin 1** (ground on J2 pin 2), with the robot probe on TP4. CH2 must equal TP4. Anything else is an open J2 solder joint or track. **[OPERATOR]** then move CH2 back to J3 pin 2.

### F5 – Frequency response and cross-check

**[OPERATOR]** Sine 100 mVpp at TP1, at 10 Hz, 30 Hz, 100 Hz, 300 Hz, 1 kHz, 3 kHz, 10 kHz, 30 kHz, 100 kHz.

Run the sweep twice: first with the robot probe on **TP2**, then on **TP4**. Record the gain in dB and the phase relative to CH2.

| Result | Healthy |
|---|---|
| fc (−3 dB at TP2 and TP4) | The same at both points (the op-amp adds no pole below about 100 kHz) |
| **fc / fu** (F5 against F1b) | = 1/D (D from F4), within 10 % |
| A0 = TP4/TP1 at low frequency | = D × (R2/R1), and matches the golden board if one is available |

| Observation | Fault indicated |
|---|---|
| fc and fu both much higher than reference, ratios correct | **C1 lost capacitance**, the most common field failure for an electrolytic |
| fc/fu ≠ 1/D | R1 or R3 drifted or damaged (overstress) |
| Gain wrong, corner frequencies correct | R2 (or R1) drifted |
| TP4 rolls off well below TP2's corner | IC1 degraded (reduced bandwidth / slew) |

### F6 – Large signal and output stage

Run only if F1–F5 do not explain the fault. The robot probe is on TP4, and CH2 stays on the input. **[OPERATOR]** Sine at fu/10 (≤ 100 Hz). Raise the amplitude so TP4 goes 1 → 5 → 10 → 20 Vpp, and up to clipping.

Then move the robot probe to TP5 and TP7 at the highest level, to check that the supply rails don't sag under signal.

| Observation | Fault indicated |
|---|---|
| Clips well below ±12 V, or clips asymmetrically | IC1 output stage damaged, or a weak rail under load (TP5/TP7 sag) |
| Crossover distortion near 0 V (scope FFT: odd harmonics, THD > 1 %) | IC1 output stage damaged |
| **[OPERATOR]** connect a 2 kΩ load (or the field load) to J2: output collapses or distorts | Output stage weakened by short-circuit or overload |

### F7 – Intermittent faults

Run this test if the unit passes F1–F6, which means "broken" is not reproducible.

The robot probe stays on TP4. **[OPERATOR]** sets a sine at fu/10, about 1 Vpp out. Scope: trigger on CH1 with a runt/edge trigger set just below the normal amplitude, or roll mode at 200 ms/div. Use single captures so each event is kept.

**[OPERATOR]** then performs each check in turn:

1. Flex the J3, J2 and JP1 cables and plugs.
2. Press gently on IC1, C1 and the connectors (non-conductive tool).
3. Warm the board with a hair-dryer, then let it cool.
4. Cycle the supply off and on several times.

| Observation on the scope | Fault indicated |
|---|---|
| Dropouts while flexing a connector | Cracked solder joint or worn contact at that connector |
| Change when pressing IC1 or C1 | Cracked joint or socket contact at that part |
| Gain/corner drift with temperature (repeat F5 spot points while warm) | C1 (electrolytic) or IC1 temperature-sensitive |
| Fails only after power cycling | Latch-up or supply sequencing issue |

## 5. Shutdown

1. **[OPERATOR]** Generator output off.
2. `robot_move(operation='home')`.
3. **[OPERATOR]** Supply off.

## 6. Likely faults and their scope signatures

| Suspected fault | F1 (unpowered) | F2 rails | F3 DC at TP4 | F4 signal trace | F5 |
|---|---|---|---|---|---|
| IC1 shorted / destroyed | Possibly TP3 ≈ 0 (input shorted) | **Rail collapses** | Stuck at a rail or 0 V | TP2 as unpowered (D ≈ 1), TP3 ≈ TP2 | — |
| IC1 not powered (open supply joint/track) | Normal | Normal at TP5/TP7 | 0 V or floating | Same as unpowered: D ≈ 1, TP3 ≈ TP2, TP4 ≈ 0 | — |
| R2 open (feedback lost) | TP4 ≈ 0 while TP3 ≈ TP2 | Normal | Saturated ±13 V | TP3 ≈ TP2, TP4 clipped | — |
| R2 shorted / solder bridge | Normal | Normal | ≈ 0 V | TP3 ≈ 0, TP4 ≈ 0 | Gain ≈ 0 |
| R1 open | TP3 ≈ 0 | Normal | Offset only | TP2 = F1 level (D ≈ 1), TP3/TP4 ≈ 0 | fc = fu |
| R3 open | TP2 ≈ 0, input not loaded | Normal | ≈ 0 V | TP1 OK, TP2/TP4 ≈ 0 | — |
| C1 shorted | TP2 ≈ 0 | Normal | ≈ 0 V | TP2 ≈ 0, TP4 ≈ 0 | — |
| C1 open / lost capacitance | No or high corner fu | Normal | Normal | OK at low frequency | **fc, fu too high**, ratio fc/fu correct |
| R1/R3 drifted | fu shifted (R3) | Normal | Normal | D wrong | **fc/fu ≠ 1/D** |
| Input overstress (overvoltage on J3) | Possibly abnormal TP3 (IC input damaged) or R3 open | Possibly normal | Offset large | TP3 not at virtual ground | — |
| J3 / J2 / JP1 solder joint or connector | TP1 ≠ CH2, or TP6 shows signal | Rail low or missing | — | F4b: J2.1 ≠ TP4 | F7 dropouts |
| No fault found | All normal | Normal | Normal | Normal | Matches reference |

If no fault is found: retest with the field supply voltage, input level and load (open item 2), then report "no fault found" with all captures.

## 7. Results to report

- A table of every measurement against healthy/reference, with pass/fail, and the saved scope screenshots and CSV captures for each failing point.
- The derived values fu, fc, D, A0 and R2/R1, against the golden board or the design relations.
- The first failing test and the suspected component(s) from section 6.
- Recommended repair, and a retest of the full [Scenario 1 characterisation](../Scenario1/test_plan.md) after repair.

## 8. Open items

1. **Golden reference:** has the Scenario 1 board been characterised? Its fu, fc, D and A0 make every comparison exact. Without it, only internal consistency (fc/fu = 1/D, single-pole shapes) and gross faults can be judged.
2. **Field conditions:** supply voltage, input level and source, and output load in use. These matter for reproducing the fault and for judging overstress.
3. **More detail on "broken":** no output, wrong output, noisy, intermittent, or physical damage?
4. **IC1 pins are not robot test points.** An open supply track at IC1 pins 4 and 7 can only be inferred (section 6), not measured. Probing DIP-8 legs at 2.54 mm pitch risks shorting adjacent pins (for example output pin 6 to V+ pin 7), so it is not planned.
5. **Generator, probes and pin height:** same assumptions as Scenario 1 (sine/square ≥ 100 kHz, High-Z setting, 10:1 probes, pin height taught in P2).
