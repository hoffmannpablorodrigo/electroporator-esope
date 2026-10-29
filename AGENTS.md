# AGENTS.md — Operational Manual for AI Agents (PCB Engineering in KiCad 10)

> **WARNING — CRITICAL SAFETY RULE (HIGH VOLTAGE — 800 VDC):**
> This repository contains the hardware design for a **High-Voltage Monopolar Electroporator (800 VDC)** intended for biomedical applications (ESOPE clinical protocol). Topological errors, accidental reduction of Clearance/Creepage distances, direct coordinate editing in `.kicad_pcb` files, or incorrect routing of the IGBT Gate loop can cause **lethal electric arcs, semiconductor explosion, destruction of the control stage, or severe hazard to the operator.**
> **No AI agent is authorized to perform automatic routing on 800 VDC nets, alter geometric track coordinates directly via text in `.kicad_pcb`, or relax isolation rules without deterministic validation and explicit human approval.**

---

## 1. Project Identity and Objective

### 1.1. Purpose and Application
* **Equipment:** High-voltage monopolar electroporator (**800 VDC**) designed according to the clinical standard **ESOPE** (*European Standard Operating Procedures of Electrochemotherapy*) for biomedical research and oncological treatment at the Applied Instrumentation and Measurement group (IMA / B.S. Electrical Engineering Thesis — IFSC Itajaí Campus).
* **Functional Architecture:** The system switches high-voltage rectangular pulses (up to 800 VDC) across a biological load (cuvette/electrodes) using a solid-state IGBT switch in a low-side configuration (`IRG4PH50KD`). This switch is driven by a smart optocoupler gate driver (`ACPL-333J`) featuring integrated desaturation protection (*DESAT*), Active Miller Clamping, and fault reporting ($\overline{\text{FAULT}}$), controlled and timed by an Arduino microcontroller.

### 1.2. Current Development State
* **Confirmed:**
  * Functional bench prototype assembled and tested on a breadboard integrating an Arduino UNO, `ACPL-333J` driver on a SOIC-16 adapter board, `IRG4PH50KD` IGBT (`TO-247`), axial diodes (`SKA 1/17`, `MUR460`, `MUR4100`), and a green tubular wirewound power resistor (10W).
  * Design calculations for track width, isolation distances (*Clearance* and *Creepage*), THT/SMD footprint selection, hole size adjustments, and Netclass rules with automatic wildcard assignments (`*HV*`, `*PWR*`, `+*`) are already validated for **KiCad 10**.
* **Current Phase:** Final transition from schematic capture (`.kicad_sch`) to strategic component placement, construction of the physical galvanic barrier (2.0 mm milled slot on `Edge.Cuts` + 10.0 mm Keepout zone), and manual routing on a **double-sided PCB (2 layers: `F.Cu` and `B.Cu`)**.
* **Expected Deliverables:**
  1. Electrical schematic (`.kicad_sch`) 100% free of ERC errors (`0 Errors`), including the topological correction of the shunt resistor ($R_{\text{shunt}}$).
  2. 2-layer PCB layout (`.kicad_pcb`) manually routed, with segregated ground planes (`GND` and `PWR_GND`), passing DRC (`0 Errors`, `0 Unconnected Items`).
  3. Complete manufacturing package in `gerbers/` (Gerber files for layers `F.Cu`, `B.Cu` + auxiliary layers and Excellon `.drl` drill files).

---

## 2. Technical Context and Engineering Specifications

### 2.1. Voltage Domains and Galvanic Isolation
The circuit is physically divided into two strictly isolated galvanic domains that **must never share conductors or reference planes:**

