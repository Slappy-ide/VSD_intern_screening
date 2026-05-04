# VSD FPGA Intern Screening — Lab Documentation

> *From Verilog to real silicon cells — a complete ASIC front-end flow using open-source tools.*

![Workshop](https://img.shields.io/badge/VSD-SKY130_RTL_%26_Synthesis-8B0000?style=for-the-badge)
![Simulator](https://img.shields.io/badge/Icarus_Verilog-Simulation-1565C0?style=for-the-badge)
![Synthesizer](https://img.shields.io/badge/Yosys-Synthesis-2E7D32?style=for-the-badge)
![PDK](https://img.shields.io/badge/SkyWater-SKY130_PDK-E65100?style=for-the-badge)

---

## What This Repository Is

This is my lab notebook for the VSD SKY130 RTL Design & Synthesis Workshop — a hands-on exploration of how Verilog descriptions of hardware become real gate-level netlists mapped to actual silicon standard cells. Every experiment here covers a different piece of that pipeline: writing RTL, simulating it, synthesizing it, and understanding what the tool actually did and *why*.

The goal isn't just to run commands — it's to build intuition about how synthesis tools think.

---

## Contents

- [Background Theory](#background-theory)
  - [What Is RTL?](#what-is-rtl)
  - [What Is Synthesis?](#what-is-synthesis)
  - [Simulation vs Synthesis — Why Both Matter](#simulation-vs-synthesis--why-both-matter)
  - [The SKY130 PDK](#the-sky130-pdk)
  - [Tool Chain Overview](#tool-chain-overview)
- [Lab Experiments](#lab-experiments)
  - [Lab 1 — 2:1 MUX](#lab-1--21-mux-good_mux)
  - [Lab 2 — Hierarchical Multi-Module Design](#lab-2--hierarchical-multi-module-design)
  - [Lab 3 — Submodule Synthesis](#lab-3--targeted-submodule-synthesis)
  - [Lab 4 — DFF: Asynchronous Reset](#lab-4--dff-with-asynchronous-reset)
  - [Lab 5 — DFF: Asynchronous Set](#lab-5--dff-with-asynchronous-set)
  - [Lab 6 — DFF: Synchronous Reset](#lab-6--dff-with-synchronous-reset)
  - [Lab 7 — Arithmetic Optimization: mul2](#lab-7--arithmetic-optimization-mul2)
  - [Lab 8 — Arithmetic Optimization: mult8](#lab-8--arithmetic-optimization-mult8)
- [Patterns & Insights](#patterns--insights)
- [Conclusion](#conclusion)

---

## Background Theory

Before diving into the labs, it helps to understand *what* we're actually doing and *why* each step matters. This section covers the foundational concepts behind RTL design and synthesis.

---

### What Is RTL?

**Register Transfer Level (RTL)** is an abstraction of a digital circuit that describes how data moves between registers and the logical operations performed on that data. It sits in the middle of the hardware design abstraction ladder:

```
High Abstraction
      │
   Algorithmic / Behavioral  (what the circuit does)
      │
   RTL  (how data moves between registers — what we write in Verilog)
      │
   Gate Level  (AND, OR, NOT, DFF cells)
      │
   Transistor Level  (how gates are physically built)
      │
Low Abstraction  (Layout — silicon polygons)
```

When you write Verilog at the RTL level, you're describing:
- **Data paths** — how values are computed and transformed
- **Control logic** — conditions that determine which computation happens
- **Registers** — where values are stored between clock cycles

RTL Verilog is *synthesizable* — meaning a tool (Yosys in our case) can automatically convert it to a gate-level netlist. Not all Verilog is synthesizable: constructs like `#delays`, `$display`, and `initial` blocks are for simulation only and will be ignored or error out during synthesis.

---

### What Is Synthesis?

**Logic Synthesis** is the automated process of converting RTL code into a gate-level netlist composed of cells from a specific technology library. It has three internal stages:

**1. Translation (RTL → Generic Logic)**
The tool parses your Verilog and converts it into a technology-independent Boolean representation — typically using AND/OR/NOT gates, multiplexers, and flip-flops as abstract primitives. At this stage, no real cells exist yet.

**2. Optimization (Logic Minimization)**
The generic logic is optimized using Boolean algebra and algebraic techniques. Goals include reducing gate count, minimizing critical path delay, and eliminating redundant logic. Yosys uses the `abc` engine (from UC Berkeley) for this, which applies BDD (Binary Decision Diagram) and SAT-based techniques.

**3. Technology Mapping (Generic → Library Cells)**
The optimized generic logic is mapped to real cells from the target PDK library. Each cell has a defined function, area, and timing. The `abc -liberty` command in Yosys performs this mapping using the SKY130 `.lib` file, which describes every cell's logical function, drive strength, and delay.

```
Verilog RTL
    │
    ▼  [read_verilog]
Generic Logic Network (RTLIL — Yosys internal representation)
    │
    ▼  [synth -top]
Optimized Generic Netlist
    │
    ▼  [abc -liberty sky130.lib]
Technology-Mapped Gate-Level Netlist (real SKY130 cells)
    │
    ▼  [show / write_verilog]
Schematic / Output Netlist
```

---

### Simulation vs Synthesis — Why Both Matter

These are two completely different processes serving different purposes. A critical trap in digital design is assuming that if simulation passes, synthesis will be correct too.

| | Simulation | Synthesis |
|---|---|---|
| **What it does** | Runs your Verilog as software, checks logic correctness | Converts Verilog to actual hardware cells |
| **Timing model** | Zero-delay or annotated — not real silicon timing | Real cell delays from the PDK `.lib` file |
| **Supports `#delay`?** | Yes | No — ignored or causes errors |
| **Supports `initial`?** | Yes (for testbenches) | No (for the DUT) |
| **Flip-flop inference** | Just a behavioral register variable | Must map to a specific cell like `sky130_fd_sc_hd__dfxtp_1` |
| **Latch detection** | Won't warn you | Yosys warns: *"Inferred latch"* |
| **Optimization** | None — RTL executes as written | Aggressive: constant folding, algebraic reduction, Boolean minimization |

**The golden rule:** A design that simulates correctly can still synthesize into wrong hardware. Always do both, and treat any discrepancy as a bug — it almost always points to a sensitivity list issue or a non-synthesizable construct leaking into the DUT.

---

### The SKY130 PDK

The **SkyWater SKY130** is an open-source 130nm CMOS Process Design Kit (PDK) — a complete description of how to turn circuit designs into real silicon chips using SkyWater Technology's 130nm fabrication process.

130nm refers to the minimum feature size of the transistors. While not cutting-edge (modern chips use 3–5nm), 130nm is ideal for learning because the entire library is open, documented, and has actually been used for real chip tape-outs through Google/efabless.

The file used throughout these labs — `sky130_fd_sc_hd__tt_025C_1v80.lib` — is the **high-density standard cell library** characterized at:
- **tt** = typical-typical process corner
- **025C** = 25°C temperature
- **1v80** = 1.8V supply voltage

Cell names follow a predictable pattern: `sky130_fd_sc_hd__and2_1` is a 2-input AND gate with drive strength 1. `sky130_fd_sc_hd__dfrtp_1` is a D flip-flop with asynchronous active-high reset. Learning to read these names makes schematics self-documenting.

---

### Tool Chain Overview

```
┌──────────────────────────────────────────────────┐
│                   Design Entry                    │
│            Write Verilog RTL (.v files)           │
└───────────────────────┬──────────────────────────┘
                        │
           ┌────────────┴────────────┐
           │                         │
    ┌──────▼──────┐           ┌──────▼──────┐
    │  Simulation  │           │  Synthesis   │
    │   iverilog   │           │    Yosys     │
    │   + vvp      │           │  + abc       │
    └──────┬──────┘           └──────┬──────┘
           │                         │
    ┌──────▼──────┐           ┌──────▼──────┐
    │   GTKWave   │           │  Gate-Level  │
    │  Waveforms  │           │   Netlist    │
    │  (.vcd)     │           │  Schematic   │
    └─────────────┘           └─────────────┘
```

| Tool | Role | Key Commands |
|------|------|-------------|
| **Icarus Verilog** | Compiles and simulates Verilog | `iverilog`, `vvp` |
| **GTKWave** | Visualizes `.vcd` waveforms | `gtkwave` |
| **Yosys** | Open-source logic synthesizer | `synth`, `abc`, `show` |
| **SKY130 `.lib`** | Technology library for cell mapping | Passed via `-liberty` flag |

---

## Lab Experiments

---

### Lab 1 — 2:1 MUX (`good_mux`)

#### Theory

A **2:1 Multiplexer** selects one of two data inputs (`i0`, `i1`) based on a select signal (`sel`). It is the most fundamental routing primitive in digital design — nearly every control structure in RTL synthesizes to some form of MUX at the gate level.

In Verilog, an `if-else` inside `always @(*)` is the natural way to describe a MUX:

```verilog
module good_mux (input i0, input i1, input sel, output reg y);
  always @(*) begin
    if (sel)
      y <= i1;
    else
      y <= i0;
  end
endmodule
```

Two things worth noting:

**Why `always @(*)`?** The wildcard sensitivity list means the block re-evaluates whenever *any* input changes. Writing `always @(sel)` instead would cause `y` not to update when `i0` or `i1` changes — a classic simulation/synthesis mismatch. Synthesis doesn't use sensitivity lists (it builds static hardware), but simulation does. `@(*)` closes this gap.

**Why `reg y` if it's not a register?** In Verilog, `reg` doesn't mean flip-flop — it means the variable holds its value between `always` block evaluations. Because this `always @(*)` block covers all conditions with no clock edge, Yosys correctly infers a combinational MUX, not a flip-flop.

#### Simulation

```bash
iverilog good_mux.v tb_good_mux.v
./a.out
gtkwave tb_good_mux.vcd
```

#### Synthesis

```bash
yosys
read_liberty -lib sky130_fd_sc_hd__tt_025C_1v80.lib
read_verilog good_mux.v
synth -top good_mux
abc -liberty sky130_fd_sc_hd__tt_025C_1v80.lib
show
```

#### Results

**GTKWave Waveform:**

<img width="717" height="231" alt="MUX waveform" src="https://github.com/user-attachments/assets/a27bbf45-5245-43b7-b362-ebd15be8a5cf" />

**Yosys Schematic:**

<img width="671" height="211" alt="MUX schematic" src="https://github.com/user-attachments/assets/80ee4eb7-3199-4b42-aaf1-e1e22a4ae940" />

**Outcome:** Yosys maps the design to a single `sky130_fd_sc_hd__mux2_1` cell. The `if-else` is inferred directly as a MUX — not decomposed into AND/OR gates. Cell count: 1.

---

### Lab 2 — Hierarchical Multi-Module Design

#### Theory

Real designs are composed of **submodules** connected together in a hierarchy. Yosys can synthesize hierarchically (each module synthesized separately, hierarchy preserved) or flat (all boundaries dissolved).

```verilog
module sub_module1 (input a, input b, output y);
  assign y = a & b;
endmodule

module sub_module2 (input a, input b, output y);
  assign y = a | b;
endmodule

module multiple_modules (input a, input b, input c, output y);
  wire net1;
  sub_module1 u1 (.a(a), .b(b), .y(net1));
  sub_module2 u2 (.a(net1), .b(c), .y(y));
endmodule
```

Overall function: `y = (a & b) | c`

**Hierarchical synthesis** (default): Yosys synthesizes each submodule separately and preserves them as distinct entities in the netlist. Useful for IP protection, incremental re-synthesis, and team-based flows.

**Flattened synthesis** (using `flatten`): Module boundaries are dissolved. The optimizer sees all logic together and can eliminate cross-boundary redundancy. Usually better area/timing, but loses structural clarity.

#### Synthesis

```bash
yosys
read_liberty -lib sky130_fd_sc_hd__tt_025C_1v80.lib
read_verilog multiple_modules.v
synth -top multiple_modules
abc -liberty sky130_fd_sc_hd__tt_025C_1v80.lib
show
```

#### Result

<img width="858" height="116" alt="Hierarchical schematic" src="https://github.com/user-attachments/assets/139a1492-99c1-46bd-8961-db23c523535f" />

**Outcome:** Two cells — one AND (`u1`) and one OR (`u2`). Module hierarchy preserved. Each submodule appears as its own box in the schematic.

---

### Lab 3 — Targeted Submodule Synthesis

#### Theory

Yosys allows synthesizing a specific module from a multi-module file using `synth -top <module_name>`. This is called **bottom-up synthesis** and is useful for:
- **IP development** — verify a block in isolation before integrating
- **Design partitioning** — team members verify their modules independently
- **Debugging** — isolate a block to understand its synthesis result

```bash
yosys
read_liberty -lib sky130_fd_sc_hd__tt_025C_1v80.lib
read_verilog sub_module.v
synth -top sub_module1
abc -liberty sky130_fd_sc_hd__tt_025C_1v80.lib
show
```

#### Result

<img width="763" height="268" alt="AND submodule schematic" src="https://github.com/user-attachments/assets/166c2b54-aba6-4184-b423-69ffd2824023" />

**Outcome:** Only `sub_module1` synthesized. Mapped to `sky130_fd_sc_hd__and2_1`. Inputs `a`, `b` → AND cell → output `y`.

---

### Lab 4 — DFF with Asynchronous Reset

#### Theory

A **D Flip-Flop (DFF)** is the fundamental storage element in synchronous digital design. It captures the value of `D` at the rising clock edge and holds it at `Q` until the next edge.

**Asynchronous reset** forces `Q` to 0 *immediately* when `reset` is asserted — no clock edge required.

```verilog
module dff_asyncres (input clk, input async_reset, input d, output reg q);
  always @(posedge clk, posedge async_reset) begin
    if (async_reset)
      q <= 1'b0;
    else
      q <= d;
  end
endmodule
```

`posedge async_reset` in the sensitivity list is what makes it asynchronous — the block triggers on either clock or reset edge. When reset goes high, `Q` drops to 0 without waiting for a clock.

In synthesis, Yosys maps this to `sky130_fd_sc_hd__dfrtp_1` — a DFF with a **dedicated asynchronous reset pin** built into the cell's transistor structure. No extra combinational gates are needed.

**Why use async reset?** It guarantees a known state on power-up or fault, even before the clock is running — critical for control logic and state machines.

> **Note:** The extra `dfflibmap` command below tells Yosys to map flip-flops from the liberty file *before* `abc` runs. Without it, flip-flops may remain in a generic form and not map to correct SKY130 cells.

#### Simulation

```bash
iverilog dff_asyncres.v tb_dff_asyncres.v
./a.out
gtkwave tb_dff_asyncres.vcd
```

#### Synthesis

```bash
yosys
read_liberty -lib sky130_fd_sc_hd__tt_025C_1v80.lib
read_verilog dff_asyncres.v
synth -top dff_asyncres
dfflibmap -liberty sky130_fd_sc_hd__tt_025C_1v80.lib
abc -liberty sky130_fd_sc_hd__tt_025C_1v80.lib
show
```

#### Results

**GTKWave Waveform:**

<img width="1249" height="368" alt="Async reset waveform" src="https://github.com/user-attachments/assets/df3f75df-f727-4b99-8b45-688556790bd6" />

**Yosys Schematic:**

<img width="1148" height="188" alt="Async reset schematic" src="https://github.com/user-attachments/assets/f06a2062-b5f0-4fe2-9fef-208547e3deba" />

**Outcome:** Single `sky130_fd_sc_hd__dfrtp_1` cell. Reset is built into the cell — no extra logic needed.

---

### Lab 5 — DFF with Asynchronous Set

#### Theory

**Asynchronous set** forces output `Q` HIGH immediately when `set` is asserted — same principle as async reset but in the opposite direction.

```verilog
module dff_async_set (input clk, input async_set, input d, output reg q);
  always @(posedge clk, posedge async_set) begin
    if (async_set)
      q <= 1'b1;
    else
      q <= d;
  end
endmodule
```

The only RTL difference from async reset is `q <= 1'b1` instead of `q <= 1'b0`. But Yosys maps to a *completely different cell* — `sky130_fd_sc_hd__dfstp_1` — because SET and RESET are structurally distinct at the transistor level. Each has its own optimized cell in the library; they are not interchangeable.

#### Synthesis

```bash
yosys
read_liberty -lib sky130_fd_sc_hd__tt_025C_1v80.lib
read_verilog dff_asyncset.v
synth -top dff_async_set
dfflibmap -liberty sky130_fd_sc_hd__tt_025C_1v80.lib
abc -liberty sky130_fd_sc_hd__tt_025C_1v80.lib
show
```

#### Results

**GTKWave Waveform:**

<img width="1034" height="186" alt="Async set waveform" src="https://github.com/user-attachments/assets/7116ded1-a518-442d-9190-eaa0f55b0550" />

**Yosys Schematic:**

<img width="1113" height="198" alt="Async set schematic" src="https://github.com/user-attachments/assets/7add3d56-cc76-4ac7-acdb-9e712c184972" />

**Outcome:** `sky130_fd_sc_hd__dfstp_1` cell. `Q` rises immediately on `async_set` assertion, clock-independent.

---

### Lab 6 — DFF with Synchronous Reset

#### Theory

**Synchronous reset** applies the reset *only at the clock edge* — mid-cycle assertions of reset have no immediate effect on `Q`.

```verilog
module dff_syncres (input clk, input sync_reset, input d, output reg q);
  always @(posedge clk) begin   // Only clock — no reset in sensitivity list
    if (sync_reset)
      q <= 1'b0;
    else
      q <= d;
  end
endmodule
```

`sync_reset` is not in the sensitivity list. The block only evaluates at `posedge clk`, so reset is sampled at the clock edge like any other signal.

**In synthesis**, no SKY130 standard cell has a synchronous reset pin. So Yosys implements it with a MUX on the D-input of a plain DFF:

```
D_effective = sync_reset ? 1'b0 : d
Q ← D_effective  (at posedge clk)
```

This produces `sky130_fd_sc_hd__dfxtp_1` (plain DFF) + `sky130_fd_sc_hd__mux2_1` — **more area** than async reset, but the reset path is purely combinational, making Static Timing Analysis simpler.

**Choosing between async and sync reset:**
- **Async reset** → guaranteed initialization before clock is running; harder to time
- **Sync reset** → simpler timing analysis; reset must arrive before the clock edge it's meant to affect

#### Synthesis

```bash
yosys
read_liberty -lib sky130_fd_sc_hd__tt_025C_1v80.lib
read_verilog dff_syncres.v
synth -top dff_syncres
dfflibmap -liberty sky130_fd_sc_hd__tt_025C_1v80.lib
abc -liberty sky130_fd_sc_hd__tt_025C_1v80.lib
show
```

#### Results

**GTKWave Waveform:**

<img width="1251" height="318" alt="Sync reset waveform" src="https://github.com/user-attachments/assets/ecbc0582-20df-497e-a81d-85967deb8273" />

**Yosys Schematic:**

<img width="1049" height="205" alt="Sync reset schematic" src="https://github.com/user-attachments/assets/ea21e273-ba82-41f8-b79e-f8d41e495539" />

**Outcome:** Plain `dfxtp_1` + `mux2_1`. MUX implements the synchronous reset in combinational logic before the DFF's D-input.

---

### Lab 7 — Arithmetic Optimization: `mul2`

#### Theory

This lab reveals one of the most satisfying aspects of synthesis: **the tool is smarter than you think**.

```verilog
module mul2 (input [2:0] a, output [3:0] y);
  assign y = a * 2;
endmodule
```

Multiplying a binary number by 2 is identical to a **left shift by 1 bit**:

```
a       =  a[2]  a[1]  a[0]
a × 2   =  a[2]  a[1]  a[0]  0

y[3] ← a[2]
y[2] ← a[1]
y[1] ← a[0]
y[0] ← 1'b0   (tied to GND)
```

The output bits are just the input bits shifted — this is purely **wiring**, not logic. Yosys's algebraic optimizer recognizes this identity before technology mapping even begins, and generates **zero logic cells**. The connections are handled directly in the netlist.

#### Synthesis

```bash
yosys
read_verilog mul2.v
synth -top mul2
show
```

#### Result

<img width="737" height="215" alt="mul2 schematic" src="https://github.com/user-attachments/assets/3180ec86-7740-4fff-b79f-ef52e46848ce" />

**Outcome:** `Number of cells: 0`. Pure wire routing — `y[3:1]` = `a[2:0]`, `y[0]` = GND.

---

### Lab 8 — Arithmetic Optimization: `mult8`

#### Theory

This extends the zero-gate concept to a less obvious constant:

```verilog
module mult8 (input [2:0] a, output [5:0] y);
  assign y = a * 9;
endmodule
```

The mathematical identity that makes this work:

```
a × 9  =  a × (8 + 1)
       =  (a × 8) + (a × 1)
       =  (a << 3) + a

For 3-bit input a, expanding the addition:
  {a[2], a[1], a[0], 0, 0, 0}
+           {0, 0, 0, a[2], a[1], a[0]}
= {a[2], a[1], a[0], a[2], a[1], a[0]}
= {a, a}   ← Concatenate a with itself
```

So `a * 9` for a 3-bit input is provably `{a, a}` — 6 wires from 3 inputs, no adder, no multiplier. Yosys's algebraic optimizer catches this automatically before any technology mapping.

#### Synthesis

```bash
yosys
read_verilog mult8.v
synth -top mult8
show
```

#### Result

<img width="739" height="178" alt="mult8 schematic" src="https://github.com/user-attachments/assets/eb072e13-3374-4bdb-80e0-53105fc2fddb" />

**Outcome:** `Number of cells: 0`. `y[5:3]` = `a[2:0]`, `y[2:0]` = `a[2:0]`. Input wired to both halves of the output.

---

## Patterns & Insights

After working through all eight labs, a few recurring themes are worth internalizing:

**Sensitivity list = simulation contract.** `always @(*)` means "re-evaluate whenever anything I read changes." A missing signal in the sensitivity list creates a latch in synthesis and a mismatch in simulation. The wildcard is almost always the right choice for combinational logic.

**Cell names are documentation.** Once you learn the SKY130 naming convention, a schematic is self-documenting. `dfrtp` = DFF with asynchronous active-high Reset. `dfstp` = DFF with asynchronous active-high Set. `dfxtp` = plain DFF, no preset/clear. `mux2` = 2:1 MUX. Reading the cell name tells you exactly what the tool inferred.

**Synchronous reset costs area; asynchronous reset costs timing.** Sync reset adds a MUX at the D-input. Async reset uses a dedicated cell pin — no extra gates, but the reset timing must be constrained carefully to avoid hold violations near the clock edge.

**Yosys algebraically reduces before mapping.** The `mul2` and `mult8` results demonstrate that Yosys doesn't blindly map `*` to a multiplier tree. It first simplifies expressions symbolically — only after exhausting algebraic reduction does it attempt technology mapping. This is why the `abc` engine matters so much.

**Hierarchical synthesis is the default; flattening is deliberate.** For most workflows, preserve hierarchy during synthesis and only flatten for final optimization passes, or when cross-boundary constant propagation is known to help.

---

## Conclusion

These eight experiments trace the full RTL-to-gates path — from writing Verilog that describes intent, to simulating that intent, to letting Yosys translate it into real SKY130 silicon cells. What stood out most is how much reasoning the synthesis tool does on its own: inferring MUX vs. latch vs. flip-flop from structural RTL patterns, choosing the correct DFF cell variant based on reset semantics, and eliminating entire blocks of arithmetic through algebraic reduction before a single real cell is touched.

Understanding *why* the tool made each decision — not just *what* the output looks like — is what separates writing Verilog from designing for synthesis.

---

*Tools: Icarus Verilog · GTKWave · Yosys · SkyWater SKY130 PDK · Ubuntu Linux*
