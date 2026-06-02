# CoralNPU Tutorials

A guided tour of the CoralNPU stack. Tutorials are numbered so you can read them
in order, and grouped by role so you can jump straight to what's relevant.

Each tutorial has a **goal**, **prereqs**, **steps with expected output**,
**source references**, and a **next** pointer. Pair each one with the project
[`README.md`](../../README.md) (intro + quick-start) and [`doc/repository_guide.md`](../repository_guide.md)
(directory layout).

> **Status legend:** ✅ written · 🚧 stub / partial · ⬜ planned (not started)

## Onramp — start here

| #  | Tutorial | Status |
|----|----------|--------|
| 00 | [Setup — devcontainer, Bazel, sanity build](00_setup/README.md) | ✅ |
| 01 | [Hello world — compile and run on the Verilator sim](01_hello_world/README.md) | ✅ |
| 02 | [Simulator landscape — when to use Verilator vs NPUSim vs UVM vs VCS](02_simulator_landscape/README.md) | ✅ |

## Software path — writing code that runs on the NPU

| #  | Tutorial | Status |
|----|----------|--------|
| 10 | [Program anatomy — buffers, linker, cocotb testbench](software/10_program_anatomy/README.md) | ✅ |
| 11 | Memory map and CSRs — AXI map, `mcycle` / `minstret`, performance counters | ⬜ |
| 12 | [RVV intrinsics — vectorize a kernel by hand](software/12_rvv_intrinsics/README.md) | ✅ |
| 13 | Optimized kernels — `sw/opt/rvv_opt.h` and the TFLite-Micro ops | ⬜ |
| 14 | [TFLite-Micro inference on NPUSim — MobileNet v1](software/14_tflite_inference/README.md) | ✅ |
| 15 | Debugging with GDB and PyOCD over the JTAG sim TAP | ⬜ |

## Hardware path — modifying the RTL

| #  | Tutorial | Status |
|----|----------|--------|
| 20 | Chisel module walkthrough — read one design end-to-end | ⬜ |
| 21 | Chisel → Verilog — how the FIRRTL/firtool flow emits `.sv` | ⬜ |
| 22 | Adding a peripheral — a new TLUL device in Chisel | ⬜ |
| 23 | The RVV backend tour — the hand-written SystemVerilog pipeline | ⬜ |
| 24 | Linting RTL with `vcstatic_lint` | ⬜ |

## Verification path — proving the design works

| #  | Tutorial | Status |
|----|----------|--------|
| 30 | Writing a cocotb test — driver, monitor, scoreboard | ⬜ |
| 31 | Running on VCS — the tag-gated commercial flow | ⬜ |
| 32 | UVM testbench tour — `tests/uvm/` and how to add a test | ⬜ |
| 33 | Spike co-simulation — the 3-way correctness check | ⬜ |
| 34 | Waveforms — FST + GTKWave / Surfer | ⬜ |

## FPGA path — getting the chip on a board

| #  | Tutorial | Status |
|----|----------|--------|
| 40 | Building a bitstream — FuseSoC + Lattice Nexus toolchain | ⬜ |
| 41 | Deploying to the Nexus board — `load_bitstream.sh` | ⬜ |
| 42 | ROM boot and the SPI-flash ZIP loader | ⬜ |
| 43 | Host → FPGA over SPI — `nexus_loader` and FTDI | ⬜ |
| 44 | The ISP camera demo — `run_isp_cam_sim.sh` | ⬜ |

## Build system

| #  | Tutorial | Status |
|----|----------|--------|
| 50 | Bazel basics for this repo — tag filters, where rules live | ⬜ |
| 51 | The `coralnpu_v2_binary` macro — anatomy of a target | ⬜ |
| 52 | Linker scripts — TCM sizing, heap/stack, `generate_linker_script` | ⬜ |

---

## Conventions

- **Code lives in `examples/`** (one Bazel target per tutorial that needs it).
  Tutorial docs *reference* example code by path; they don't duplicate it.
- **Tutorial layout.** Each tutorial is a single `README.md` in its own
  directory. The directory name is the slug (`12_rvv_intrinsics`), the heading
  inside starts with the same number.
- **Working assumption.** Commands assume you're inside the devcontainer (see
  [`00_setup`](00_setup/README.md)) and at the repository root.

## Contributing a tutorial

When adding a tutorial, please follow this template (so they all read alike):

```markdown
# NN. <Title>

> Part of the [CoralNPU tutorial series](../../README.md) · *<Path name>.*

**Goal.** One sentence — what the reader can do after.
**Prereqs.** Links to earlier tutorials, required env.
**Time.** Rough estimate.

## What you'll learn
- bullet
- bullet

## Steps
### 1. <Step name>
    bazel run …
Expected output:
    …
Why this works: one or two lines.

## Source references
- `path/to/code` — what it is.

## Common pitfalls
- "Error X" → cause + fix.

## Next
- Suggested next tutorial(s).
```

A note on maintenance: every tutorial that runs code should also be run by a
CI job (or at minimum `bazel build`-ed). Otherwise these docs rot fast.