| Domain | Main Nets | Nominal Voltage | Reference (Ground) | Components in Domain |
| :--- | :--- | :--- | :--- | :--- |
| **Primary (Control / Logic)** | `+5V`, Arduino digital signals, `FAULT`, LED anode/cathode | 5 VDC | **`GND`** | Arduino (Mega 2560 / UNO), `LM7805`, driver input resistors, `TMA-0512D` primary pins, `ACPL-333J` pins 1 to 8. |
| **Secondary (Gate Drive)** | `+18V`, `PWR_24V`, `VOUT` (*Gate*), `VCLAMP`, `DESAT` | +18 VDC ($\approx 15.5\text{ V}$ at Gate) | **IGBT Emitter Node** ($V_E / V_{EE}$) | `ACPL-333J` pins 9 to 16, `TMA-0512D` isolated output, `LM7818` regulator, $0.1\text{ }\mu\text{F}$ bypass cap, $R_G$ ($10\text{ }\Omega$), $C_{\text{BLANK}}$ ($100\text{ pF}$). |
| **Secondary (Power / Pulse)** | `/HV0F`, `/HV0N`, `/HV1B`, `/HV2B`, and `*HV*` | **Up to 800 VDC** | **`PWR_GND`** | Bridge rectifier, 800 VDC capacitor bank, IGBT Collector/Emitter, HV diodes, power resistor, $R_{\text{shunt}}$, and electrode terminals. |

### 2.2. Critical Topological Rule: ACPL-333J Driver, IGBT Emitter, and $R_{\text{shunt}}$
* **STRICT PROHIBITION:** When the current measurement resistor ($R_{\text{shunt}}$) is inserted between the IGBT Emitter and `PWR_GND`, pins **9 ($V_{EE}$), 12 ($V_{EE}$), and 16 ($V_E$)** of the `ACPL-333J` **CANNOT** be connected to `PWR_GND`.
* **Technical Justification:** During the discharge pulse, the peak current $I_{\text{peak}}$ generates a voltage drop $V_{\text{shunt}} = R_{\text{shunt}} \cdot I_{\text{peak}}$. If the driver is referenced to `PWR_GND` (below the shunt), the effective Gate-Emitter voltage drops to $V_{GE} = V_{\text{driver}} - V_{\text{shunt}}$, destroying the component via thermal failure.
* **Mandatory Implementation:**
  1. Pins **9, 12, and 16** of the `ACPL-333J` (along with the +18V bypass capacitor) must close the loop **directly at the IGBT Emitter terminal node** (Kelvin connection on the high side of $R_{\text{shunt}}$).
  2. The **$R_{\text{shunt}}$** must be **strictly non-inductive** and placed between the IGBT Emitter and the `PWR_GND` plane.

### 2.3. Critical Components and Validated Footprints
| Component | Circuit Function | Validated Footprint in KiCad 10 | Physical Parameters and Adjustments |
| :--- | :--- | :--- | :--- |
| **`ACPL-333J`** | Optocoupler Gate Driver | `Package_SO:SOIC-16W_7.5x10.3mm_P1.27mm` | Pitch: 1.27 mm; Gap between pads: 0.63 mm; Internal clearance: **7.31 mm**. Pin 15 is NC. |
| **`IRG4PH50KD`** | 1200V IGBT | `Package_TO_SOT_THT:TO-247-3_Vertical` | **Flush mounting:** Hole size = 2.4 mm, Oval, Pad size = 2.8x4.0 mm. |
| **`MUR4100`** | 1000V Ultrafast Diode | `MUR4100ERLG_footprint:DIOAD1950...` | **Mandatory adjustment:** Hole size = 1.6 mm, Pad size = 2.6 mm. |
| **`MUR460`** | 600V Ultrafast Diode | `MUR460_footprint:DIOAD1990W...` | Verify Hole size $\ge 1.5\text{ mm}$. |
| **Power Resistor** | Load / Power Limiter | `Resistor_THT:R_Axial_Power_L38.0mm_W6.4mm_P45.72mm_Horizontal` | Pitch of **45.72 mm** (18 breadboard holes). |

### 2.4. Netclass Configuration and Netclass Assignments
* **`Potencia`**: Clearance `2.5 mm`, Width `4.0 mm` (In open PCB areas, manually enforce 10.0 mm). Captures `*HV*`.
* **`Alimentacao`**: Clearance `0.5 mm`, Width `0.8 mm`. Captures `*PWR*` (except `PWR_GND`) and `+*`.
* **`Default`**: Clearance `0.25 mm`, Width `0.3 mm`. **Mandatory** for Gate loop, `GND`, `PWR_GND`, and IGBT Emitter node.

---

## 3. Verified Development Environment

