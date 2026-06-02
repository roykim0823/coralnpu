# 00. Setup — devcontainer, Bazel, and a sanity build

> Part of the [CoralNPU tutorial series](../README.md) · *Onramp.*

**Goal.** Confirm that the devcontainer and Bazel toolchain are configured
correctly, then run a sanity build and the cocotb smoke test to prove
end-to-end the environment works.

**Prereqs.** A working VS Code Dev Containers / `devcontainer` CLI setup, or
equivalent. This repo assumes you're inside the project's devcontainer (see
`.devcontainer/Dockerfile`).

**Time.** ~10 minutes (most of it Bazel's first-time build).

## What you'll learn
1. How to verify the toolchain is wired up correctly (ccache, clang, Bazel).
2. How to run the project's smoke test suite (`tests/cocotb`).
3. The common failure modes you'll hit and what they mean.

## Steps

### 1. Open the devcontainer

Open this repository in VS Code with the Dev Containers extension and choose
**Reopen in Container**. Wait for the post-create build to finish. The
container ships with: Bazel 7.4.1, clang-19, ccache, Verilator, SystemC,
Python 3.12, and the prebuilt RISC-V toolchain (fetched by Bazel on first
build).

### 2. Verify ccache + clang wiring

This repo uses ccache through a wrapper-script masquerade. Confirm it's
healthy:

```bash
clang --version
cat $(which clang)            # should be a #!/bin/sh wrapper, not a binary
ccache --show-stats | head
```

Expected:
- `clang --version` reports `Ubuntu clang version 19.x.y`.
- `which clang` resolves to `/usr/local/bin/clang`, and the file is a tiny
  shell script that ends with `exec /usr/bin/ccache /usr/bin/clang "$@"`.
- `ccache --show-stats` runs without errors.

### 3. Bazel info

```bash
bazel info workspace release
```

Expected:
- `workspace: /workspaces/.../coralnpu` (or wherever you mounted the repo)
- `release: release 7.4.1`

### 4. Sanity-build the hello-world example

```bash
bazel build //examples:coralnpu_v2_hello_world_add_floats
```

This compiles a tiny RISC-V program with the cross-compile toolchain. First
time it'll fetch the RISC-V toolchain tarball and external deps; expect a few
minutes.

Expected:
- Final line: `INFO: Build completed successfully, …`
- Produces `bazel-bin/examples/coralnpu_v2_hello_world_add_floats.elf` (and a
  `.bin`).

### 5. Smoke-test the cocotb suite

```bash
bazel run //tests/cocotb:core_mini_axi_sim_cocotb
```

This builds the Verilator RTL simulator from the Chisel-generated SystemVerilog
and runs the cocotb test harness against it. First time is slow (Verilator
compile); cached runs are quick.

Expected:
- Verilator compiles `VCoreMiniAxi`.
- Cocotb tests run — basic R/W, WFI, CSR, stress, RiscV tests, etc.
- Final summary `*** TESTS PASSED ***`.

If you got here, the environment is good. Continue to
[`01_hello_world`](../01_hello_world/README.md).

## Source references
- [`.devcontainer/Dockerfile`](../../../.devcontainer/Dockerfile) — the
  devcontainer image, including the ccache wrapper scripts.
- [`.bazelrc`](../../../.bazelrc) — global Bazel flags (C++17, tag filters,
  platform default).
- [`README.md`](../../../README.md) — quick-start commands.
- [`doc/simulation.md`](../../simulation.md) — VCS sim setup (optional).

## Common pitfalls

- **`/usr/bin/ccache: invalid option -- 'U'`** during a Bazel build.
  Cause: an older devcontainer image where `/usr/local/bin/clang` is a symlink
  straight to `ccache`. Bazel realpath-collapses the symlink and invokes ccache
  with no compiler argument; the first compiler flag (`-U_FORTIFY_SOURCE`)
  parses as ccache's `-U` option and dies.
  Fix: rebuild the devcontainer with the current
  [`Dockerfile`](../../../.devcontainer/Dockerfile) — the masquerade is now a
  set of wrapper *scripts* (regular files), which survive realpath. As a
  short-term workaround inside a running container, run
  `for n in clang clang++ cc c++; do sudo bash -c "printf '#!/bin/sh\nexec /usr/bin/ccache /usr/bin/%s \"\$@\"\n' $n > /usr/local/bin/$n && chmod +x /usr/local/bin/$n"; done`.

- **`Repository 'toolchain_coralnpu_v2' is not defined`** on first build.
  Cause: external repo fetch hasn't completed (network slow or blocked).
  Fix: re-run the build; if it persists, check `bazel fetch //...` and your
  proxy settings.

- **Bazel `cc_wrapper.sh` baked in the wrong CC.** If `bazel info output_base`
  shows a `local_config_cc/cc_wrapper.sh` that calls `ccache` directly,
  regenerate:
  ```bash
  bazel clean --expunge
  ```
  Then redo step 4.

- **`Verilator: error: ...` during the cocotb build.** Usually a stale
  Verilator object cache. `bazel clean` and rebuild.

## Next
- [`01_hello_world`](../01_hello_world/README.md) — run a binary on the
  Verilator simulator and inspect the instruction trace.
