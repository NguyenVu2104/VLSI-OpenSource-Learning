# Chapter 2: Synthesis (Yosys)

> Commands in this chapter were verified with Yosys 0.33 and the Sky130 HD library `sky130_fd_sc_hd__tt_025C_1v80.lib`. Re-run them on your own Yosys version and record the version in each lab README.

* 2.1 RTL to gate-level conversion
* 2.2 Technology mapping using `.lib` files
* 2.3 Gate-level netlist generation
* 2.4 Area estimation
* 2.5 Logic optimization techniques
* 2.6 Cell usage and gate count analysis

Synthesis converts RTL written in SystemVerilog into a gate-level netlist built from the standard cells of a technology library. The result is optimized for area (and, with constraints, timing). Yosys performs Logic Synthesis; it plays the role that Cadence Genus plays in a commercial flow.

Set the library path once per shell:

```
export LIB=<path to>/sky130_fd_sc_hd__tt_025C_1v80.lib
```

---

## 2.1 RTL to Gate-Level Conversion

RTL describes behavior; gate-level describes the actual hardware as interconnected logic gates. Yosys converts RTL in stages: parse the source, build a generic netlist, optimize it, then map it to a target library.

Example RTL (`rtl/and_gate.sv`):

```systemverilog
module and_gate (
    input  logic a,
    input  logic b,
    output logic y
);
    assign y = a & b;
endmodule
```

Read the SystemVerilog source and run the generic synthesis script:

```
read_verilog -sv rtl/and_gate.sv
synth -top and_gate -flatten
```

* `read_verilog -sv` → enables the SystemVerilog parser mode (`logic`, `always_comb`, `always_ff`)
* `synth -top and_gate` → run the generic synthesis flow with `and_gate` as the top module
* `-flatten` → merge the module hierarchy into a single module

`synth` already runs these steps internally: `hierarchy` (resolve the design hierarchy), `proc` (convert `always` blocks to logic and Flip-flop cells), `opt` (clean-up and optimization), `fsm` (state machine extraction), `memory`, `techmap` (map to generic gates such as `$_AND_`), and `abc` (gate mapping). After `synth`, the netlist contains only generic gates, not library cells.

### SystemVerilog constructs supported by the native Yosys frontend

Verified with Yosys 0.33:

| Construct | Result |
|---|---|
| `logic`, `always_comb`, `always_ff`, `assign` | Supported |
| `typedef enum` declared inside the module, `unique case` | Supported |
| `package` imported in the module header (`module m import pkg::*; (...)`) | Syntax error |
| Latch inferred inside `always_comb` | Error: `Latch inferred for signal ... from always_comb process` |

The last row is useful: `always_comb` turns an unintended latch into a hard error. Assign every output on every path, or give the signal a default value at the top of the block.

For SystemVerilog beyond this subset, the yosys-slang plugin (`read_slang`) is an option; its use in this flow has not been verified here.

---

## 2.2 Technology Mapping (.lib)

Technology mapping converts generic gates into real standard cells defined in a `.lib` (LIB) file. The LIB file describes each cell's function, area, and timing.

```
dfflibmap -liberty $LIB
abc -liberty $LIB
```

* `dfflibmap -liberty` → maps Flip-flops to library Flip-flop cells. Needed only when the design has sequential logic.
* `abc -liberty` → maps combinational logic to library cells.

Run `dfflibmap` before `abc`. Without `-liberty`, `abc` produces generic gates, and the Netlist will not contain any Sky130 cell.

Expected warnings from `dfflibmap` (`unsupported expression ... sdf*/edf*/sedf*`): `dfflibmap` skips scan and enable Flip-flops. They are harmless for these labs.

Note: Yosys 0.33 `abc` has no `-dont_use` option, so cells that a physical flow would exclude (for example `lpflow_*`) can appear in the result. Check `yosys -h abc` on your version for a way to exclude cells.

---

## 2.3 Gate-Level Netlist

After mapping, the design is a Netlist of Sky130 cells. Remove unused wires and write it out:

```
opt_clean -purge
write_verilog -noattr netlist/and_gate_net.v
```

* `opt_clean -purge` → remove unused cells and wires, including internal names
* `write_verilog -noattr` → write the Netlist without `src` attributes, which keeps it readable

Verified result for `and_gate`:

```verilog
module and_gate(a, b, y);
  input a;
  wire a;
  input b;
  wire b;
  output y;
  wire y;
  sky130_fd_sc_hd__and2_0 _0_ (
    .A(b),
    .B(a),
    .X(y)
  );
endmodule
```

The pin order `.A(b), .B(a)` is chosen by ABC; it does not change the function. The output is a Verilog-syntax Netlist even though the source is SystemVerilog, which is what downstream tools (OpenSTA, OpenROAD) expect.

