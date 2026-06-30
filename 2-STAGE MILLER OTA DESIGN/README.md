# Two-Stage Miller-Compensated OTA — TSMC 65nm

A two-stage Miller-compensated operational transconductance amplifier (OTA) designed in **TSMC 65nm CMOS**, sized using the **gm/ID methodology**, and verified in **Cadence Virtuoso (ADE L)** with Spectre/APS.

> ⚠️ **Verification notes before treating this as final documentation**
> 1. **SR vs. Cc inconsistency.** With Cc = 10 pF and a 20 µA tail bias, SR = I/Cc ≈ 2 mV/µs, not the 20 V/µs listed below. Either Cc, the tail current, or the SR target needs correcting — see [Known Issues](#known-issues--open-items).
> 2. **CL mismatch.** The schematic's output capacitor (`c1`) is annotated `c=1p`, but the target spec below lists CL = 2 pF.
> 3. **Device widths for M0, M5, M7, M10** were read from a screenshot using "N×Wµm" multiplier notation that was not fully legible. Confirm against the netlist or a DC operating-point printout before publishing.
> 4. **Phase margin** computed from the AC sweep cursor is ≈56.8°, not the ~60° stated as a target — confirm with `calculator → phaseMargin()` rather than a manual cursor read.

---

## 1. Target Specifications

| Parameter | Target | Simulated / As-Drawn | Status |
|---|---|---|---|
| Technology | TSMC 65nm CMOS | — | — |
| Topology | Two-stage Miller-compensated OTA | — | Confirmed |
| Supply Voltage (VDD) | — | 1.2 V (inferred from testbench) | **Confirm** |
| Bias Current (Ibias) | 20 µA | 20 µA (DC current source, testbench) | Confirmed |
| DC Gain (Av) | 55 dB | **56.20 dB** (AC sim) | Confirmed (sim) |
| Phase Margin (PM) | ~60° | **≈56.8°** (derived from cursor phase of −123.17° at marked frequency) | **Re-verify** |
| Gain-Bandwidth Product (GBW) | — | Not annotated on supplied AC plot | **Needs extraction** |
| Slew Rate (SR) | 20 V/µs | Inconsistent with Cc = 10 pF (see notes) | **Needs correction** |
| Compensation Cap (Cc) | 10 pF | C0 (Miller cap, value not visible in schematic export) | **Confirm** |
| Load Capacitance (CL) | 2 pF | C1 shows `c = 1 pF` in schematic | **Mismatch** |
| Input Common-Mode Range (ICMR) | 0.6 V – 1.0 V | — | Confirmed (target) |
| Power Consumption | — | Ibias × VDD ≈ 24 µW (at 1.2 V, excluding mirror branches) | Estimate |

---

## 2. Topology

Classic two-stage Miller-compensated OTA, single-ended output:

- **Stage 1 — Differential transconductance stage:** NMOS input pair with PMOS current-mirror load, single NMOS tail current source.
- **Stage 2 — Common-source gain stage:** PMOS gain device with NMOS current-sink load.
- **Compensation:** Miller capacitor (C0) from the output node back to the stage-1 output / stage-2 gate node, implementing pole-splitting. RHP zero from Cc is not actively cancelled (no nulling resistor observed in the schematic).
- **Bias generation:** One diode-connected NMOS reference device mirrors Ibias to the stage-1 tail and stage-2 current sink.

### Device Role Map

| Instance | Type | Role |
|---|---|---|
| M1, M2 | NMOS | Differential input pair (gates: VIN−, VIN+) |
| M3, M4 | PMOS | Current-mirror active load, stage 1 (M3 diode-connected) |
| M5 | NMOS | Tail current source, stage 1 |
| M0 | NMOS | Diode-connected bias reference (sets Ibias = 20 µA) |
| M7 | PMOS | Common-source gain device, stage 2 |
| M10 | NMOS | Current-sink load, stage 2 |
| C0 | — | Miller compensation capacitor |
| C1 | — | Output load capacitance (CL) |

---

## 3. Transistor Sizing

| Instance | Type | W | L | Fingers | Confidence |
|---|---|---|---|---|---|
| M1, M2 | NMOS | 10 µm | 1 µm | 1 | Likely |
| M3, M4 | PMOS | 12 µm | 1 µm | 1 | Likely |
| M5 | NMOS | ~20 µm (2 × 10 µm) | 2 µm | 1 | **Guessing — confirm** |
| M0 | NMOS | ~20 µm (2 × 10 µm) | 2 µm | 1 | **Guessing — confirm** |
| M7 | PMOS | ~30 µm | 1 µm | 1 | **Guessing — confirm** |
| M10 | NMOS | ~30 µm | 2 µm | 1 | **Guessing — confirm** |

**Recommended fix:** export `Design > Create Netlist` or run a parametric "Print Selected Parameter" pass in ADE and paste the W/L/M values directly into this table — manual reads off a schematic screenshot are not a reliable source of record for a public repository.

Design rationale evident from the sizing pattern: input pair and mirror/gain devices use minimum-ish length (1 µm) to maximize gm and bandwidth; current-source/sink devices (M0, M5, M10) use longer length (2 µm) to maximize output resistance (ro) and improve current-source matching — standard practice, consistent with the topology.

---

## 4. Design Methodology

The design followed a **gm/ID-based sizing flow**, rather than direct square-law hand calculations, to account for moderate/weak-inversion behavior typical of low-current 65nm analog design:

1. **Spec definition.** Fixed targets: DC gain, GBW (via Cc), phase margin, SR, ICMR, load cap, and a hard power/current budget (Ibias = 20 µA).
2. **Topology selection.** Two-stage Miller chosen over a single-stage telescopic/folded-cascode topology because the ICMR window (0.6 V – 1.0 V) and gain target (≥55 dB) at a constrained supply are difficult to satisfy with a single cascoded stage at this bias current; the two-stage approach trades a second gain stage (and the associated RHP zero / compensation complexity) for relaxed output-swing and ICMR headroom.
3. **Current budgeting.** The 20 µA reference (M0) sets the stage-1 tail current via M5; the stage-2 bias current (M10) is set independently by the mirror ratio between M10 and M0, sized to meet the stage-2 transconductance requirement below.
4. **Stage-1 sizing (gm/ID).** Using TSMC 65nm gm/ID characterization curves for the NMOS input device, an operating point (gm/ID) was selected to balance transconductance efficiency against the available 10 µA per side (half of the 20 µA tail). The corresponding W/L was read off the characterization curve for the chosen ID and gm/ID.
5. **Mirror load sizing (M3/M4).** Sized for matching and to keep gds low enough to support the stage-1 gain target Av1 = gm1/(gds2+gds4), while keeping VOV small enough not to eat into the upper ICMR limit.
6. **Compensation capacitor selection.** Cc chosen against the rule-of-thumb lower bound Cc ≥ 0.22·CL for ~60° phase margin in a single-Miller-pole approximation, then iterated against the stage-2 gm condition below.
7. **Stage-2 sizing for phase margin.** gm7 (stage-2 gm) sized to satisfy gm7 ≈ 2.2 · gm(1,2) · (CL/Cc), pushing the non-dominant pole sufficiently beyond the unity-gain frequency to realize the target phase margin.
8. **Slew-rate check.** SR = I(M5)/Cc checked against the 20 V/µs target. *(This is the step where the current inconsistency above was introduced — Cc and I(M5) need to be revisited together, since increasing Cc to satisfy phase margin directly degrades SR and GBW.)*
9. **ICMR / operating-point verification.** DC operating-point analysis across the input common-mode range to confirm every device remains in saturation with adequate VOV margin at both 0.6 V and 1.0 V.
10. **AC verification.** Loop/open-loop AC analysis for DC gain and phase margin.
11. **Transient verification.** Large-signal step response for slew rate and settling time.

---

## 5. Key Design Equations

Mapped from standard two-stage Miller OTA theory to this design's instance names (textbook generic M6 → this design's M7; textbook generic M7 (stage-2 current source) → this design's M10).

**DC Gain**

$$A_{v1} = g_{m(1,2)} \cdot R_{out1} = \frac{g_{m(1,2)}}{g_{ds2}+g_{ds4}}$$

$$A_{v2} = g_{m7} \cdot R_{out2} = \frac{g_{m7}}{g_{ds7}+g_{ds10}}$$

$$A_v = A_{v1} \cdot A_{v2}$$

**Frequency Response & Compensation**

$$GBW = \omega_u \approx \frac{g_{m(1,2)}}{C_c}$$

$$p_1 \approx \frac{-1}{R_{out1} \cdot R_{out2} \cdot C_c} \quad \text{(dominant pole)}$$

$$p_2 \approx \frac{-g_{m7}}{C_L} \quad \text{(non-dominant / output pole)}$$

$$z_1 \approx \frac{g_{m7}}{C_c} \quad \text{(RHP zero)}$$

Cc forces the dominant pole to a low frequency (pole splitting), pushing p2 out past the unity-gain frequency.

**Stability & Slew Rate**

$$g_{m7} \approx 2.2 \cdot g_{m(1,2)} \left(\frac{C_L}{C_c}\right) \quad \text{(condition for ≈60° PM)}$$

$$SR = \frac{I_{M5}}{C_c}$$

**Compensation Rule of Thumb**

$$C_c \gtrsim 0.22 \cdot C_L$$

---

## 6. Simulation Results

- **DC Gain:** 56.1954 dB (AC sweep, 1 Hz – 1 GHz)
- **Phase at sweep cursor:** −123.1678° — phase margin derivable as 180° − 123.17° ≈ **56.8°** if the cursor is at the unity-gain crossover (recommend confirming with the Calculator's built-in `phaseMargin()` function rather than a manual cursor read)
- **GBW:** not directly annotated in the supplied AC plot — extract via the 0 dB crossing frequency or `gainBwProd()`
- **SR, ICMR:** as stated in target table; SR requires re-verification per the Known Issues section

---

## 7. Repository Structure

*(adjust paths to match your actual repo layout before publishing)*

```
.
├── README.md
├── schematics/
│   ├── opamp_schematic.png
│   └── opamp_testbench.png
├── results/
│   ├── ac_response.png
│   └── transient_response.png
└── netlist/
    └── opamp.scs
```

---

## 8. Tools & Environment

- Cadence Virtuoso (ADE L), Spectre/APS simulator
- TSMC 65nm CMOS PDK
- Host: Linux (Ubuntu 24.04)

---

## 9. Known Issues / Open Items

- [ ] Reconcile SR (20 V/µs target) with Cc (10 pF) and I(M5) (20 µA) — current math gives SR ≈ 2 mV/µs, not 20 V/µs.
- [ ] Resolve CL: target spec says 2 pF, schematic-drawn C1 shows 1 pF.
- [ ] Confirm exact W/L/multiplier for M0, M5, M7, M10 via netlist export — current values are screenshot-derived estimates.
- [ ] Confirm phase margin via `phaseMargin()` rather than manual cursor reading (~56.8° read vs. ~60° target).
- [ ] Extract and report GBW numerically.
- [ ] Confirm supply voltage (currently inferred as 1.2 V from testbench `vdc` source, not independently stated).
