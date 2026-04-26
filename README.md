# FP4 Arithmetic Unit — Caravel User Project

[![License](https://img.shields.io/badge/License-Apache%202.0-blue.svg)](https://opensource.org/licenses/Apache-2.0)
[![CI](../../actions/workflows/user_project_ci.yml/badge.svg)](../../actions/workflows/user_project_ci.yml)
[![Caravel Build](../../actions/workflows/caravel_build.yml/badge.svg)](../../actions/workflows/caravel_build.yml)

---

## Project Overview

This repository contains the **Caravel harness integration** of a
custom **FP4 E2M1 Arithmetic Unit** — a 4-bit floating point
hardware accelerator designed for AI inference acceleration on
edge devices.

The FP4 E2M1 format (1-bit sign, 2-bit exponent, 1-bit mantissa)
is the same format used in **NVIDIA's Blackwell GB200 GPU** for
transformer model inference at ultra-low precision.

**Designer:** Asad Ali
**Institution:** Quaid-e-Awam University of Engineering Sciences
and Technology (QUEST), Nawabshah, Pakistan
**Email:** asadshar0123@gmail.com
**GitHub:** [Asadkhan282](https://github.com/Asadkhan282)
**LinkedIn:** [asad-ali-4932a028b](https://linkedin.com/in/asad-ali-4932a028b)

---

## What is FP4 E2M1?

FP4 is a 4-bit floating point format that packs a number into:

```
Bit 3     Bits 2:1     Bit 0
┌───────┬────────────┬──────────┐
│ Sign  │  Exponent  │ Mantissa │
│  1b   │    2b      │   1b     │
└───────┴────────────┴──────────┘
         Bias = 1
```

**All 16 representable values:**

| Bits | Value | Bits | Value |
|------|-------|------|-------|
| 0000 | 0.000 | 1000 | -0.000 |
| 0001 | 0.250 | 1001 | -0.250 |
| 0010 | 1.000 | 1010 | -1.000 |
| 0011 | 1.500 | 1011 | -1.500 |
| 0100 | 2.000 | 1100 | -2.000 |
| 0101 | 3.000 | 1101 | -3.000 |
| 0110 | 4.000 | 1110 | -4.000 |
| 0111 | 6.000 | 1111 | -6.000 |

---

## Why FP4 for AI?

| Format | Bits | Memory Bandwidth | vs FP4 |
|--------|------|-----------------|--------|
| FP32   | 32   | 8× more         | 8× slower |
| FP16   | 16   | 4× more         | 4× slower |
| FP8    | 8    | 2× more         | 2× slower |
| **FP4 (this)** | **4** | **1× baseline** | **fastest** |

FP4 reduces memory bandwidth by **4× vs FP16** and **8× vs FP32**
— enabling larger AI models on power-constrained edge hardware.

---

## Design Architecture

### Core Design — Precomputed ROM Lookup

Since FP4 has only 16 values, all 16×16 = 256 arithmetic results
are precomputed offline and stored in a combinational ROM
implemented as a synthesisable Verilog `case` statement.

This eliminates all floating-point arithmetic from the critical
path — only a ROM lookup delay separates input from output.

### Pipeline

```
       clk (200 MHz / 5ns period)
        │
        │   Stage 1         Stage 2          Stage 3
a[3:0] ─┤ ┌──────────┐   ┌────────────┐   ┌──────────┐
b[3:0] ─┼►│ Input FF │──►│ 256-entry  │──►│Output FF │──► result[3:0]
valid  ─┤ │ addr<=a,b│   │ Comb. ROM  │   │          │──► valid_out
         │ └──────────┘   └────────────┘   └──────────┘
         │
         Latency: 2 cycles | Throughput: 1 result/cycle
```

### Module Hierarchy

```
user_project_wrapper  (Caravel interface)
└── fp4_top           (FP4 top-level)
    ├── fp4_mul        (FP4 multiplier — 256-entry ROM)
    └── fp4_add        (FP4 adder     — 256-entry ROM)
```

### Caravel Interface

The FP4 unit is connected to the Caravel harness through the
wishbone bus and logic analyser signals:

```
Caravel Signals Used:
├── wb_clk_i          → clk (system clock)
├── wb_rst_i          → rst (system reset)
├── la_data_in[3:0]   → a[3:0] (FP4 operand A)
├── la_data_in[7:4]   → b[3:0] (FP4 operand B)
├── la_data_in[8]     → op (0=multiply, 1=add)
├── la_data_in[9]     → valid_in
├── la_data_out[3:0]  → result[3:0] (FP4 result)
└── la_data_out[4]    → valid_out
```

---

## Implementation Results

### Physical Design Sign-off

| Metric | Result | Status |
|--------|--------|--------|
| Clock Frequency | 200 MHz | ✅ |
| WNS | Positive | ✅ |
| TNS | 0.00 ns | ✅ |
| DRC Violations | 0 | ✅ |
| LVS Result | Circuits Match | ✅ |
| Antenna Violations | 0 | ✅ |

### Floorplan

| Metric | Value |
|--------|-------|
| Die Area | 200 × 200 µm |
| Core Area | 180 × 180 µm |
| Core Utilization | 25% |
| Process | SKY130A (130nm) |
| Standard Cell Library | sky130_fd_sc_hd |

---

## GDSII Layout

![GDSII Layout](fp4_gdsii.png)

*Fabrication-ready GDSII layout viewed in KLayout 0.30.8.
Dense SKY130 standard-cell rows with Met1–Met5 metal
routing visible. I/O pins on all four sides.*

---

## RTL-to-GDSII Flow

```
Verilog RTL
    │
    ▼ Yosys
Gate-level Netlist (SKY130 std cells)
    │
    ▼ OpenROAD Floorplan
Die: 200×200µm | Core: 180×180µm
    │
    ▼ OpenROAD Placement
Global + Detailed (density 25%)
    │
    ▼ TritonCTS
Clock Tree Synthesis
    │
    ▼ TritonRoute
Global + Detailed Routing (Met1–Met5)
    │
    ▼ OpenSTA
Post-route STA — 200 MHz ✅
    │
    ▼ Magic + Netgen
DRC: 0 violations ✅  |  LVS: Match ✅
    │
    ▼
GDSII — fp4_top.gds ✅
```

---

## Tools Used

| Tool | Purpose |
|------|---------|
| OpenLane | RTL-to-GDSII automated flow |
| Yosys | Logic synthesis |
| OpenROAD | Floorplan, placement, CTS |
| TritonRoute | Detailed routing |
| OpenSTA | Static timing analysis |
| Magic VLSI | DRC verification |
| Netgen | LVS verification |
| KLayout | GDSII layout viewing |
| SKY130A PDK | 130nm open-source process |
| Python 3 | LUT table generator |
| Docker | OpenLane container |

---

## How to Run

### Prerequisites

```bash
# Clone OpenLane
git clone https://github.com/The-OpenROAD-Project/OpenLane.git
cd OpenLane
make
make test
```

### Setup

```bash
# Clone this repo
git clone https://github.com/Asadkhan282/caravel_fp4_arithmetic.git

# Copy design into OpenLane
cp -r openlane/ ~/OpenLane/designs/fp4_arithmetic/

# Generate Verilog files
cd ~/OpenLane/designs/fp4_arithmetic
python3 generate_lut.py
```

### Run Flow

```bash
cd ~/OpenLane
sudo make mount
```

Inside Docker:

```bash
./flow.tcl -design fp4_arithmetic
```

### Verify Results

```bash
# Timing
grep "WNS\|TNS" designs/fp4_arithmetic/runs/RUN_*/reports/signoff/*sta*.rpt

# DRC
cat designs/fp4_arithmetic/runs/RUN_*/reports/magic/*drc*.rpt

# View GDSII
klayout designs/fp4_arithmetic/runs/RUN_*/results/final/gds/fp4_top.gds
```

---

## Project Significance

This project demonstrates a complete **student-to-silicon** journey:

```
Concept → RTL → FPGA Prototype → ASIC Physical Design → GDSII
```

| Phase | Achievement |
|-------|-------------|
| RTL Design | FP4 E2M1 arithmetic in Verilog HDL |
| FPGA Prototype | 200 MHz on Artix-7 (WNS +1.898ns) |
| ASIC Synthesis | SKY130 standard cells via Yosys |
| Physical Design | Full OpenLane RTL-to-GDSII flow |
| Sign-off | DRC clean, LVS verified, 200 MHz |
| Output | Fabrication-ready GDSII layout |

---

## Main FP4 Repository

The full standalone FP4 project with all source files,
reports, and documentation is available at:

```
https://github.com/Asadkhan282/FP4-Arithmetic-Unit
```

---

## References

- [OpenLane](https://github.com/The-OpenROAD-Project/OpenLane)
- [SKY130 PDK](https://github.com/google/skywater-pdk)
- [Caravel Harness](https://github.com/efabless/caravel)
- [NVIDIA Blackwell FP4](https://developer.nvidia.com/blog/nvidia-blackwell-platform)
- [FP4 in AI Inference](https://arxiv.org/abs/2209.05433)

---

## License

This project is licensed under the
[Apache License 2.0](https://opensource.org/licenses/Apache-2.0)
to comply with the Caravel User Project template requirements.
<img width="1918" height="1023" alt="Fp4_gdsii" src="https://github.com/user-attachments/assets/18cd5f6b-0ae5-4c80-981c-8d294a119092" />


---

*Designed by Asad Ali — QUEST, Nawabshah, Pakistan — 2026*