Run `check -assert` after `synth` and before mapping. After mapping, `check` does not know the output pins of library cells and can report false `no driver` warnings.

---

## 2.4 Area Estimation

Yosys estimates area as the sum of the cell areas listed in the LIB file. The `-liberty` option is required; without it, `stat` only counts cells.

```
tee -o reports/stat.txt stat -liberty $LIB
```

Verified result for `and_gate`:

```
   Number of cells:                  1
     sky130_fd_sc_hd__and2_0         1

   Chip area for module '\and_gate': 6.256000
```

This is a pre-layout estimate in LIB area units. It does not include routing or any physical-design effect.

---

## 2.5 Logic Optimization

Optimization removes redundant and constant logic. `synth` already runs `opt` several times; the passes below show the effect.

Example (`rtl/redundant.sv`):

```systemverilog
module redundant (
    input  logic a, b, c,
    output logic y, z
);
    assign y = (a & b) | (a & b) | (a & 1'b0);
    assign z = (a & b) | (a & c);
endmodule
```

Run:

```
read_verilog -sv rtl/redundant.sv
proc
tee -o reports/pre.txt stat
synth -top redundant -flatten
abc -liberty $LIB
opt_clean -purge
tee -o reports/post.txt stat -liberty $LIB
```

Verified result:

| Stage | Cells |
|---|---|
| Before optimization (after `proc`) | 6 (`$and` × 4, `$or` × 2) |
| After `synth` + `abc` | 2 (`and2_0` × 1, `o21a_1` × 1), area 13.7632 |

Yosys reduced `y` to `a & b` (the duplicate term and the constant-zero term are removed) and factored `z` into `a & (b | c)`.

Main passes involved:

* `opt_expr` → constant folding and simple expression simplification
* `opt_merge` → merge identical cells
* `opt_clean` → remove unused cells and wires
* `abc` → re-structure and map the logic

---

## 2.6 Cell Usage and Gate Count Analysis

Gate count and cell usage come from the `stat` report. Cell usage shows which library cells were used and how many times; combined with area, it shows what dominates the design.

Example with sequential logic (`rtl/alu_slice.sv`):

```systemverilog
module alu_slice (
    input  logic       clk,
    input  logic       rst_n,
    input  logic [3:0] a,
    input  logic [3:0] b,
    input  logic       sel,
    output logic [3:0] q
);
    logic [3:0] y;

    always_comb begin
        if (sel) y = a + b;
        else     y = a & b;
    end

    always_ff @(posedge clk or negedge rst_n) begin
        if (!rst_n) q <= '0;
        else        q <= y;
    end
endmodule
```

Full flow:

```
yosys -p "read_verilog -sv rtl/alu_slice.sv; \
          synth -top alu_slice -flatten; \
          check -assert; \
          dfflibmap -liberty $LIB; \
          abc -liberty $LIB; \
          opt_clean -purge; \
          tee -o reports/stat.txt stat -liberty $LIB; \
          write_verilog -noattr netlist/alu_slice_net.v" | tee reports/yosys.log
```

Verified result:

```
   Number of cells:                 23
     sky130_fd_sc_hd__a21oi_1        4
     sky130_fd_sc_hd__dfrtp_1        4
     sky130_fd_sc_hd__lpflow_inputiso1p_1      1
     sky130_fd_sc_hd__maj3_1         1
     sky130_fd_sc_hd__mux2i_1        1
     sky130_fd_sc_hd__nand2_1        3
     sky130_fd_sc_hd__nand3_1        1
     sky130_fd_sc_hd__o21ai_0        2
     sky130_fd_sc_hd__xnor2_1        4
     sky130_fd_sc_hd__xor2_1         2

   Chip area for module '\alu_slice': 225.216000
```

How to read it:

* 4 × `dfrtp_1` are the four Flip-flops with asynchronous reset, one per bit of `q`; they come from `always_ff` with `negedge rst_n`.
* The remaining 19 cells implement the adder, the AND, and the `sel` multiplexer.
* Total cell count = 23; total area = 225.216.

---

## Common errors

* Netlist contains `$_AND_` instead of `sky130_fd_sc_hd__*` → `abc` was run without `-liberty`.
* `stat` shows no `Chip area` line → `stat` was run without `-liberty`.
* `syntax error, unexpected TOK_ID` on `module m import pkg::*;` → package import in the module header is not supported by the native frontend (Yosys 0.33); declare the types inside the module.
* `Latch inferred for signal ... from always_comb` → a signal is not assigned on every path in `always_comb`.
* `Wire ... is used but has no driver` from `check` after mapping → run `check` before `dfflibmap`/`abc`.
* Warnings `unsupported expression ... in pin attribute` from `dfflibmap` → harmless; scan and enable Flip-flops are skipped.
