# 1.5 Simulation using Verilator

> Commands in this section were verified with Verilator 5.020.

## Basic Concepts

### 1. What is Verilator?

* An open-source tool that lints, compiles, and simulates SystemVerilog and Verilog
* Works as a compiler: translates RTL into C++ and builds a native executable
* Designed for fast RTL simulation of edge-sensitive (flop-based) designs
* Mostly a two-state simulator (`0`/`1`): `X` and `Z` are not modeled, so checks such as `=== 1'bx` are not reliable

### 2. Compilation and Simulation

Compile the RTL and testbench into an executable:

```
verilator --binary -j 0 --timing --timescale 1ns/1ps --top-module tb and_gate.sv tb.sv
```

* `--binary` → build an executable simulation
* `-j 0` → use all available CPU cores for the build
* `--timing` → support timing controls such as `#10`
* `--timescale 1ns/1ps` → set the time unit/precision for all files
* `--top-module tb` → specify the testbench top module

The executable is generated at:

```
obj_dir/Vtb
```

Run it from the current working directory:

```
./obj_dir/Vtb
```

Simulation delays such as `#10` require timing support:

```
#10 a = 1;
#10 b = 1;
```

**Timescale note:** if some files contain a `` `timescale `` directive and others do not, Verilator reports the `TIMESCALEMOD` error. Use `--timescale` on the command line and leave the directive out of the source files. Without any timescale, the default waveform unit is 1 ps, so `#10` means 10 ps.

### 3. Waveform Generation

Add the dump commands to the testbench:

```
$dumpfile("wave.vcd");
$dumpvars(0, tb);
```

Waveform tracing must be enabled at compile time. Without a trace option, Verilator prints `$dumpvar ignored, as Verilated without --trace` and creates no file.

**VCD format:**

```
verilator --binary -j 0 --timing --timescale 1ns/1ps --trace --top-module tb and_gate.sv tb.sv
./obj_dir/Vtb
```

**FST format** (smaller file, faster to load; use `$dumpfile("wave.fst")` in the testbench):

```
verilator --binary -j 0 --timing --timescale 1ns/1ps --trace-fst --top-module tb and_gate.sv tb.sv
./obj_dir/Vtb
```

The waveform file (`wave.vcd` or `wave.fst`) is written to the directory where the executable is run, not to `obj_dir`.

### 4. Viewing Results

Terminal output:

```
$monitor("t=%0t a=%b b=%b y=%b", $time, a, b, y);
```

* `%0t` prints time in the simulation precision (ps with the setting above)
* Observed on Verilator 5.020: `$monitor` printed each line twice per time step. If this occurs, use `$display` after each delay instead.

Waveform analysis:

```
gtkwave wave.vcd
gtkwave wave.fst
```

### 5. Multiple File Compilation

Compile multiple SystemVerilog files together:

```
verilator --binary -j 0 --timing --timescale 1ns/1ps --top-module tb design.sv tb.sv
```

For a larger design:

```
verilator --binary -j 0 --timing --timescale 1ns/1ps --top-module tb \
    alu.sv control.sv register.sv tb.sv
```

### 6. Specify Build Directory

Verilator stores generated build files in `obj_dir` by default. Use `--Mdir` to choose another directory:

```
verilator --binary -j 0 --timing --timescale 1ns/1ps --Mdir build --top-module tb and_gate.sv tb.sv
./build/Vtb
```

The executable is located in the specified directory.

### 7. Linting, Assertions, and Debugging

**Lint:**

```
verilator --lint-only -Wall and_gate.sv
```

* `--lint-only` → check the design without building an executable
* `-Wall` → enable additional warnings; warnings are treated as errors (non-zero exit code)
* With `-Wall`, the file name must match the module name (`and_gate.sv` for `module and_gate`), otherwise the `DECLFILENAME` warning is raised. It can be disabled with `-Wno-DECLFILENAME`.

**Assertions:** enable assertion checking with `--assert`:

```
verilator --binary -j 0 --timing --timescale 1ns/1ps --assert --top-module tb and_gate.sv tb.sv
```

Example check in the testbench:

```
a = 1; b = 1; #10
assert (y === 1'b1) else $error("11 fail");
```

A failed assertion prints the error with the source location and simulation time, then aborts the run with a non-zero exit code, which makes it usable in scripts and CI.

**Common errors:**

* `Invalid option: --timed` → the correct option is `--timing`
* `TIMESCALEMOD` → mixed `` `timescale `` usage across files; use `--timescale`
* `$dumpvar ignored, as Verilated without --trace` → add `--trace` or `--trace-fst`

### 8. Complete Flow

```
verilator --binary -j 0 --timing --timescale 1ns/1ps --assert --trace-fst --top-module tb and_gate.sv tb.sv
./obj_dir/Vtb
gtkwave wave.fst
```

```
SystemVerilog RTL + Testbench
            ↓
         Verilator
            ↓
      Native executable
            ↓
         Simulation
            ↓
     wave.fst / terminal output
            ↓
     GTKWave / Terminal
```