* **OS:** Linux **Ubuntu 26.04 LTS ("Resolute Raccoon")**, **Zsh** shell.
* **EDA Suite:** **KiCad 10** (`kicad-cli`).
* **Editor and AI Agents:** **VS Code** with **Antigravity / Roo Code**, **Python 3**.
* **Host Tools:** Git, `rclone bisync` (`--fast-list`, excluding `.git`).

---

## 4. Fundamental Work Rules for AI Agents

1. **Context-First Auditing Rule (MANDATORY):** Whenever you receive a prompt to audit the project, **you MUST FIRST read the local datasheets (`ACPL-333J-Avago.pdf` and `IRG4PH50KD.PDF`)** and read all necessary source files (`.kicad_pro`, `.kicad_sch`, `.kicad_pcb`). You must acquire the total context of the project before executing any auditing command, script, or analysis. Do not rely on assumptions.
2. **Prohibition of Blind Text Routing and Placement:** It is strictly prohibited to create or move copper tracks (`(segment ...)`), zone polygons, or component coordinates by directly editing the `.kicad_pcb` text file. Language models lack 2D/3D geometric solvers and will violate 800 VDC isolation barriers.
3. **No Version Control Fallback:** Because this project uses `rclone bisync` instead of standard Git (`git status` returns error), **there is no git fallback**. You must NEVER modify files without explicitly creating a backup copy via `cp` first.
4. **Prohibition of Artificial Error Suppression:** Never hide ERC or DRC violations by changing severities to *Ignore* or reducing global `Clearance`.

---

## 5. Engineering Operational Process (Auditing Flow)

1. **Read Request & Gather Context:** Read `ACPL-333J-Avago.pdf`, `IRG4PH50KD.PDF`, and the `.kicad_*` files.
2. **Verify Tooling:** Run `kicad-cli --version`.
3. **Connectivity Audit:** Export the netlist (`kicad-cli sch export netlist`) and parse it to ensure nodes like the IGBT Emitter, driver pins 9/12/16, and $R_{\text{shunt}}$ terminals are topologically correct.
4. **Deterministic Validation:** Run `kicad-cli sch erc --format json` and `kicad-cli pcb drc --schematic-parity --format json`.
5. **Evidence Reporting:** Report the exact command, tool version, and the geometric coordinates of any violations.

---

## 6. Specific PCB Design Rules for this Electroporator (800 VDC)

* **Mandatory Milled Slot:** On `Edge.Cuts`, a closed rectangular cutout of **2.0 mm width by 13.5 mm height** must exist exactly under the center of the `ACPL-333J`.
* **Copper Exclusion Zone (Keepout):** An active Rule Area on `F.Cu` and `B.Cu` prohibiting copper with a width of **7.31 mm** under the `ACPL-333J` and **10.0 mm** along the rest of the isolation barrier.
* **Gate Loop Area:** Forward and return tracks between the driver and IGBT must run close together and short on `F.Cu` to minimize parasitic loop inductance.
* **800 VDC Open Routing:** Edge-to-edge spacing between opposite-polarity 800 VDC conductors must be manually maintained at **$\ge 10.0\text{ mm}$**.
* **Segregated Ground Planes:** `GND` (Primary) and `PWR_GND` (Secondary) must be isolated. `PWR_GND` must be manually pulled back by 10.0 mm around all 800 VDC live islands.

---

## 7. Automated Validation and Tool Limitations

* The `Potencia` class DRC is set to **`2.5 mm`** to enable access to the `TO-247` terminals. **`kicad-cli pcb drc` WILL NOT flag an error if two 800 VDC tracks are 3.0 mm apart in the middle of the board!** The 10.0 mm open-area rule must be audited manually.
* DRC checks logical netlist connectivity but **cannot detect inductive coupling on the shunt return**.

---

## 8. Technical References and Local Documents

* **ACPL-333J Datasheet:** Read local file `ACPL-333J-Avago.pdf` for pinout, thermal grounding, and isolated fault sensing requirements.
* **IRG4PH50KD Datasheet:** Read local file `IRG4PH50KD.PDF` for switching characteristics and physical dimensions.
* **KiCad CLI Official Documentation:** [https://docs.kicad.org/master/en/cli/cli.html](https://docs.kicad.org/master/en/cli/cli.html)
* **IEC 60601-1 / IEC 61010-1 Standards:** Normative safety references for *Clearance* and *Creepage* at 800 VDC.