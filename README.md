# Blinky 5.1

![KiCad](https://img.shields.io/badge/ECAD-KiCad_9.0-314CB0?style=flat-square&logo=kicad&logoColor=white)
![Layers](https://img.shields.io/badge/PCB-2--Layer_FR--4-2E7D32?style=flat-square)
![Topology](https://img.shields.io/badge/Circuit-CMOS_7555_Astable-D97706?style=flat-square)
![Status](https://img.shields.io/badge/Status-Fab_Ready_(DRC_Clean)-success?style=flat-square)

This is an implementation of Contextual Electronics course "Getting to Blinky 5.0", this is my first fab-ready start-to-finish PCB. I added my touch with a 500 kΩ potentiometer, to adjust the simetrical speed of the charge-discharge capacitor cycle.

---

## 1. Design Overview & Hardware Modifications

While the baseline *Getting to Blinky 5.0* uses a fixed timing 470 kΩ resistor (R<sub>2</sub>), this revision replaces R<sub>2</sub> with a **SMD 500 kΩ linear trimmer potentiometer (`RV1`, Bourns 3314G)**. This allows continuous hardware adjustment of the oscillation frequency across three orders of magnitude (1.44 Hz to 1.44 kHz) without firmware or external instrumentation.

### fun facts :)
- **100% Top-Side SMD Component Placement:** Keeps the bottom layer (`B.Cu`) free of through-hole leads, eliminating mechanical interference with the rear-mounted CR2032 battery clip (`BT1`) and preserving ground plane continuity.
- **Split Power/Ground Plane Strategy:** Dedicated `+3V` (`VDD`) copper pour on `F.Cu` paired with a solid `GND` reference plane on `B.Cu`.
- **Hand-Soldering Optimization:** Uses IPC-compliant `_HandSolder` extended pads for all `0805` passives, 4-spoke thermal reliefs on high-mass copper connections, and via-in-pad avoidance (*dog-bone* trace offsets) to prevent solder wicking.

---

## 2. Circuit Architecture & Theory of Operation

The circuit operates as a classic **CMOS 555 Astable Multivibrator** powered by a nominal 3.0 V lithium coin cell (`BT1`).

### Functional Block Breakdown
1. **Internal Voltage Divider (3 × 5 kΩ):** Establishes ratiometric switching thresholds at 1/3 V<sub>DD</sub> (1.0 V) and 2/3 V<sub>DD</sub> (2.0 V). Because capacitor charging follows an exponential curve proportional to V<sub>DD</sub>, supply voltage variations cancel out, keeping oscillation frequency stable as the CR2032 discharges.
2. **Charge Phase (`OUT` = HIGH, `DISCHARGE` = High-Z):**
   * When V<sub>C1</sub> < 1/3 V<sub>DD</sub>, the lower comparator sets the internal SR flip-flop.
   * `U1` Pin 3 (`OUT`) drives `+3V` through current-limiting resistor `R3` (1 kΩ), forward-biasing `D1`.
   * Simultaneously, `C1` (1 µF) charges from `VDD` through `R1` (1 kΩ) in series with `RV1` (0–500 kΩ).
3. **Discharge Phase (`OUT` = LOW, `DISCHARGE` = GND):**
   * Once V<sub>C1</sub> reaches 2/3 V<sub>DD</sub>, the upper comparator resets the flip-flop.
   * `U1` Pin 3 drops to 0 V (turning off `D1`), and Pin 7 (`DISCHARGE`) saturates an internal open-drain MOSFET to `GND`.
   * `C1` discharges exclusively through `RV1` into Pin 7 until V<sub>C1</sub> drops back to 1/3 V<sub>DD</sub>, restarting the cycle.
4. **Fail-Safe Rheostat & Short-Circuit Protection:**
   * `RV1` Pin 3 is shorted directly to Pin 2 (*wiper*). If the mechanical wiper momentarily loses contact during adjustment, the resistance defaults to the full 500 kΩ track rather than creating an open circuit.
   * `R1` (1 kΩ) remains fixed in series with `VDD`, limiting maximum sink current into Pin 7 to 3.0 mA even when `RV1` is dialed to 0 Ω.

---

## 3. Mathematical Modeling & Timing Envelope

The charge (t<sub>HIGH</sub>) and discharge (t<sub>LOW</sub>) intervals are governed by the RC time constant τ = R · C between the 1/3 V<sub>DD</sub> and 2/3 V<sub>DD</sub> thresholds (ln(2) ≈ 0.693):

$$
t_{\mathrm{HIGH}} = \ln(2) \cdot (R_1 + R_{V1}) \cdot C_1
$$

$$
t_{\mathrm{LOW}} = \ln(2) \cdot R_{V1} \cdot C_1
$$

$$
f = \frac{1}{t_{\mathrm{HIGH}} + t_{\mathrm{LOW}}} = \frac{1}{\ln(2) \cdot (R_1 + 2R_{V1}) \cdot C_1}
$$

$$
\mathrm{Duty\ Cycle\ (\%)} = \frac{R_1 + R_{V1}}{R_1 + 2R_{V1}} \times 100\%
$$

### Operating Sweep (R<sub>1</sub> = 1 kΩ, C<sub>1</sub> = 1 µF)

| `RV1` Position | Effective R<sub>V1</sub> | t<sub>HIGH</sub> | t<sub>LOW</sub> | Frequency (f) | Duty Cycle | Visual Behavior |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| **0% (CCW Min)** | ~0 Ω | 0.69 ms | ~0.02 ms* | **~1.44 kHz** | ~97.2% | Appears continuously lit (above flicker fusion threshold) |
| **50% (Mid-Span)** | 250 kΩ | 173.9 ms | 173.3 ms | **2.88 Hz** | 50.1% | Fast, symmetric rhythmic blink |
| **100% (CW Max)** | 500 kΩ | 347.2 ms | 346.5 ms | **1.44 Hz** | 50.05% | Slow, distinct pulse (~1.4 blinks/sec) |

*\*Bounded by the internal `DISCHARGE` MOSFET on-resistance (R<sub>DS(on)</sub> ≈ 20–30 Ω).*

---

## 4. PCB Stackup, Copper Pours & Routing Architecture

### 2-Layer Standard FR-4 Stackup
| Layer | Name | Type | Copper Weight | Role & Plane Assignment |
| :--- | :--- | :--- | :--- | :--- |
| **Top** | `F.Cu` | Signal / Power | 1 oz (35 µm) | Component mounting, signal routing (`DIS`, `THR`, `OUT`), and **`VDD` (+3V) copper pour** |
| **Core** | Dielectric | FR-4 (ε<sub>r</sub> ≈ 4.4) | — | 1.6 mm (63 mil) mechanical substrate |
| **Bottom** | `B.Cu` | Ground / Power | 1 oz (35 µm) | Battery clip `BT1`, `VDD` input pads, and continuous **`GND` (0V) reference plane** |

### Layout & Routing Decisions
- **Via Architecture (5 Total Vias):**
  * **2 Power Stitching Vias:** Route `+3V` from the left and right `BT1` positive pads on `B.Cu` up to the `VDD` copper pour on `F.Cu`.
  * **3 Ground Return Vias:** Connect the ground pads of `U1` (Pin 1), `C1` (Pin 2), and `D1` (Pin 1) directly into the solid `B.Cu` `GND` plane.
- **Anti-Wicking Dog-Bones:** Every via is offset from its parent SMD pad via a short trace segment rather than placed inside the pad, preventing molten solder from wicking down the barrel during manual reflow/iron assembly.
- **Thermal Reliefs:** All zone connections (`VDD` on `F.Cu` and `GND` on `B.Cu`, including the large central `BT1` ground pad) use **4-spoke thermal reliefs** to prevent heat sinking into the copper planes during hand soldering.
- **45° Routing Geometry:** All signal traces (`DIS`, `THR`, `OUT`) are routed exclusively on `F.Cu` using 45° corners and generous clearance around the `SOIC-8` gull-wing leads.

---

## 5. Signal Integrity (SI), Impedance & Power Integrity (PI) Analysis

### Microstrip Characteristic Impedance & Transmission Line Regime
Assuming a standard 1.6 mm FR-4 core (ε<sub>r</sub> = 4.4), 1 oz copper (T = 0.035 mm), and standard signal trace width W = 0.25 mm (9.84 mils) referenced over the continuous `B.Cu` `GND` plane:

- **Characteristic Impedance (Z<sub>0</sub>):**
  * **Hammerstad–Jensen Microstrip Model:** Z<sub>0</sub> = **131.0 Ω** (ε<sub>eff</sub> = 2.90)
  * **IPC-2141 Approximation:** Z<sub>0</sub> = **133.8 Ω**
- **Propagation Delay (t<sub>pd</sub>):**

$$
t_{pd} = \frac{\sqrt{\varepsilon_{\mathrm{eff}}}}{c} \approx 5.68\ \mathrm{ps/mm}
$$

- **Critical Length & Lumped-Element Verification:**
  For a CMOS `7555` output transition time of t<sub>r</sub> ≈ 100 ns (signal bandwidth BW ≈ 0.35 / t<sub>r</sub> = 3.5 MHz), the critical transmission-line length threshold (l<sub>crit</sub> = t<sub>r</sub> / (6 · t<sub>pd</sub>)) is **2.93 m**. Since the longest physical trace on this PCB is **< 15 mm** (0.5% of l<sub>crit</sub>), reflections are negligible and controlled-impedance termination is unnecessary.

### Power Budget, Ampacity & Battery Runtime
- **Trace Ampacity (IPC-2221):** A 0.25 mm external 1 oz copper trace supports ~800 mA for a ΔT = 10 °C rise—providing a **>260× current margin** over the circuit's 3.0 mA worst-case branch current.
- **Current Consumption & CR2032 Runtime (~220 mAh capacity):**
  * Quiescent IC current (I<sub>Q</sub>): ~60 µA
  * LED drive current (V<sub>DD</sub> = 3.0 V, V<sub>F</sub> ≈ 1.9 V, R<sub>3</sub> = 1 kΩ): I<sub>LED</sub> = (3.0 − 1.9) / 1000 = 1.1 mA (active during `HIGH`)
  * Discharge sink current through `R1` (1 kΩ): I<sub>R1</sub> = 3.0 / 1000 = 3.0 mA (active during `LOW`)
  * Average current at 50% duty cycle: I<sub>avg</sub> ≈ 0.06 + 0.5(1.1) + 0.5(3.0) = **2.11 mA**
  * **Estimated Continuous Runtime:** 220 mAh / 2.11 mA ≈ **104 hours** (~4.3 days continuous operation).

---

## 6. Bill of Materials (BOM) & Footprint Mapping

| Ref | Value | Description | Package | KiCad Footprint Assignment | Layer |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **`U1`** | `7555` | CMOS Low-Power Timer (LMC555 / TLC555) | SOIC-8 | `Package_SO:SOIC-8_3.9x4.9mm_P1.27mm` | `F.Cu` |
| **`RV1`** | `500K` | Linear SMD Trimmer Potentiometer (1-Turn) | 4.5×4.5 mm | `Potentiometer_SMD:Potentiometer_Bourns_3314G_Vertical` | `F.Cu` |
| **`R1`** | `1K` | Thick-Film Chip Resistor (1%, 1/8 W) | 0805 | `Resistor_SMD:R_0805_2012Metric_Pad1.20x1.40mm_HandSolder` | `F.Cu` |
| **`R3`** | `1K` | Thick-Film Chip Resistor (1%, 1/8 W) | 0805 | `Resistor_SMD:R_0805_2012Metric_Pad1.20x1.40mm_HandSolder` | `F.Cu` |
| **`C1`** | `1U` | MLCC Ceramic Capacitor (16 V, X7R/X5R) | 0805 | `Capacitor_SMD:C_0805_2012Metric_Pad1.18x1.45mm_HandSolder` | `F.Cu` |
| **`D1`** | `LED` | Indicator LED (Red/Green/Amber, V<sub>F</sub> ≈ 1.9 V) | 0805 | `LED_SMD:LED_0805_2012Metric_Pad1.15x1.40mm_HandSolder` | `F.Cu` |
| **`BT1`** | `CR2032` | 3V Coin-Cell Battery Retainer Clip | SMD | `getting_to_blinky:S8211-46R` | `B.Cu` |

---

## 7. DFM Verification & Hardware Bring-Up Procedure

### Pre-Fabrication Checklist
- [x] **ERC (Electrical Rules Checker):** Zero errors/warnings (`PWR_FLAG` assigned on `VDD` and `GND`; Pin 5 `CTRL` flagged `No-Connect`).
- [x] **DRC (Design Rules Checker):** Zero clearance violations, zero unrouted nets, and silkscreen text isolated from exposed solder mask openings.
- [x] **Gerber Inspection:** Verified `F.Cu`, `B.Cu`, `F.Mask`, `B.Mask`, `F.Silkscreen`, `B.Silkscreen`, `Edge.Cuts`, and Excellon drill files.
