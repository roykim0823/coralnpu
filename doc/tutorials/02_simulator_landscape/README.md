# 02. Simulator landscape — when to use which one

> Part of the [CoralNPU tutorial series](../README.md) · *Onramp.*

**Goal.** Understand the four (really five) simulation paths this project
supports, so you can pick the right one for your task without burning a slow
recompile to find out.

**Prereqs.** [`00_setup`](../00_setup/README.md). Helps to have done
[`01_hello_world`](../01_hello_world/README.md).

**Time.** ~10 minutes to read; longer if you exercise each.

## TL;DR — pick one

| Your task | Use | Why |
|-----------|-----|-----|
| "I want to run an ML model fast" | **NPUSim** | Pure software ISS, no RTL. Seconds, not minutes. |
| "I want to test my program on the actual DUT" | **Verilator + cocotb** | The default. Open-source, fast enough, RVV-capable. |
| "I want a C++-driven RTL sim with FST waveform" | **Verilator (C++ driver)** | The `core_mini_axi_sim` binary from [`01_hello_world`](../01_hello_world/README.md). |
| "I need formal-grade verification with Spike cosim" | **UVM** | Constrained-random + 3-way check vs Spike ISS via MPACT. |
| "We have a VCS license and need gate-level / commercial flow" | **VCS + cocotb / SystemC** | Tag-gated, license-required. |

## The five paths

### A. NPUSim — pure-software instruction-set simulator

- **Where:** `sw/coralnpu_sim/` (C++ + pybind11), drives Python.
- **Speed:** ~seconds for a MobileNet inference. No RTL anywhere.
- **Use for:** ML workload bring-up, model conversion debugging, profiling
  cycle counts of compiled C/C++ at the instruction level.
- **Run:**
  ```bash
  bazel run //tests/npusim_examples:npusim_run_mobilenet
  ```
- **Don't use it to verify hardware** — it implements the ISA, not the
  pipeline. A bug in the RTL won't show up here.

See [`14_tflite_inference`](../software/14_tflite_inference/README.md) for the
plumbing.

### B. Verilator + cocotb — the default RTL test path

- **Where:** `tests/cocotb/` (Python tests) drive a Verilator-compiled model
  of the Chisel-generated SystemVerilog.
- **Speed:** moderate. Verilator compile is slow once; sim runs are quick.
- **Use for:** day-to-day verification of programs, RVV kernels, bus protocol
  tests, regressions. **This is what the project's CI runs.**
- **Run:**
  ```bash
  # Base core (no vector ext.)
  bazel run //tests/cocotb:core_mini_axi_sim_cocotb

  # RVV-capable core
  bazel run //tests/cocotb:rvv_core_mini_axi_sim_cocotb
  ```

### C. Verilator with a C++ driver

- **Where:** `tests/verilator_sim/` (C++) — same Verilator model as path B, but
  driven from C++ instead of cocotb.
- **Speed:** as B; slightly closer to bare metal.
- **Use for:** running an arbitrary ELF and dumping an instruction trace or
  FST waveform without writing a cocotb test. The pattern from
  [`01_hello_world`](../01_hello_world/README.md).
- **Run:**
  ```bash
  bazel build //tests/verilator_sim:core_mini_axi_sim
  bazel-bin/tests/verilator_sim/core_mini_axi_sim --binary <elf> [--instr_trace] [--trace]
  ```

### D. UVM — formal-grade verification with Spike cosim

- **Where:** `tests/uvm/` (UVM 1.2 SystemVerilog) — `Makefile`-driven, runs on
  VCS, integrates with MPACT-RiscV for instruction-by-instruction cosim
  against the Spike ISS.
- **Speed:** slow. Designed for thoroughness, not turnaround.
- **Use for:** sign-off-quality regressions, finding bugs that random tests
  uncover and directed tests miss, formal correctness checks.
- **Run (requires VCS license + `$CORALNPU_MPACT`):**
  ```bash
  cd tests/uvm/tb
  make compile
  make run            # single test
  make run_3way       # 3-way check vs Spike
  ```

### E. VCS — the commercial RTL simulator

- **Where:** `tests/vcs_sim/` (SystemC+VCS top), and VCS is also a backend
  option for cocotb (`rvv_core_mini_axi_sim_cocotb` `--vcs` variant) and for
  the UVM testbench above.
- **Status in this repo.** Tag-gated behind `--config=vcs`; the default
  `.bazelrc` filters out `vcs`-tagged targets so people without licenses
  aren't bothered.
- **Use for:** any of the above where you specifically need VCS (license
  requirement, gate-level netlist, particular coverage flow).
- **Run (requires VCS license, `$VCS_HOME`):**
  ```bash
  bazel build --config=vcs //tests/vcs_sim:core_mini_axi_sim
  ```

## How they relate

```
                    Chisel RTL (.scala) ──► firtool ──► SystemVerilog
                                                            │
              ┌──────────────────────────┬─────────────────┴───────────────┐
              │                          │                                 │
     Verilator compile              Verilator compile                  VCS compile
              │                          │                                 │
     ┌────────┴────────┐         ┌───────┴──────────┐               ┌──────┴────────┐
     │  C++ driver     │         │  cocotb (Python) │               │ cocotb / UVM  │
     │  (path C)       │         │  (path B)        │               │ (paths D, E)  │
     └─────────────────┘         └──────────────────┘               └───────────────┘

           NPUSim (path A) — entirely separate, no RTL involved
```

Paths B, C, D, E all exercise the *same* generated SystemVerilog. Path A does
not — it's a functional ISS that obeys the ISA contract but is unrelated to
the pipeline implementation.

## Decision flowchart

```
Is the question "does my program work?"
  │
  ├─ Yes, and it's an ML model       → NPUSim (A)
  ├─ Yes, and I just want to run it  → Verilator C++ (C)  [01_hello_world]
  └─ Yes, and I want a test          → Verilator + cocotb (B)
Is the question "does the RTL work?"
  │
  ├─ Smoke / dev iteration           → Verilator + cocotb (B)
  ├─ Constrained-random sign-off     → UVM (D)
  └─ License / vendor flow required  → VCS (E)
```

## Source references
- [`tests/npusim_examples/`](../../../tests/npusim_examples/) — NPUSim examples.
- [`tests/cocotb/BUILD`](../../../tests/cocotb/BUILD) — Verilator+cocotb targets.
- [`tests/verilator_sim/`](../../../tests/verilator_sim/) — C++-driven Verilator.
- [`tests/uvm/`](../../../tests/uvm/) — UVM testbench, `Makefile`.
- [`tests/vcs_sim/`](../../../tests/vcs_sim/) — VCS SystemC top.
- [`doc/simulation.md`](../../simulation.md) — VCS setup notes.
- [`doc/repository_guide.md`](../../repository_guide.md) §5 *Verification*.

## Next
- [`software/10_program_anatomy`](../software/10_program_anatomy/README.md) —
  write a program and a cocotb test (path B).
- [`software/12_rvv_intrinsics`](../software/12_rvv_intrinsics/README.md) —
  vectorize a kernel and run it on NPUSim (path A).
- [`software/14_tflite_inference`](../software/14_tflite_inference/README.md) —
  a full ML model on NPUSim.
