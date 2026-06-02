# Coral NPU

Coral NPU is a hardware accelerator for ML inferencing. Coral NPU is an Open Source IP designed by Google Research and is freely available for integration into ultra-low-power System-on-Chips (SoCs) targeting wearable devices such as hearables, augmented reality (AR) glasses and smart watches.

Coral NPU is a neural processing unit (NPU), also known as an AI accelerator or deep-learning processor. Coral NPU is based on the 32-bit RISC-V Instruction Set Architecture (ISA).

Coral NPU includes three distinct processor components that work together: matrix, vector (SIMD), and scalar.

![Coral NPU Archicture](doc/images/arch_data_flow.png)
[Coral NPU Architecture Datasheet](https://developers.google.com/coral/guides/hardware/datasheet)

New to the codebase? See [doc/repository_guide.md](doc/repository_guide.md) for a
directory-by-directory tour of the repository.

## Coral NPU Features
Coral NPU offers the following top-level feature set:

* RV32IMF_Zve32x RISC-V instruction set (specifically `rv32imf_zve32x_zicsr_zifencei_zbb`)
* 32-bit address space for applications and operating system kernels
* Four-stage processor, in-order dispatch, out-of-order retire
* Four-way scalar, two-way vector dispatch
* 128-bit SIMD, 256-bit (future) pipeline
* 8 KB ITCM memory (tightly-coupled memory for instructions)
* 32 KB DTCM memory (tightly-coupled memory for data)
* Both memories are single-cycle-latency SRAM, more efficient than cache memory
* AXI4 bus interfaces, functioning as both manager and subordinate, to interact with external memory and allow external CPUs to configure Coral NPU

## System Requirements

* Bazel 7.4.1
* Python 3.9-3.12 (3.13 support is in progress)
* [SRecord](https://srecord.sourceforge.net/)

## Setup

Before running Bazel for the first time, pin the C/C++ compiler so Bazel's
auto-generated `local_config_cc` toolchain picks up `clang` directly. If
`ccache` is present on `PATH`, Bazel may otherwise generate a `cc_wrapper.sh`
that invokes `/usr/bin/ccache` with no compiler argument, causing builds to
fail with errors like `ccache: invalid option -- 'U'`.

```bash
export CC=/usr/bin/clang
export CXX=/usr/bin/clang++
unset BAZEL_CC
```

Add these to `~/.bashrc` (or the devcontainer setup) to persist them across
shells.

If you have already run Bazel and hit the `ccache` error, regenerate the
toolchain after exporting the variables above:

```bash
bazel clean --expunge
```

You can verify the toolchain is correct by checking the last meaningful line
of the generated wrapper — it should call `clang`, not `ccache`:

```bash
grep -A1 "Call the C++" "$(bazel info output_base)/external/local_config_cc/cc_wrapper.sh"
# Expected: /usr/lib/llvm-19/bin/clang "$@"
```

## Quick Start

```bash
# Ensure that test suite passes
bazel run //tests/cocotb:core_mini_axi_sim_cocotb

# Build a binary
bazel build //examples:coralnpu_v2_hello_world_add_floats

# Build the Simulator (non-RVV for shorter build time):
bazel build //tests/verilator_sim:core_mini_axi_sim

# Run the binary on the simulator:
bazel-bin/tests/verilator_sim/core_mini_axi_sim --binary bazel-out/k8-fastbuild-ST-dd8dc713f32d/bin/examples/coralnpu_v2_hello_world_add_floats.elf

# Run with an instruction trace printed to stdout:
bazel-bin/tests/verilator_sim/core_mini_axi_sim \
  --binary bazel-out/k8-fastbuild-ST-dd8dc713f32d/bin/examples/coralnpu_v2_hello_world_add_floats.elf \
  --instr_trace

# Run and dump a VCD waveform (open in GTKWave / Surfer):
bazel-bin/tests/verilator_sim/core_mini_axi_sim \
  --binary bazel-out/k8-fastbuild-ST-dd8dc713f32d/bin/examples/coralnpu_v2_hello_world_add_floats.elf \
  --trace
```

Note: the `hello_world_add_floats` example is a silent computation (no `printf`),
so a plain run exits with only SystemC's `Simulation stopped by user.` banner.
That message just means the testbench called `sc_stop()` after detecting program
completion — it does not indicate an error. Use `--instr_trace` or `--trace`
above to observe execution.


![](doc/images/Coral_Logo_200px-2x.png)
