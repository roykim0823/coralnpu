<!--
Repository layout & subsystem guide for the CoralNPU hardware project.
This is a navigation/orientation doc; for the project intro see ../README.md,
and for architecture details see overview.md and microarch/.
-->

# CoralNPU — Repository Layout

CoralNPU is an open-source, ultra-low-power **ML accelerator** built around a
32-bit RISC-V core (`rv32imf_zve32x_zicsr_zifencei_zbb`). It fuses three
processing elements — a **scalar** RISC-V frontend, a **vector/SIMD** core, and
a **matrix (outer-product MAC)** engine — into one design intended for wearables
(hearables, AR glasses, smartwatches).

This document explains how the repository is organized and what each directory
is for. For the user-facing intro and quick-start commands see
[`README.md`](../README.md); for architecture see [`doc/overview.md`](overview.md).

---

## 1. The big picture

The repo is one Bazel workspace (`WORKSPACE`, Bazel 7.4.1 pinned in
`.bazelversion`) spanning the entire stack of a hardware accelerator:

| Layer | Where | Language(s) |
|-------|-------|-------------|
| RTL design (most of the chip) | `hdl/chisel/` | Chisel/Scala → Verilog |
| RTL design (RVV vector backend) | `hdl/verilog/rvv/` | hand-written SystemVerilog |
| SoC integration + FPGA bring-up | `fpga/` | SystemVerilog + C firmware |
| On-device software / ML kernels | `sw/`, `examples/`, `lib/` | C/C++ (RVV intrinsics), pybind |
| Cross-compile toolchain & build rules | `toolchain/`, `rules/`, `platforms/` | Starlark, shell |
| Verification (4 methodologies) | `tests/`, `hw_sim/`, `coralnpu_test_utils/` | Python (cocotb), C++, UVM SV |
| Vendored dependencies | `third_party/` | mixed |
| Docs | `doc/` | Markdown |

A useful mental model of the data flow: **Chisel** generates most of the RTL;
the **RVV backend** is dropped in as hand-written SystemVerilog; everything is
stitched together in `fpga/` into a SoC; programs are cross-compiled by the
RISC-V **toolchain** and run either on the **simulators** (Verilator / VCS /
SystemC / the pure-software NPUSim) or on a **Lattice Nexus FPGA**.

---

## 2. Hardware design — `hdl/`

The RTL source of the chip. ~120 Scala files + ~137 SystemVerilog/Verilog files.

### `hdl/chisel/` — the bulk of the design (Chisel/Scala)
Compiled to Verilog via the FIRRTL/firtool flow. Organized under `src/`:

- **`coralnpu/`** — the core. Scalar pipeline (`scalar/Fetch`, `Decode`, `Alu`,
  `Mlu`, `Lsu`, `Dvu`, `Fpu`, `Bru`, `Csr`, `Regfile`, `SCore`), the RVV vector
  glue (`rvv/RvvCore`, `RvvDecode`, `RvvInterface`), caches (`L1ICache`,
  `L1DCache`), memories (`Sram`, `TCM`), top-level assembly (`Core`, `CoreAxi`,
  `CoreTlul`, `Fabric`), and config (`Parameters.scala`, `MemorySize.scala`).
  `Parameters.scala` sets the datapath width (`lsuDataBits = 256`) and TCM sizes.
- **`bus/`** — interconnect: AXI ↔ TileLink-UL bridges (`Axi2TLUL`, `TLUL2Axi`),
  crossbars/FIFOs, `DmaEngine`, peripherals (`GPIO`, `SpiMaster`), interrupt
  controllers (`Clint`, `Plic`), and SECDED integrity.
- **`common/`** — reusable primitives: FIFOs, arbiters, aligners, scatter/gather,
  floating-point (`Fp`, `Fma`), and Verilog-generation/test utilities.
- **`soc/`** — multi-core/system assembly: `CoralNPUXbar`, `CoralNPUChiselSubsystem`,
  crossbar config. These produce the `coralnpu_chisel_subsystem_*` IP used in `fpga/`.
- **`peripherals/`** — generic peripheral-attach framework.

Build via custom `chisel_library` / `chisel_cc_library` / `chisel_test` rules
(see `rules/chisel.bzl`). The CI artifact is `RvvCoreMiniAxi.sv` (emitted Verilog).

