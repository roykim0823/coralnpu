# 01. Hello world — run a binary on the Verilator sim

> Part of the [CoralNPU tutorial series](../README.md) · *Onramp.*

**Goal.** Take a tiny C++ program (`examples/hello_world_add_floats.cc`),
compile it for the CoralNPU, run it on the Verilator RTL simulator, and observe
the execution two ways: as an instruction trace, and as an FST waveform.

**Prereqs.** [`00_setup`](../00_setup/README.md).

**Time.** ~5 minutes after Verilator is cached.

## What you'll learn
1. How to compile a CoralNPU program with the `coralnpu_v2_binary` Bazel rule.
2. How to drive that ELF through the Verilator simulator (`core_mini_axi_sim`).
3. What the `--instr_trace` and `--trace` flags do.
4. Why a "silent" program looks like it did nothing — and how to confirm it did.

## The program

Open [`examples/hello_world_add_floats.cc`](../../../examples/hello_world_add_floats.cc):

```cpp
#include <string.h>

float input1[8] __attribute__((section(".data")));
float input2[8] __attribute__((section(".data")));
float output[8] __attribute__((section(".data")));

int main() {
  for (int i = 0; i < 8; i++) {
    output[i] = input1[i] + input2[i];
  }
  return 0;
}
```

Three observations:
- Buffers live in `.data` so they get placed by the linker in **DTCM** (the
  32 KB tightly-coupled data SRAM).
- The program does **no I/O**. It writes into `output[]` and returns. The core
  halts when `main` returns. To *see* anything happen we need the simulator's
  trace facilities.
- There's no `printf`. That's deliberate — `printf` of floats blows the 8 KB
  ITCM by ~5 KB. See [`doc/tutorials/software/14_tflite_inference`](../software/14_tflite_inference/README.md)
  for how to use HTIF semihosting if you need stdout.

## Steps

### 1. Compile the program

```bash
bazel build //examples:coralnpu_v2_hello_world_add_floats
```

Produces an ELF under `bazel-bin/examples/coralnpu_v2_hello_world_add_floats.elf`.

### 2. Build the (non-RVV) simulator

```bash
bazel build //tests/verilator_sim:core_mini_axi_sim
```

This Verilator-compiles `VCoreMiniAxi` (the base CoreMiniAxi DUT, no vector
extension — fastest to build). The first build is slow; subsequent ones are
cached.

### 3. Run the ELF on the simulator

The Bazel output dir for the ELF embeds a platform-config hash. Get it from
the `bazel build` output of step 1 — it'll look like
`bazel-out/k8-fastbuild-ST-<hash>/bin/examples/coralnpu_v2_hello_world_add_floats.elf`.

```bash
ELF=$(find bazel-out -name coralnpu_v2_hello_world_add_floats.elf -print -quit)

bazel-bin/tests/verilator_sim/core_mini_axi_sim --binary "$ELF"
```

Expected:
```
... SystemC banner ...
Simulation stopped by user.
```
That's it — silent. `Simulation stopped by user.` means the testbench called
`sc_stop()` after detecting the core halted normally. It is **not an error**;
it just means the program ran and there was nothing else to print.

### 4. See what executed — instruction trace

```bash
bazel-bin/tests/verilator_sim/core_mini_axi_sim --binary "$ELF" --instr_trace
```

Expected: a stream of retired instructions with PC, mnemonic, and register
writes. You'll see eight `flw` / `fadd.s` / `fsw` patterns for the loop body,
flanked by the CRT entry/exit code.

### 5. Get a waveform

```bash
bazel-bin/tests/verilator_sim/core_mini_axi_sim --binary "$ELF" --trace
```

This writes an FST file (open in [GTKWave](https://gtkwave.sourceforge.net/)
or [Surfer](https://surfer-project.org/)) capturing every signal in the DUT.
Useful when the trace stops giving you enough — e.g. you want to inspect a
specific bus transaction or a fault condition.

## Source references
- [`examples/hello_world_add_floats.cc`](../../../examples/hello_world_add_floats.cc) — the program.
- [`examples/BUILD.bazel`](../../../examples/BUILD.bazel) — the `coralnpu_v2_binary` target definition.
- [`tests/verilator_sim/coralnpu/core_mini_axi_sim.cc`](../../../tests/verilator_sim/coralnpu/core_mini_axi_sim.cc) — the simulator driver.
- [`tests/verilator_sim/coralnpu/core_mini_axi_tb.cc`](../../../tests/verilator_sim/coralnpu/core_mini_axi_tb.cc) — testbench (AXI master/slave, memory, trace).
- [`README.md`](../../../README.md) — the project's official quick-start, which this tutorial expands on.

## Common pitfalls

- **"Did nothing happen?"** Without a trace flag you'll only see
  `Simulation stopped by user.`. The program *did* run; there was just no stdout.
  Use `--instr_trace` (cheap) or `--trace` (FST waveform) to verify.

- **`find` returned no ELF.** You skipped step 1. Re-run
  `bazel build //examples:coralnpu_v2_hello_world_add_floats`.

- **Wrong simulator for RVV programs.** `core_mini_axi_sim` is the *non*-RVV
  variant. If you swap in an RVV intrinsics program and re-run it here, you'll
  trap on the first vector instruction. For RVV use NPUSim (see
  [`12_rvv_intrinsics`](../software/12_rvv_intrinsics/README.md)) or the
  cocotb-driven `rvv_core_mini_axi_sim_cocotb` (covered in the
  verification path).

## Next
- [`02_simulator_landscape`](../02_simulator_landscape/README.md) — when to
  use which simulator and why there are four of them.
- [`software/10_program_anatomy`](../software/10_program_anatomy/README.md) —
  write your own program and drive it from a cocotb testbench.
