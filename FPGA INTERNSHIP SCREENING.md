# SKY130 RTL Design & Synthesis Workshop

**Hands-On Lab: RTL Simulation, Logic Synthesis & Standard Cell Mapping**  
**Using Icarus Verilog · GTKWave · Yosys · SkyWater 130nm PDK**

![Assessment](https://img.shields.io/badge/Workshop-SKY130_RTL_Design_&_Synthesis-critical?style=flat-square)
![Tools](https://img.shields.io/badge/Tools-Icarus_Verilog_%7C_GTKWave_%7C_Yosys-2471A3?style=flat-square)
![PDK](https://img.shields.io/badge/PDK-SkyWater_130nm-F39C12?style=flat-square)
![Status](https://img.shields.io/badge/Status-Completed-27AE60?style=flat-square)

---

## Table of Contents

1. [Project Overview](#project-overview)
2. [Tools Used](#tools-used)
3. [Design Flow](#design-flow)
4. [Lab 1 — 2:1 Multiplexer (good\_mux)](#lab-1--21-multiplexer-good_mux)
5. [Lab 2 — Hierarchical Design: Multiple Modules](#lab-2--hierarchical-design-multiple-modules)
6. [Lab 3 — Submodule Synthesis (AND Gate)](#lab-3--submodule-synthesis-and-gate)
7. [Lab 4 — D Flip-Flop with Asynchronous Reset](#lab-4--d-flip-flop-with-asynchronous-reset)
8. [Lab 5 — D Flip-Flop with Asynchronous Set](#lab-5--d-flip-flop-with-asynchronous-set)
9. [Lab 6 — D Flip-Flop with Synchronous Reset](#lab-6--d-flip-flop-with-synchronous-reset)

10. [Lab 7 — Optimization: Multiply by 2 (mul2)](#lab-7--optimization-multiply-by-2-mul2)
11. [Lab 8 — Optimization: Multiply by 9 (mult8)](#lab-8--optimization-multiply-by-9-mult8)
12. [Key Learnings & Observations](#key-learnings--observations)
13. [Conclusion](#conclusion)

---

## Project Overview

This repository documents the complete output of a hands-on RTL Design and Synthesis lab conducted as part of the VSD (VLSI System Design) SKY130 RTL Design & Synthesis Workshop. The lab covers the full ASIC front-end design flow:

- **RTL Coding** — Writing synthesizable Verilog HDL to describe hardware behaviour.
- **Functional Simulation** — Compiling designs and testbenches using Icarus Verilog to generate `.vcd` waveform files.
- **Waveform Analysis** — Viewing `.vcd` files in GTKWave for visual validation of signal transitions.
- **Logic Synthesis** — Using Yosys with the SkyWater SKY130 standard cell library to generate gate-level netlists.

Design modules were intentionally chosen to cover combinational logic, sequential circuits, hierarchical design patterns, and synthesis optimizations.

---

## Tools Used

| Tool | Purpose |
|------|---------|
| **Icarus Verilog (iverilog)** | RTL Simulation |
| **GTKWave** | Waveform Visualization |
| **Yosys** | Logic Synthesis |
| **SKY130 PDK** | Standard Cell Technology Mapping |

---

## Design Flow

```
RTL Design (Verilog)
        ↓
Simulation (Icarus Verilog + GTKWave)
        ↓
Synthesis (Yosys)
        ↓
Technology Mapping (Sky130 Library)
        ↓
Gate-Level Netlist + Schematic Visualization
```

**One-Line Flow:**
```
design.v + tb_design.v  →  iverilog  →  a.out  →  vvp  →  design.vcd  →  GTKWave
design.v                →  Yosys + sky130.lib  →  gate_level_netlist.v  +  schematic
```

> **Key Insight:** RTL Simulation proves the RTL *works*. RTL Synthesis proves it *builds*. A design can simulate correctly but fail synthesis if it contains non-synthesizable constructs (e.g., `#delay`, `initial` blocks in the DUT).

---

## Lab 1 — 2:1 Multiplexer (`good_mux`)

### Description

A 2:1 multiplexer is designed and synthesized. Output `y` selects between `i0` and `i1` based on the `sel` signal.

### RTL Code

```verilog
module good_mux (input i0, input i1, input sel, output reg y);
  always @ (*)
  begin
    if (sel)
      y <= i1;
    else
      y <= i0;
  end
endmodule
```

- `always @ (*)` uses a wildcard sensitivity list — correct for combinational logic. Writing `always @(sel)` would miss changes on `i0` and `i1`, causing a simulation/synthesis mismatch.
- Despite `y` being declared as `reg` (required for `always` block assignment), no flip-flop is inferred because of `@(*)`. Yosys maps this directly to a MUX cell.

### Simulation Steps

```bash
# Step 1: Compile
iverilog good_mux.v tb_good_mux.v

# Step 2: Run simulation
./a.out

# Step 3: View waveform
gtkwave tb_good_mux.vcd
```

### Synthesis Steps

```bash
# Step 4: Start Yosys
yosys

# Step 5: Read files
read_liberty -lib sky130_fd_sc_hd__tt_025C_1v80.lib
read_verilog good_mux.v

# Step 6: Synthesize
synth -top good_mux

# Step 7: Technology mapping
abc -liberty sky130_fd_sc_hd__tt_025C_1v80.lib

# Step 8: Show schematic
show
```

### Results

**GTKWave Waveform:**

<img width="717" height="231" alt="586878629-b2699a68-230c-4ef1-a810-2a515d114e5c" src="https://github.com/user-attachments/assets/a27bbf45-5245-43b7-b362-ebd15be8a5cf" />




**Yosys Schematic:**

<img width="671" height="211" alt="586878671-ab0b2407-939e-4903-b173-f25e9132764e" src="https://github.com/user-attachments/assets/80ee4eb7-3199-4b42-aaf1-e1e22a4ae940" />




**Synthesis Outcome:** Yosys infers a single `sky130_fd_sc_hd__mux2_1` cell — `Number of cells: 1`. The `if-else` block is correctly mapped as a MUX, not a latch.

---

## Lab 2 — Hierarchical Design: Multiple Modules

### Description

A design consisting of two submodules (`sub_module1` = AND gate, `sub_module2` = OR gate) connected hierarchically. Demonstrates how Yosys handles module boundaries during synthesis.

### RTL Code

```verilog
module sub_module2 (input a, input b, output y);
  assign y = a | b;   // OR gate
endmodule

module sub_module1 (input a, input b, output y);
  assign y = a & b;   // AND gate
endmodule

module multiple_modules (input a, input b, input c, output y);
  wire net1;
  sub_module1 u1 (.a(a), .b(b), .y(net1));   // net1 = a & b
  sub_module2 u2 (.a(net1), .b(c), .y(y));   // y = (a & b) | c
endmodule
```

### Synthesis Steps

```bash
yosys
read_liberty -lib sky130_fd_sc_hd__tt_025C_1v80.lib
read_verilog multiple_modules.v
synth -top multiple_modules
abc -liberty sky130_fd_sc_hd__tt_025C_1v80.lib
show
```

### Result

<img width="858" height="116" alt="586878731-3ac3ef37-aca9-4154-a682-e8864141e46e" src="https://github.com/user-attachments/assets/139a1492-99c1-46bd-8961-db23c523535f" />



**Synthesis Outcome:** Yosys synthesizes each submodule separately and preserves the module hierarchy in the netlist. Two cells are generated — one AND (`u1`) and one OR (`u2`).

> **Hierarchical vs. Flat:** In hierarchical synthesis, module boundaries are preserved and each sub-module is synthesized as a separate entity. In flattened synthesis (using the `flatten` command), all boundaries are dissolved, enabling cross-module optimization — important for large designs.

---

## Lab 3 — Submodule Synthesis (AND Gate)

### Description

A simple AND gate submodule synthesized independently using `synth -top sub_module1`. Demonstrates targeted submodule synthesis, useful for IP development and design partitioning.

### Synthesis Steps

```bash
yosys
read_liberty -lib sky130_fd_sc_hd__tt_025C_1v80.lib
read_verilog sub_module.v
synth -top sub_module1
abc -liberty sky130_fd_sc_hd__tt_025C_1v80.lib
show
```

### Result

<img width="763" height="268" alt="image" src="https://github.com/user-attachments/assets/166c2b54-aba6-4184-b423-69ffd2824023" />



**Synthesis Outcome:** Only `sub_module1` (the AND gate) is synthesized. The diagram shows inputs `a` and `b` feeding a single `sky130_fd_sc_hd__and2_1` cell, with output `y`.

---

## Lab 4 — D Flip-Flop with Asynchronous Reset

### Description

A D Flip-Flop that resets immediately when `async_reset` is asserted — independent of the clock edge.

### RTL Code

```verilog
module dff_asyncres (input clk, input async_reset, input d, output reg q);
  always @ (posedge clk, posedge async_reset)
  begin
    if (async_reset)
      q <= 1'b0;
    else
      q <= d;
  end
endmodule
```

`async_reset` is listed in the sensitivity list, so the always block triggers on either the clock edge or the reset edge. When reset is asserted, `q` drops to 0 immediately — no clock needed.

### Simulation Steps

```bash
iverilog dff_asyncres.v tb_dff_asyncres.v
./a.out
gtkwave tb_dff_asyncres.vcd
```

### Synthesis Steps

```bash
yosys
read_liberty -lib sky130_fd_sc_hd__tt_025C_1v80.lib
read_verilog dff_asyncres.v
synth -top dff_asyncres
dfflibmap -liberty sky130_fd_sc_hd__tt_025C_1v80.lib
abc -liberty sky130_fd_sc_hd__tt_025C_1v80.lib
show
```

### Results

**GTKWave Waveform:**

<img width="1237" height="598" alt="image" src="https://github.com/user-attachments/assets/6781900e-d4ae-458b-8806-5aed7f23ec0b" />




**Yosys Schematic:**

<img width="1152" height="179" alt="image" src="https://github.com/user-attachments/assets/4ac1a53b-c4b7-4fcf-b6d1-44587f56b115" />



**Synthesis Outcome:** Yosys maps to `sky130_fd_sc_hd__dfrtp_1` — a DFF with a dedicated asynchronous active-high reset pin. No extra combinational logic is added; the reset is handled entirely within the cell.

---

## Lab 5 — D Flip-Flop with Asynchronous Set

### Description

A D Flip-Flop that forces output `q` HIGH immediately when `async_set` is asserted, independent of the clock.

### RTL Code

```verilog
module dff_async_set (input clk, input async_set, input d, output reg q);
  always @ (posedge clk, posedge async_set)
  begin
    if (async_set)
      q <= 1'b1;
    else
      q <= d;
  end
endmodule
```

### Synthesis Steps

```bash
yosys
read_liberty -lib sky130_fd_sc_hd__tt_025C_1v80.lib
read_verilog dff_asyncset.v
synth -top dff_async_set
dfflibmap -liberty sky130_fd_sc_hd__tt_025C_1v80.lib
abc -liberty sky130_fd_sc_hd__tt_025C_1v80.lib
show
```

### Results

**GTKWave Waveform:**

<img width="1034" height="186" alt="image" src="https://github.com/user-attachments/assets/7116ded1-a518-442d-9190-eaa0f55b0550" />



**Yosys Schematic:**

<img width="1113" height="198" alt="image" src="https://github.com/user-attachments/assets/7add3d56-cc76-4ac7-acdb-9e712c184972" />




**Synthesis Outcome:** Yosys maps to `sky130_fd_sc_hd__dfstp_1` — a DFF with a dedicated asynchronous active-high set pin. When `async_set` goes high, `q` immediately rises to 1, clock-independent.

---

## Lab 6 — D Flip-Flop with Synchronous Reset

### Description

A D Flip-Flop where the reset is applied only on the rising clock edge — not immediately.

### RTL Code

```verilog
module dff_syncres (input clk, input sync_reset, input d, output reg q);
  always @ (posedge clk)   // Only clock in sensitivity list
  begin
    if (sync_reset)
      q <= 1'b0;
    else
      q <= d;
  end
endmodule
```

`sync_reset` is checked inside the always block but is NOT in the sensitivity list. The block only evaluates at `posedge clk`, so even if `sync_reset` asserts mid-cycle, `q` only changes at the next clock edge.

### Synthesis Steps

```bash
yosys
read_liberty -lib sky130_fd_sc_hd__tt_025C_1v80.lib
read_verilog dff_syncres.v
synth -top dff_syncres
dfflibmap -liberty sky130_fd_sc_hd__tt_025C_1v80.lib
abc -liberty sky130_fd_sc_hd__tt_025C_1v80.lib
show
```

### Results

**GTKWave Waveform:**

<img width="1251" height="318" alt="image" src="https://github.com/user-attachments/assets/ecbc0582-20df-497e-a81d-85967deb8273" />



**Yosys Schematic:**

<img width="1049" height="205" alt="image" src="https://github.com/user-attachments/assets/ea21e273-ba82-41f8-b79e-f8d41e495539" />



**Synthesis Outcome:** Yosys uses a plain `sky130_fd_sc_hd__dfxtp_1` (no reset pin) and adds a MUX on the D-input to implement the synchronous reset in combinational logic: `D_input = sync_reset ? 0 : d`. This uses more area than the asynchronous variant but is easier to time.

---



## Lab 7 — Optimization: Multiply by 2 (`mul2`)

### Description

Multiplication by 2 is synthesized with zero logic gates — only wire connections.

### RTL Code

```verilog
module mul2 (input [2:0] a, output [3:0] y);
  assign y = a * 2;
endmodule
```

**Why zero gates?** Multiplying by 2 in binary is a left shift:

```
a * 2  =  {a[2], a[1], a[0], 1'b0}
```

The output is simply `a` with `0` appended at the LSB — no arithmetic hardware required.

### Synthesis Steps

```bash
yosys
read_verilog mul2.v
synth -top mul2
show
```

### Result

<img width="737" height="215" alt="image" src="https://github.com/user-attachments/assets/3180ec86-7740-4fff-b79f-ef52e46848ce" />



**Synthesis Outcome:** `Number of cells: 0`. Yosys implements `mul2` with direct wire routing — `a[2:0]` connects to `y[3:1]` and `y[0]` is tied to `1'b0`.

---

## Lab 8 — Optimization: Multiply by 9 (`mult8`)

### Description

Multiplication by 9 is synthesized with zero logic gates using bit concatenation.

### RTL Code

```verilog
module mult8 (input [2:0] a, output [5:0] y);
  assign y = a * 9;
endmodule
```

**Why zero gates?** The mathematical trick:

```
a × 9  =  a × (8 + 1)  =  (a << 3) + a  =  {a, 3'b000} + {3'b000, a}

Since a is 3-bit:
  {a[2], a[1], a[0], 0, 0, 0}
+            {0, 0, 0, a[2], a[1], a[0]}
= {a[2], a[1], a[0], a[2], a[1], a[0]}
= {a, a}   ← Just concatenate a with itself!
```

### Synthesis Steps

```bash
yosys
read_verilog mult8.v
synth -top mult8
show
```

### Result

<img width="739" height="178" alt="image" src="https://github.com/user-attachments/assets/eb072e13-3374-4bdb-80e0-53105fc2fddb" />



**Synthesis Outcome:** `Number of cells: 0`. Input `a[2:0]` is directly wired to both `y[5:3]` and `y[2:0]` — the 3-bit input is repeated twice to form the 6-bit output.

---

## Key Learnings & Observations

**1. Sensitivity List Correctness**  
Using `always @(*)` for combinational logic ensures all inputs are in the sensitivity list, preventing simulation/synthesis mismatches. Incomplete sensitivity lists (e.g., `always @(sel)`) cause latches.

**2. Asynchronous vs. Synchronous Resets**  
Asynchronous resets use dedicated cell pins (`dfrtp_1`, `dfstp_1`) and trigger independently of the clock. Synchronous resets are implemented with an additional MUX at the D-input of a plain DFF, consuming more area but simplifying Static Timing Analysis.

**3. Hierarchical vs. Flattened Synthesis**  
Hierarchical synthesis preserves module boundaries for incremental re-synthesis, IP protection, and team-based flows. Flattened synthesis (`flatten` command) dissolves all boundaries, enabling cross-module constant propagation and optimization — essential for large designs where significant area reduction is possible.

**4. Zero-Gate Arithmetic**  
Yosys performs algebraic reduction before technology mapping. Multiplications by powers of 2 (shift operations) and specific constants like 9 (`{a, a}`) are recognized and implemented with zero cells — purely through wire routing.

**5. MUX Inference from `if-else`**  
An `always @(*) if-else` structure is correctly inferred as a 2:1 MUX and mapped to `sky130_fd_sc_hd__mux2_1` directly, not decomposed into AND/OR gates.

**6. Importance of `abc` in Yosys**  
The `abc` engine applies Boolean optimization (BDD/SAT techniques) before technology mapping. Skipping it results in significantly higher gate counts. It is critical for industrial-quality synthesis.

---

## Synthesis vs. Simulation — Comparison

| Dimension | Simulation (iverilog + GTKWave) | Synthesis (Yosys + SKY130) |
|-----------|--------------------------------|----------------------------|
| **Purpose** | Functional verification | Physical implementation |
| **Input** | Design `.v` + Testbench `.v` | Design `.v` + Liberty `.lib` |
| **Output** | `.vcd` waveform file | Gate-level netlist `.v` |
| **Timing Model** | Zero-delay / delta-cycle | Real cell delays from PDK |
| **Non-synthesizable Constructs** | Fully supported | Errors or ignored |
| **Optimization** | None — RTL executes as written | Aggressive (constant folding, algebraic reduction) |
| **Flip-Flop Inference** | Behavioural register | Mapped to specific library cell |
| **Latch Warning** | Not warned | Yosys warns: "Inferred latch" |

> **Golden Rule:** Just because something simulates doesn't mean it synthesizes. Always simulate and synthesize — a discrepancy means either a non-synthesizable construct exists in the RTL, or there is a sensitivity list problem.

---

## Conclusion

This workshop covered the complete RTL-to-gates pipeline including Verilog HDL coding, functional simulation and waveform observation, and technology-mapped synthesis using an open-source PDK. Designs ranged from a 2:1 multiplexer to hierarchical AND/OR logic, three variants of D flip-flops, and zero-gate arithmetic — each chosen to demonstrate key synthesis behaviors: MUX inference, hierarchical vs. flat netlist strategies, asynchronous vs. synchronous sequential circuits, and arithmetic constant folding. The SKY130 standard cell library grounded every result in real silicon transistors, making this workshop directly relevant to practical chip design flows.

**Future Scope**
- Static Timing Analysis (STA)
- FPGA implementation and place-and-route
- Power optimization and estimation

---

*Workshop completed by: **Keshav***  
*Tools: Icarus Verilog · GTKWave · Yosys · SkyWater SKY130 PDK · Ubuntu*