### `hdl/verilog/` — hand-written RTL, primarily the RVV backend
- **`rvv/design/`** (~56 files) — the RISC-V Vector execution engine: decode →
  dispatch/scoreboard → execution units (`alu`, `mul`/`mac` for ML, `div`,
  `fma`/`fdiv` floating-point via cvfpu, `pmtrdt` permute/reduce) → reorder
  buffer (`rob`) → retire → vector register file (`vrf`).
- **`rvv/common/`** (~17 files) — synthesizable cells (adders, compressors,
  flip-flops, FIFOs, round-robin arbiter).
- **`rvv/inc/`** — `.svh` headers with parameters/macros/assertions.
- **`rvv/sve/`** — UVM testbenches for the RVV backend and its FIFOs.
- Top-level cells: `ClockGate.sv`, `RstSync.sv`, SRAM compiler models (`Sram*.v`).

> Note on vector width: `doc/overview.md` describes 256-bit vector registers
> (int32×8); the RVV extension backend is configured at **VLEN=128**
> (`ZLEN_128`/`VLEN_128` defines), and the top-level README lists "128-bit SIMD,
> 256-bit (future) pipeline." Treat 128-bit as the current RVV config.

### `hw_sim/` — C++ harness to *drive* a Verilator build of the core
`coralnpu_simulator.h` (abstract interface), `core_mini_axi_simulator.cc` /
`core_mini_axi_wrapper.h` (Verilator+DPI wrapper), `mailbox.h`, `hw_primitives.*`.
Produces `libcoralnpu_simulator.so` (+ `_rvv` variant). API: `ReadTCM`/`WriteTCM`,
`Run(start_addr)`, `WaitForTermination`, mailbox IPC. This is the C++ entry point
external software uses to talk to a simulated core.

### `lib/`
A single `chisel_lib` Bazel target aggregating the Chisel/FIRRTL Scala
dependencies used by everything under `hdl/chisel`.

---

## 3. SoC integration & FPGA — `fpga/`

Turns the generated core into a deployable system on a **Lattice Nexus** FPGA,
plus a Verilator simulation target.

- **`rtl/`** — hand-written SoC RTL. `coralnpu_soc.sv` (top-level integrator:
  Chisel subsystem + TLUL fabric + AXI4 + ISP/I2C/SPI/GPIO/UART), `chip_nexus.sv`
  (Lattice Nexus board wrapper: DDR4, clocks, pins), `chip_verilator.sv`
  (simulation top), `autoboot.sv`, clock generators.
- **`ip/`** — IP blocks: `coralnpu_chisel_subsystem_{default,highmem}` (8KB/32KB
  vs 1MB/1MB TCM variants generated from Chisel), `ispyocto` (camera/ISP
  front-end), `coralnpu_tlul`, `i2c_master`, and **DPI simulation models**
  (`spi_dpi_master`, `gpio_dpi`, `display_dpi`, `s25fl512s_dpi` flash, `ddr4_stub`).
- **`sw/`** — embedded C firmware: ROM bootloader (`rom_boot/`, loads code from a
  ZIP in SPI flash into ITCM/DTCM), drivers (`uart`, `spi`, `spi_flash`, `i2c`,
  `gpio`, `dma`, `clk`, `display_hal`), and an ISP camera test.
- **`nexus/`** — `load_bitstream.sh`, deploys a bitstream to the Nexus board over
  SSH/UART.
- FuseSoC manifests (`*.core`) and `main.cc` (Verilator C++ testbench) drive
  synthesis and simulation. `run_isp_cam_sim.sh` (repo root) runs the ISP camera sim.

---

## 4. Software & examples — `sw/`, `examples/`

On-device code plus the host-side simulator bindings.

- **`sw/coralnpu_sim/`** — the **NPUSim** instruction-set simulator exposed to
  Python via pybind11. `coralnpu_v2_sim_pybind.cc` wraps the C++
  `CoralNPUV2Simulator`; `coralnpu_v2_sim_utils.py` is the high-level Python API
  (`LoadProgram`, `Run`, `Step`, `ReadMemory`/`WriteMemory`, `GetCycleCount`,
  configurable ITCM/DTCM/DDR ranges). This is what ML-model tutorials drive.
- **`sw/opt/`** — optimized kernels. `rvv_opt.h` (vectorized `Memcpy`/`Memset`
  via RVV intrinsics) and `litert-micro/` — RVV-accelerated TFLite-Micro ops
  (`conv`, `depthwise_conv`, `pooling`, `fully_connected`, `logistic`) with
  per-channel quantization and weight repacking.
- **`sw/utils/`** — `utils.h` (cycle/instret CSR readers) and `nexus_loader/` (a
  host CLI that loads ELFs onto hardware over SPI via FTDI/libusb).
