# 1.5 Simulation using Verilator

## Basic Concepts

### 1. What is Verilator?

* An open-source **SystemVerilog and Verilog simulator and linting tool**
* Compiles RTL code into a **native executable**
* Designed for fast RTL simulation

---

### 2. Compilation and Simulation

Verilator can compile the RTL and testbench into an executable simulation program.

```bash
verilator --binary --timed --top-module tb and.sv tb.sv
```

* `--binary` → build an executable simulation
* `--timed` → enable timing controls such as `#10`
* `--top-module tb` → specify the testbench top module

The executable is typically generated as:

```bash
obj_dir/Vtb
```

Run it with:

```bash
./obj_dir/Vtb
```

---

### 3. Output File Generation

Verilator can generate waveform files such as `.vcd` for analysis.

In the testbench:

```systemverilog
$dumpfile("wave.vcd");
$dumpvars(0, tb);
```

Compile with waveform tracing enabled:

```bash
verilator --binary --timed --trace --top-module tb and.sv tb.sv
```

Then run:

```bash
./obj_dir/Vtb
```

This generates:

```text
wave.vcd
```

---

### 4. Viewing Results

Output values can be printed in the terminal or viewed using a waveform viewer.

For terminal output:

```systemverilog
$monitor("a=%b b=%b y=%b", a, b, y);
```

For waveform analysis:

```bash
gtkwave wave.vcd
```

---

### 5. Multiple File Compilation

Compile multiple Verilog/SystemVerilog files together:

```bash
verilator --binary --timed --top-module tb design.sv tb.sv
```

For a larger design:

```bash
verilator --binary --timed --top-module tb \
    alu.sv control.sv register.sv tb.sv
```

---

### 6. Specify Build Directory

Verilator stores generated build files in a directory such as `obj_dir`.

You can specify another directory:

```bash
verilator --binary --timed --Mdir build --top-module tb and.sv tb.sv
```

The generated executable will be located in the specified build directory.

---

### 7. Debugging Errors

Verilator reports syntax, elaboration, and lint errors during compilation.

For example:

```bash
verilator --lint-only -Wall and.sv
```

* `--lint-only` → check the design without building a simulation executable
* `-Wall` → enable additional warnings

This is useful for checking RTL before running the complete simulation.

---

### 8. Simulation Time Control

Simulation delays such as `#10` require timing support.

Example:

```systemverilog
#10 a = 1;
#10 b = 1;
```

Compile with:

```bash
verilator --binary --timed --top-module tb and.sv tb.sv
```

---

### 9. Complete Flow

```bash
verilator --binary --timed --trace --top-module tb and.sv tb.sv
./obj_dir/Vtb
gtkwave wave.vcd
```

The complete flow is:

```text
SystemVerilog RTL + Testbench
            ↓
         Verilator
            ↓
      Native executable
            ↓
         Simulation
            ↓
      wave.vcd / output
            ↓
     GTKWave / Terminal
```
