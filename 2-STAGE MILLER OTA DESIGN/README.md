# Two-Stage Miller-Compensated OTA — TSMC 65nm

A two-stage Miller-compensated operational transconductance amplifier (OTA) designed in **TSMC 65nm CMOS**, sized using the **gm/ID methodology**, and verified in **Cadence Virtuoso (ADE L)** with Spectre/APS.

---

## Target Specifications

**Technology:** TSMC 65nm CMOS

**Topology:** Two-stage Miller-compensated OTA

| Parameter | Target | Simulated |
|-----------|--------|-----------|
| VDD | 1.2 V | 1.2 V |
| Ibias | 20 µA | 20 µA |
| DC Gain | 50 dB | 56.20 dB |
| Phase Margin | ~60° | 56.8° |
| GBW | 30 MHz | 17 MHz |
| Slew Rate | 20 V/µs | 20.35 V/µs |
| Cc | 1 pF | 1 pF |
| CL | 1 pF | 1 pF |
| ICMR | 0.6–1.0 V | 0.6~1.0V |
| CMRR | — | 57.75 dB |
---

## 2. Topology

Classic two-stage Miller-compensated OTA, single-ended output:

- **Stage 1 — Differential transconductance stage:** NMOS input pair with PMOS current-mirror load, single NMOS tail current source.
- **Stage 2 — Common-source gain stage:** PMOS gain device with NMOS current-sink load.
- **Compensation:** Miller capacitor (C0) from the output node back to the stage-1 output / stage-2 gate node, implementing pole-splitting. 
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

| Instance | Type | W | L | 
|---|---|---|---|
| M1, M2 | NMOS | 10 µm | 1 µm | 
| M3, M4 | PMOS | 12 µm | 1 µm | 
| M5 | NMOS | 20 µm  | 2 µm | 
| M0 | NMOS | 20 µm  | 2 µm  |
| M7 | PMOS | 30 µm | 1 µm |
| M10 | NMOS | 30 µm | 2 µm |  



---

## 4. Design Methodology

The design followed a **gm/ID-based sizing flow**, rather than direct square-law hand calculations, to account for moderate/weak-inversion behavior typical of low-current 65nm analog design:

1. **Spec definition.** Fixed targets: DC gain, GBW (via Cc), phase margin, SR, ICMR, load cap, and a hard power/current budget (Ibias = 20 µA).
2. **Topology selection.** Two-stage Miller chosen over a single-stage telescopic/folded-cascode topology because the ICMR window (0.6 V – 1.0 V) and gain target (≥55 dB) at a constrained supply are difficult to satisfy with a single cascoded stage at this bias current; the two-stage approach trades a second gain stage (and the associated RHP zero / compensation complexity) for relaxed output-swing and ICMR headroom.
3. **Current budgeting.** The 20 µA reference (M0) sets the stage-1 tail current via M5; the stage-2 bias current (M10) is set independently by the mirror ratio between M10 and M0, sized to meet the stage-2 transconductance requirement below.
4. **Stage-1 sizing (gm/ID).**  Using the GBW specification, gm1 was obtained. As ID was defined by the specs, using TSMC 65nm gm/ID characterization curves for the NMOS input device, an operating point (gm/ID) was selected to balance transconductance efficiency against the available 10 µA per side (half of the 20 µA tail). The corresponding W/L was read off the characterization curve for the chosen ID and gm/ID.
5. **Mirror load sizing (M3/M4).** Sized for matching and to keep gds low enough to support the stage-1 gain target Av1 = gm1/(gds2+gds4), while keeping VOV small enough not to eat into the upper ICMR limit.
6. **Compensation capacitor selection.** Cc chosen against the rule-of-thumb lower bound Cc ≥ 0.22·CL for ~60° phase margin in a single-Miller-pole approximation, then iterated against the stage-2 gm condition below.
7. **Stage-2 sizing for phase margin.** gm7 (stage-2 gm) sized to satisfy gm7 ≈ 2.2 · gm(1,2) · (CL/Cc), pushing the non-dominant pole sufficiently beyond the unity-gain frequency to realize the target phase margin.
8. **Slew-rate check.** SR = I(M5)/Cc checked against the 20 V/µs target. 
9. **ICMR / operating-point verification.** DC operating-point analysis across the input common-mode range to confirm every device remains in saturation with adequate VOV margin at both 0.6 V and 1.0 V.
10. **AC verification.** Loop/open-loop AC analysis for DC gain and phase margin.
11. **Transient verification.** Large-signal step response for slew rate and settling time.

---

## 5. Key Design Equations


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
- **Phase at sweep cursor:** −123.1678° 
- **GBW:** ~17MHz
- **SR** 20.35 V/µs
- **CMRR** 57.75 dB

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

## 9. References

- IITK SSCD lectures
- "Systematic Design of Analog CMOS Circuits" by Paul G. A. Jespers and Boris Murmann

---