- **`examples/`** — minimal programs: `hello_world_add_floats.cc` and
  `rvv_add_intrinsic.cc` (vector widening add). Built as `coralnpu_v2_binary`
  → `.elf` for the simulators.

Programs are produced by the `coralnpu_v2_binary` rule (see §6).

---

## 5. Verification — `tests/`, `coralnpu_test_utils/`

CoralNPU is verified at four levels; the same DUT and many test binaries are
shared across simulators.

| Dir | Methodology | What it does |
|-----|-------------|--------------|
| **`tests/cocotb/`** | Python cocotb testbenches | Largest suite. Drives `CoreMiniAxi`/`RvvCoreMiniAxi` over AXI. Subdirs: `rvv/` (load/store, arithmetic, `ml_ops` matmul), `tlul/` (bus protocol), `exceptions/`, `tutorial/` (counters, TFLite-micro). Runs on **both** Verilator and VCS. |
| **`tests/verilator_sim/`** | Verilator C++ sim | The main C++ simulator driver (`coralnpu/core_mini_axi_sim.cc`, `core_mini_axi_tb.*`), plus cache/bus unit testbenches. Supports `--instr_trace` and `--trace` (FST waveform). |
| **`tests/systemc/`** | SystemC/TLM | Infrastructure (`Xbar.h`, `instruction_trace.*`) used by the Verilator testbenches, not a standalone suite. |
| **`tests/vcs_sim/`** | Synopsys VCS + SystemC | Mixed-language sim (`top.cc`, `core_mini_axi.map`). License/tag-gated. |
| **`tests/uvm/`** | UVM 1.2 (SystemVerilog) | Formal-grade constrained-random verification of `RvvCoreMiniVerificationAxi`, including **3-way cosim against the Spike ISS** via MPACT. `make run` / `make run_3way`. |
| **`tests/npusim_examples/`** | NPUSim (pure software) | `npusim_run_mobilenet.py` runs full MobileNet-v1 int8 inference through the Python ISS — no RTL. (`bazel run tests/npusim_examples:npusim_run_mobilenet`.) |

**`coralnpu_test_utils/`** — shared Python test infrastructure: cocotb AXI/TLUL
masters & slaves (`core_mini_axi_interface.py`, `TileLinkULInterface.py`,
`axi_slave.py`), a sim fixture, PyOCD GDB-server glue, FTDI/SPI hardware drivers
(`ftdi_spi_master.py` wraps the C++ `nexus_loader`), RVV type helpers, and a
SECDED golden model.

The simulator landscape: **Verilator** (fast, default, used in CI) and **VCS**
(commercial, gate-level, tag-gated) both run the cocotb suite against the same
generated RTL; **UVM** adds formal coverage + Spike cosim; **NPUSim** is a
separate pure-software ISS for ML workload profiling — it does not simulate RTL.

---

## 6. Build system — `toolchain/`, `platforms/`, `rules/`

How RISC-V binaries and the simulators get built. ~21 Starlark rule files.

- **`toolchain/`** — the RISC-V cross-compile toolchain.
  `cc_toolchain_config.bzl` + `BUILD.bazel` define `coralnpu_v2_toolchain` and a
  `*_semihosting` variant (stdio redirected to host). `wrappers/driver.sh` (and
  `gcc`/`clang`/`ld`/`ar`/`objcopy` shims) bridge Bazel to the external prebuilt
  `toolchain_coralnpu_v2` (rv32imf_zve32x GCC+LLVM). `crt/` holds C-runtime
  startup, `host_clang/` the host toolchain. Note: `printf`-float is disabled by
  default to fit the 8 KB ITCM budget.
- **`platforms/`** — Bazel `platform()`/constraint defs: `coralnpu_v2` (bare-metal)
  and `coralnpu_v2_semihosting`, plus `cpu/` and `os/` constraint values selected
  by `.bazelrc` (`--config=coralnpu_v2`).
- **`rules/`** — the custom rule library. Key macros:
  - `coralnpu_v2_binary` (`coralnpu_v2.bzl`) — compile/link an embedded RISC-V
    `.elf`/`.bin` with a platform transition and generated linker script.
  - `linker.bzl` (`generate_linker_script`) — templated ITCM/DTCM/heap/stack layout
    (default 8 KB/32 KB; highmem 1 MB/1 MB).
  - `chisel.bzl`, `coco_tb.bzl`, `vcs.bzl`, `lint.bzl` (`lint/`), `mpact.bzl` —
    Chisel, cocotb, VCS sim, Verilog lint, and MPACT-sim integration.
  - `repos.bzl` / `deps.bzl` — external repo & Scala/Maven dependency declarations.
  - `host_cpus.bzl` — detects core count to tune Verilator threads / make jobs.

