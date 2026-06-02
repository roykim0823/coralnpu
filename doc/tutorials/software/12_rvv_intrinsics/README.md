# 12. RVV intrinsics — vectorize a kernel by hand

> Part of the [CoralNPU tutorial series](../../README.md) · *Software path.*

**Goal.** Read and run a small RISC-V Vector (RVV) kernel that adds two
1024-element int8 arrays and widens the result to int16. Understand the four
intrinsics it uses — `vsetvl`, `vle`, `vse`, `vwadd` — and the LMUL/SEW
parameters that drive them.

**Prereqs.** [`00_setup`](../../00_setup/README.md),
[`01_hello_world`](../../01_hello_world/README.md),
[`02_simulator_landscape`](../../02_simulator_landscape/README.md).

**Time.** ~15 minutes.

## What you'll learn
1. The shape of an RVV intrinsic kernel: `vsetvl` → load → compute → store.
2. The meaning of the `i8m4` / `i16m8` suffixes (SEW + LMUL).
3. Why a widening op like `vwadd` doubles the destination LMUL.
4. How to run the resulting binary on NPUSim, since the default Verilator sim
   doesn't include the vector extension.

## The kernel

[`examples/rvv_add_intrinsic.cc`](../../../../examples/rvv_add_intrinsic.cc):

```cpp
#include <riscv_vector.h>
#include <string.h>

int8_t  input_1[1024];
int8_t  input_2[1024];
int16_t output[1024];

int main() {
  memset(input_1, 1, 1024);
  memset(input_2, 6, 1024);

  const int8_t* input1_ptr = &input_1[0];
  const int8_t* input2_ptr = &input_2[0];
  int16_t*      output_ptr = &output[0];

  for (int idx = 0; (idx + 31) < 1024; idx += 32) {
    vint8m4_t  input_v2 = __riscv_vle8_v_i8m4(input2_ptr + idx, 32);
    vint8m4_t  input_v1 = __riscv_vle8_v_i8m4(input1_ptr + idx, 32);

    vint16m8_t temp_sum = __riscv_vwadd_vv_i16m8(input_v1, input_v2, 32);
    __riscv_vse16_v_i16m8(output_ptr + idx, temp_sum, 32);
  }
  return 0;
}
```

### Decoding the intrinsic names

Each intrinsic encodes four things in its name: the op, the SEW (element
width), the LMUL (register grouping), and the element type.

| Intrinsic | Op | SEW | LMUL | Element |
|-----------|----|-----|------|---------|
| `__riscv_vle8_v_i8m4` | vector load, element-strided | 8 bits | m4 (4 regs grouped) | int8 |
| `__riscv_vwadd_vv_i16m8` | widening add, vector-vector | 16 bits *(dst)* | m8 *(dst)* | int16 |
| `__riscv_vse16_v_i16m8` | vector store | 16 bits | m8 | int16 |

The last argument `32` is the **vector length (`vl`)**: how many elements this
op should touch. On a VLEN=128 machine with int8 + LMUL=4, the maximum `vl` is
`(VLEN * LMUL) / SEW = (128 * 4) / 8 = 64`. Our loop uses 32, well within that.

### Why `vwadd` widens LMUL

`vwadd` is a *widening* add: int8 + int8 → int16. The result has twice the
element width, so to hold the same number of elements it needs twice the
register grouping — hence `i8m4` inputs become an `i16m8` result. Try writing
it as `__riscv_vwadd_vv_i16m4(...)` and the compiler will refuse: the LMUL
arithmetic has to work out.

## Steps

### 1. Build the binary

```bash
bazel build //examples:coralnpu_v2_rvv_add_intrinsic
```

Inspect the disassembly to confirm it really emitted vector instructions:

```bash
ELF=$(find bazel-out -name coralnpu_v2_rvv_add_intrinsic.elf -print -quit)
$(bazel info output_base)/external/toolchain_coralnpu_v2/bin/riscv32-unknown-elf-objdump \
    -d "$ELF" | grep -E 'vle|vse|vwadd' | head
```

Expected: you should see `vle8.v`, `vwadd.vv`, and `vse16.v` instructions.

### 2. Run it on NPUSim

We use NPUSim (the Python ISS) rather than the default Verilator sim from
[`01_hello_world`](../../01_hello_world/README.md), because the latter
instantiates the base non-RVV core (`VCoreMiniAxi`) and would trap on the
first vector instruction. The RVV variant of the Verilator sim is wired up
through cocotb; if you want that path, see
[`02_simulator_landscape`](../../02_simulator_landscape/README.md) path B.

Create a quick host runner — `run_rvv_add.py`:

```python
from python.runfiles import runfiles
from sw.coralnpu_sim.coralnpu_v2_sim_utils import CoralNPUV2Simulator
import numpy as np

r = runfiles.Create()
elf = r.Rlocation('coralnpu_hw/examples/coralnpu_v2_rvv_add_intrinsic.elf')

sim = CoralNPUV2Simulator(highmem_ld=True, exit_on_ebreak=True)
entry, symbols = sim.get_elf_entry_and_symbol(elf, ['output'])

sim.run()
sim.wait()

result = np.frombuffer(sim.read_memory(symbols['output'], 1024 * 2),
                       dtype=np.int16)
print('first 8 results:', result[:8])
assert np.all(result == 7), f'expected all 7, got {result[:8]}…'
print('OK')
```

Expected output:
```
first 8 results: [7 7 7 7 7 7 7 7]
OK
```

(`memset(1)` + `memset(6)` → all elements are 7. Trivial, but it proves the
kernel ran end-to-end.)

The `bazel run` integration follows the same pattern as
[`tests/npusim_examples/BUILD`](../../../../tests/npusim_examples/BUILD)
(see `npusim_run_mobilenet` as the template).

## Source references
- [`examples/rvv_add_intrinsic.cc`](../../../../examples/rvv_add_intrinsic.cc) — the kernel.
- [`sw/opt/rvv_opt.h`](../../../../sw/opt/rvv_opt.h) — the same pattern applied
  to `memcpy` and `memset`. Worth reading once you've grokked the intrinsics.
- [`sw/opt/litert-micro/`](../../../../sw/opt/litert-micro/) — full-strength
  examples in `conv.cc`, `depthwise_conv.cc`, etc., where the same primitives
  are used to accelerate TFLite-Micro ops.
- The official RVV intrinsics reference: <https://github.com/riscv-non-isa/rvv-intrinsic-doc>.

## Common pitfalls

- **Illegal instruction trap on the default sim.** You ran the ELF on
  `core_mini_axi_sim` (no RVV). Use NPUSim, or the cocotb `rvv_*` variants.

- **`vsetvl` mismatch.** Calling `__riscv_vwadd_vv_i16m8(..., 64)` with int8
  inputs at LMUL=4 is fine on VLEN=128 (max element count is 64). At
  LMUL=8 it would be 32, and asking for `vl=64` would silently get clamped.
  Always derive `vl` from `__riscv_vsetvl_e<SEW>m<LMUL>(n)` if `n` isn't a
  compile-time constant.

- **Forgetting that `vwadd` is `_vv` not `_wv`.** `vwadd.vv` is two
  non-widened inputs producing a widened result. `vwadd.wv` is widened+narrow
  → widened. Different intrinsic name (`__riscv_vwadd_wv_*`), different LMUL
  on the first input.

## Next
- [`13_optimized_kernels`](../) *(planned)* — read `sw/opt/rvv_opt.h` and the
  litert-micro op kernels as scaled-up case studies.
- [`14_tflite_inference`](../14_tflite_inference/README.md) — see the same
  primitives drive a full MobileNet inference.
