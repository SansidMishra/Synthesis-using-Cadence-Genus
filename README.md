# RISC-V Single-Cycle Processor — Synthesis & GLS Flow

This repository documents the complete process of taking a single-cycle RISC-V RV32I processor from RTL (Verilog source code) all the way through **logic synthesis** and **gate-level simulation (GLS)**, using industry-standard EDA tools (Cadence Genus 18.1 and ncverilog) on a 45nm standard cell technology.

This was done as part of an academic VLSI/digital design internship at **IIT Bhubaneswar**. The same flow was first validated on a simpler mod-10 counter design before being applied to a full RISC-V core, so if anything is unclear, that simpler project is a good reference point.

This README is written so that someone with little or no prior exposure to the digital ASIC/VLSI flow can follow along and understand not just *what* commands were run, but *why*.

---

## Table of Contents

1. [Background](#background)
2. [Key Terms Explained](#key-terms-explained)
3. [Tools Used](#tools-used)
4. [Directory Structure](#directory-structure)
5. [Flow Overview](#flow-overview)
6. [Setup Instructions](#setup-instructions)
7. [Running Synthesis](#running-synthesis)
8. [Reading the Reports](#reading-the-reports)
9. [Running Gate-Level Simulation (GLS)](#running-gate-level-simulation-gls)
10. [Viewing Waveforms](#viewing-waveforms)
11. [Results Summary](#results-summary)
12. [Known Limitations](#known-limitations)
13. [Common Errors and Fixes](#common-errors-and-fixes)
14. [Deliverables for Physical Design Handoff](#deliverables-for-physical-design-handoff)
15. [Notes](#notes)

---

## Background

When you design a digital circuit (like a processor), you usually start by describing its behavior in a hardware description language like **Verilog**. This Verilog code is called **RTL** (Register Transfer Level) — it describes *what* the circuit should do (for example, "on every clock edge, update the program counter"), but it is not yet a physical circuit.

To turn that RTL into something that can actually be manufactured as a chip, it has to go through a process called **logic synthesis**. A synthesis tool (here, Cadence Genus) reads the RTL and converts it into a netlist made of real, physical standard logic cells — AND gates, flip-flops, multiplexers, and so on — that exist in a specific manufacturing technology. In this project, that technology is a 45nm process node.

Once the gate-level netlist exists, it is important to verify that it still behaves the way the original RTL did. This is done with **Gate-Level Simulation (GLS)** — simulating the synthesized netlist (instead of the original RTL) and confirming that outputs still match expectations.

In short, this project takes:

```
RTL (riscv.v)
    │
    ▼
Synthesis (Cadence Genus)
    │
    ▼
Gate-Level Netlist (riscv_net.v)
    │
    ▼
Gate-Level Simulation (ncverilog)
    │
    ▼
Waveform Verification (SimVision)
```

---

## Key Terms Explained

| Term | Meaning |
|---|---|
| **RTL** | Register Transfer Level — Verilog/SystemVerilog code describing circuit behaviour |
| **Synthesis** | Converting RTL into a gate-level netlist using a standard cell library |
| **Netlist** | A text file listing actual logic gates and how they are connected |
| **Standard Cell Library (.lib)** | Describes all basic logic gates available in a given manufacturing technology, including timing, power, and area data |
| **45nm** | The manufacturing process node — refers to approximate transistor feature size. 45nm is commonly used in academic settings |
| **SDC** | Synopsys Design Constraints — tells the synthesis tool the timing requirements (clock speed, I/O delays, etc.) |
| **Clock Period** | Duration of one clock cycle. 20 ns period = 1 / 20 ns = 50 MHz |
| **Slack** | Time margin in a timing path. Positive slack = timing met. Negative slack = timing violation |
| **GLS** | Gate-Level Simulation — simulating the post-synthesis netlist to verify correctness |
| **Testbench** | Verilog code that drives inputs into the design and checks outputs during simulation |
| **VCD** | Value Change Dump — records signal transitions over time for waveform viewing |
| **False Path** | A timing path that does not affect real circuit behaviour and should be excluded from timing analysis (e.g. asynchronous reset) |

---

## Tools Used

| Tool | Version | Purpose |
|---|---|---|
| Cadence Genus | 18.10 | RTL synthesis — converts Verilog to gate-level netlist |
| ncverilog | Cadence Incisive | Gate-level simulation |
| SimVision | Cadence | Waveform viewing from VCD files |
| Technology Library | 45nm GPDK | Standard cell library (`slow_vdd1v0_basicCells.lib`) |

All tools were run on a remote lab server at IIT Bhubaneswar (`vlsiws13`), running Red Hat Enterprise Linux 7.9, accessed via a Linux terminal. These are not free or open-source tools — a valid Cadence license is required.

---

## Directory Structure

```
.
├── rtl/
│   ├── riscv.v                  Single-cycle RV32I processor (RTL source)
│   └── riscv_tb.v               Testbench for simulation
├── constraint/
│   └── constraint.sdc           SDC timing constraints (50 MHz clock)
├── synthesis/
│   ├── syn.tcl                  Genus synthesis script
│   ├── riscv_net.v              Synthesized gate-level netlist       [output]
│   ├── constraint_out.sdc       Back-annotated constraints           [output]
│   ├── timing.rpt               Setup/hold slack report              [output]
│   ├── area.rpt                 Cell count and area report           [output]
│   ├── power.rpt                Power estimate report                [output]
│   └── gates.rpt                Gate-type breakdown report           [output]
├── gls/
│   └── riscv.vcd                Waveform dump from GLS run           [output]
└── README.md
```

Files marked `[output]` are generated automatically when you run the scripts — you do not need to create them by hand.

> **Note:** The standard cell library files (`.lib` and `.v` model files) are **not included** in this repository. They contain proprietary foundry IP distributed under a lab license agreement. See [Setup Instructions](#setup-instructions) for how to supply them.

---

## Flow Overview

This is the full sequence from start to finish:

1. **Write the RTL** — `riscv.v` implements a single-cycle RV32I core supporting all six RISC-V instruction types (R, I, S, B, U, J).
2. **Define timing constraints** — `constraint.sdc` specifies a 20 ns clock (50 MHz), input/output delays, clock quality parameters, and a false path for the asynchronous reset.
3. **Write the synthesis script** — `syn.tcl` tells Genus where to find the RTL, library, and constraints, and what commands to run.
4. **Run synthesis** in three passes:
   - `syn_generic` — converts RTL to technology-independent logic
   - `syn_map` — maps generic logic onto real 45nm library cells
   - `syn_opt` — optimizes the mapped netlist for timing and area
5. **Export reports** — timing, area, power, and gate count written to `synthesis/`.
6. **Set up GLS** — copy the synthesized netlist and cell simulation models into `gls/`.
7. **Run gate-level simulation** with `ncverilog` to confirm correct behaviour.
8. **View waveforms** in SimVision to visually verify the processor (e.g. `pc_out` incrementing by 4 each cycle after reset).

---

## Setup Instructions

### 1. Clone this repository

```bash
git clone https://github.com/SansidMishra/RISCV-Synthesis.git
cd RISCV-Synthesis
```

### 2. Supply the foundry library files

These are not included in this repository. Obtain them from your lab or foundry license:

```bash
mkdir -p lib
cp /path/to/slow_vdd1v0_basicCells.lib lib/
cp /path/to/slow_vdd1v0_basicCells.v   gls/
```

If you do not know where they are installed on your system:

```bash
find /cad/FOUNDRY/digital/45nm -name "*.lib" 2>/dev/null | head -20
```

Confirm the library contains real logic cells (not just filler/decap cells):

```bash
grep "cell (" lib/slow_vdd1v0_basicCells.lib | head -10
```

You should see cell names like `INV_X1`, `NAND2_X1`, `DFF_X1`, etc. If you only see filler or decap cells, you have the wrong library file.

### 3. Load the Cadence tool environment

```bash
source /cad/cshrc
```

This sets up the tool paths and license variables. You must run this in every new terminal session before using any Cadence tool.

---

## Running Synthesis

```bash
cd synthesis
genus -f syn.tcl |& tee synthesis.log
```

`tee synthesis.log` saves a copy of all terminal output to a log file — useful for debugging if the window closes unexpectedly.

### Milestone messages to watch for

As synthesis runs, these lines confirm each stage completed successfully:

```
Info : Done synthesizing. [SYNTH-2]
       Done synthesizing 'riscv' to generic gates.     ← syn_generic complete

Info : Done incrementally optimizing. [SYNTH-8]
       Done incrementally optimizing 'riscv'.           ← syn_opt complete
```

If you do not see these, something went wrong. Scroll up in the log to find the first `Error` line, or consult the [Common Errors and Fixes](#common-errors-and-fixes) table.

---

## Reading the Reports

After synthesis, four report files are written to `synthesis/`. View any of them with:

```bash
cat timing.rpt
cat area.rpt
cat power.rpt
cat gates.rpt
```

### Timing Report (`timing.rpt`)

The most important report. Tells you whether the circuit meets your target clock speed.

```
slack (MET)      :  0.397   ← Timing is satisfied — circuit works at 50 MHz
slack (VIOLATED) : -1.230   ← Timing violation — circuit is too slow
```

If `timing.rpt` is empty (0 bytes), it usually means some ports were not properly constrained. Diagnose inside Genus with:

```tcl
report_timing -lint
```

### Area Report (`area.rpt`)

| Field | Meaning |
|---|---|
| Cell Count | Total number of standard cells used |
| Cell Area | Area occupied by logic cells |
| Net Area | Interconnect area (typically 0 before place-and-route) |
| Total Area | Cell Area + Net Area |

### Power Report (`power.rpt`)

| Field | Meaning |
|---|---|
| Leakage Power | Power consumed when circuit is idle |
| Dynamic Power | Power consumed due to signal switching |
| Total Power | Sum of leakage and dynamic |

A full RISC-V processor will show significantly higher dynamic power than a simple counter design due to the larger number of switching gates per clock cycle.

### Gates Report (`gates.rpt`)

Lists which standard cell types were used and how many of each — useful for understanding how the synthesizer implemented your design.

---

## Running Gate-Level Simulation (GLS)

GLS verifies that the synthesized netlist — the actual gates — behaves identically to the original RTL.

### 1. Copy required files into `gls/`

```bash
cd gls
cp ../synthesis/riscv_net.v .
cp /cad/FOUNDRY/digital/45nm/svt/verilog/slow_vdd1v0_basicCells.v .
```

### 2. Run the simulation

```bash
source /cad/cshrc
ncverilog slow_vdd1v0_basicCells.v riscv_net.v ../rtl/riscv_tb.v +access+r
```

> **Important:** File order matters. Always list the cell model file first, then the netlist, then the testbench. Reversing this order causes "module not found" errors because the simulator tries to instantiate cells before they are defined.

`+access+r` enables read access to internal signals, which is required for waveform dumping.

### 3. Expected output

A successful simulation ends with:

```
Simulation complete via $finish(1) at time 540 NS
```

You will also see per-cycle `$monitor` output and a final PASS/FAIL summary from the testbench.

---

## Viewing Waveforms

Once `riscv.vcd` is generated:

```bash
simvision riscv.vcd &
```

### Steps inside SimVision

1. If a database is already open and locked, reopen it from the console:
   ```
   database open -overwrite riscv.vcd
   ```
2. In the **Design Browser** panel (left side), click the triangle next to the top module to expand the hierarchy.
3. Click on `riscv_tb` — its signals (`clk`, `rst_n`, `pc_out`, etc.) appear in the panel below.
4. Select all signals with `Ctrl+A`, then click **"Send to Waveform Viewer"**.
5. Press **F** (or go to `View → Zoom → Fit`) to display the full simulation timeline.

### What to expect

For a correctly working single-cycle RISC-V core running NOPs, `pc_out` should increment by 4 every clock cycle once `rst_n` goes high:

```
Time 0    → pc_out = 0x00000000  (reset active)
Time 40ns → pc_out = 0x00000004  (first instruction fetched)
Time 60ns → pc_out = 0x00000008
Time 80ns → pc_out = 0x0000000C
...
```

This is correct because RISC-V instructions are 4 bytes wide and a single-cycle processor advances the program counter by one instruction per clock cycle.

> If no expand arrow appears next to the top module in the Design Browser, the VCD only captured a flat scope. Check the `$dumpvars` line in the testbench, delete the old `.vcd` file, and re-run the simulation.

---

## Results Summary

## Results Summary

| Metric | Value |
|---|---|
| Clock Period | 20 ns (50 MHz) |
| Worst Slack | +16.057 ns (MET) |
| Critical Path Delay | 3.8 ns |
| Total Standard Cells | 88 |
| Total Cell Area | 283.860 units |
| Sequential Cells (FF) | 30 (65.3% of area) |
| Logic Cells | 57 (34.5% of area) |
| Leakage Power | 5.440 nW |
| Dynamic Power | 9368.974 nW |
| Total Power | 9374.414 nW (~9.37 µW) |
| GLS Result | Pending |
---

## Known Limitations

These are design decisions made intentionally for simplicity at this stage of learning. They would need to be addressed before any real tapeout.

- **Branch condition logic is simplified** — the current RTL checks only `alu_zero` for all branch types. Full correctness requires separate condition checks for BNE, BLT, BGE, BLTU, and BGEU.
- **LUI implementation** — the ALU operand mux for LUI uses `imm_u` correctly only when `alu_src` is also set in the control unit. Verify this is set correctly in `control_unit.v`.
- **No pipeline** — this is a single-cycle design. Every instruction takes exactly one clock cycle regardless of complexity.
- **No hazard detection** — since it is single-cycle, there are no data or control hazards to handle.
- **No exception handling** — illegal instructions and misaligned memory accesses are not handled.
- **Fixed memory size** — instruction memory is 256 words (1 KiB) and data memory is 1 KiB. These are sufficient for simulation but not for real programs.
- **Hold timing not fully constrained** — the SDC currently specifies `-min` delays. Verify that `constraint.sdc` includes `set_input_delay -min` and `set_output_delay -min` for complete hold analysis.

---

## Common Errors and Fixes

| Error | Cause | Fix |
|---|---|---|
| `Multiple designs are available. Specify the design you want to use.` | `read_sdc` or `report_*` run without a current design selected | Add `set_db design:riscv .current 1` before `read_sdc`, or use `current_design riscv` |
| `Cannot perform synthesis — libraries have no usable inverters. [LBR-171]` | The `.lib` file has no real logic cells (filler-only or decap-only library) | Confirm with `grep "cell (" library.lib` — you need cells like INV, NAND, DFF |
| `An option named '-file' could not be found.` | Genus 18.10 does not support `write_hdl -file` | Use shell redirection: `write_hdl > riscv_net.v` |
| `cp: cannot create regular file: No such file or directory` | Target directory does not exist | Run `mkdir -p <directory>` before copying |
| Abnormal exit / segfault after `gui_show` | Genus GUI crashed due to X11/display issue (common over SSH) | Skip the GUI entirely — run `genus -f syn.tcl` from the terminal |
| `Could not interpret SDC command. [SDC-202]` | Bus port references like `out[0]` are misinterpreted by Tcl (square brackets are Tcl syntax) | Wrap in curly braces: `{out[0]}` — or use `all_inputs` / `all_outputs` |
| `timing.rpt` is empty | Ports not properly constrained — Genus found no timing paths to report | Run `report_timing -lint` inside Genus to identify unconstrained paths |
| `module not found` in ncverilog | Cell model file listed after the netlist in the compile command | Always list `slow_vdd1v0_basicCells.v` first in the ncverilog command |
| Negative slack after synthesis | Clock period too tight for the logic depth in the design | Increase clock period in `constraint.sdc` (e.g. change 20 ns to 25 ns) |

---

## Deliverables for Physical Design Handoff

Once synthesis and GLS are both verified, these files are handed off to the next stage (place-and-route / physical design):

| File | Purpose |
|---|---|
| `synthesis/riscv_net.v` | Gate-level netlist — the synthesized design |
| `synthesis/constraint_out.sdc` | Back-annotated timing constraints for P&R tool |
| `synthesis/timing.rpt` | Timing closure baseline for the P&R team |
| `synthesis/area.rpt` | Area estimate for floorplanning |
| `synthesis/power.rpt` | Power estimate for PDN (power delivery network) planning |

---

## Notes

- `set_db design:riscv .current 1` or `current_design riscv` must be called before `read_sdc` to avoid the "Multiple designs" error in Genus 18.10.
- Genus 18.10 does not support the `write_hdl -file` flag seen in some tutorials. Use shell redirection (`write_hdl > riscv_net.v`) instead.
- Bus-style port references in SDC (e.g. `out[0]`) must be wrapped in `{}` or replaced with `all_inputs` / `all_outputs`.
- Remove `\`timescale` directives from `riscv.v` before synthesis — they are simulation-only constructs and do not belong in synthesis RTL.
- The 20 ns (50 MHz) clock was chosen as a conservative starting point for a single-cycle 45nm design. If timing is violated, increase the period in `constraint.sdc` before debugging the logic path.
- This flow was validated end-to-end on a simpler mod-10 counter before being applied to the RISC-V core. Working through that simpler example first is recommended if you are new to this flow.