Top-level build files: `WORKSPACE` (pulls in rules_hdl, rules_scala, rules_python,
protobuf, abseil, googletest, TFLite-Micro, OpenTitan, MPACT-RISCV, the prebuilt
toolchain, FreeRTOS, pybind11), `.bazelrc` (C++17, workspace mode, tag filters
hiding `vcs`/`synthesis`/`power` by default, BFD linker), and `jtag-sim.cfg`
(OpenOCD RISC-V TAP for debug).

---

## 7. Vendored dependencies — `third_party/`

Each subdir vendors an external project (mostly Apache-2.0, several patched):

| Dir | What / why |
|-----|-----------|
| `common_cells` | PULP reusable RTL primitives |
| `cvfpu` | Configurable FP unit — the RVV FMA/divide backend (patched ×5) |
| `fpu_div_sqrt_mvp` | FP divide/sqrt unit |
| `freertos` | FreeRTOS-Kernel 11.1.0 (optional RTOS; default config provided) |
| `libsystemctlm-soc` | SystemC/TLM-2.0 SoC modeling for sims |
| `llvm-firtool` | FIRRTL→Verilog compiler for the Chisel flow |
| `python` | Python deps/requirements |
| `riscv-tests` | Official RISC-V ISA test suite |
| `rocket_chip` | Rocket Chip generator (reference IP) |
| `rules_hdl` | Bazel HDL rules — Verilator/cocotb/VCS (patched ×10) |
| `RVVI` | RISC-V Verification Interface (retirement trace) |
| `spike` | RISC-V ISS — the UVM cosim golden reference (patched ×5) |
| `srecord` | S-record/`.vmem` tooling |
| `systemc` | SystemC simulation kernel |
| `tflite-micro` | TensorFlow Lite Micro — on-device ML inference |
| `waveshare_display` | LCD display driver for the FPGA demo |

---

## 8. Documentation — `doc/`

- **`overview.md`** — architecture: scalar/vector cores, the outer-product MAC,
  stripmining, cache hierarchy.
- **`integration_guide.md`** — AXI/TileLink interfaces, memory map, CSRs, boot, debug.
- **`simulation.md`** — VCS simulator setup, env vars, ccache notes.
- **`microarch/`** — pipeline (`microarch.md`), `mlu.md`, `lsu.md`, `dispatch.md`,
  `debug.md`.
- **`peripherals/dma.md`** & **`sw/dma.md`** — DMA engine hardware + software.
- **`tutorials/`** — guided walkthroughs. Start at the
  [tutorials index](tutorials/README.md); it lists an onramp (`00_setup`,
  `01_hello_world`, `02_simulator_landscape`) plus role-based deep dives under
  `software/`, `hardware/`, `verification/`, `fpga/`, and `build_system/`.
- **`images/`** — block/architecture/MAC diagrams.

---

## 9. Misc / repo meta

- **`.github/workflows/main.yml`** — CI: builds the Chisel core and emits/release
  `RvvCoreMiniAxi.sv` + `.zip` (30-min timeout).
- **`.claude/settings.local.json`** — local Claude Code permissions (Bazel/cache).
- **`utils/`** — repo tooling: `get_workspace_status.sh` (version stamping),
  UVM regression runners, and `coralnpu_soc_loader/` (Python sim runner driving
  the Verilator SoC over the SPI DPI model).
- **`CONTRIBUTING.md`** — CLA + Gerrit review process (this is a Google
  Research open-source project).

---

## Where to start

| I want to… | Go to |
|------------|-------|
| Run the test suite | `bazel run //tests/cocotb:core_mini_axi_sim_cocotb` |
| Build & run a program on the RTL sim | `examples/` + `tests/verilator_sim/` (see README) |
| Run an ML model without RTL | `tests/npusim_examples/` (NPUSim) |
| Understand the core | `doc/overview.md`, `doc/microarch/`, `hdl/chisel/src/coralnpu/` |
| Modify the vector engine | `hdl/verilog/rvv/design/` |
| Bring up the FPGA | `fpga/` (RTL + firmware), `fpga/nexus/load_bitstream.sh` |
| Add a build rule | `rules/` |
