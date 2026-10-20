---
name: PolyStrata Build
description: Build PolyStrata - which command produces which target, the CMake options, the CUDA / HIP / SYCL split, the installer's families, and how to build for an AMD card. Use when a build fails, when a new target or test needs compiling, when configuring for an AMD card, or when checking that a change links.
---

# Building PolyStrata

There is no single entrypoint — **which target you want decides the command**.

| Want | Command |
| --- | --- |
| The engine (`strata`) | `cmake -S . -B build -DSTRATA_ENABLE_CUDA=ON -DSTRATA_BUILD_TESTS=ON` then `cmake --build build --target strata` |
| The poly engine (`strata-poly`) | the same configure, then `--target strata-poly` |
| An AMD card, wave32 RDNA | `-DSTRATA_ENABLE_HIP=ON` (gfx1100, gfx1201; gfx1012/gfx1030 unvalidated) |
| An AMD card, gfx906 wave64 | `-DSTRATA_HIP_GFX906=ON` — it forces CUDA on and HIP off, so every "CUDA" branch of the tree is the one taken |
| Intel Arc | `-DSTRATA_ENABLE_SYCL=ON` — **a separate project**: it `return()`s from the top-level file, and it is exclusive with CUDA and HIP |
| The installer | `./setup.sh` (Linux) or `SETUP.bat` (Windows) |
| The server | `python polystrata.py --model <name>` |

## The options

| Option | Default | Meaning |
| --- | --- | --- |
| `STRATA_ENABLE_CUDA` | OFF | The CUDA targets. |
| `STRATA_ENABLE_HIP` | OFF | wave32 AMD RDNA (gfx1100, gfx1201). |
| `STRATA_HIP_GFX906` | OFF | AMD gfx906 through HIP, wave64. Forces CUDA on, HIP off. |
| `STRATA_ENABLE_SYCL` | OFF | The Intel Arc engine from `sycl/`. A separate project. |
| `STRATA_PREFILL_MMQ` | OFF | The optional GGML MMQ prefill path, for HIP. |
| `STRATA_MMQ_KQUANTS` | ON | The MMQ prompt kernels for Q4_K / Q5_K / Q5_1 / Q6_K — the experts of Unsloth's K-quant files, and on HIP the dense GGUF projections. |
| `STRATA_Q6K_EXPERTS` | ON | The Q6_K gate/up grouped expert kernels (UD-Q6_K_XL, UD-Q5_K_XL). |
| `STRATA_ORCA_Q4KS_MMQ` | OFF | The GGML Q5_0 MMQ instance for OrcaRouter Q4_K_S. |
| `STRATA_MMQ_TAB` | OFF | The pointer-table prompt MMQ (CUDA; ~37 MiB VRAM per GPU). |
| `STRATA_NATIVE_EXPERTS` | ON | Native (IQ) expert support through ggml-cpu. |
| `STRATA_BUILD_TESTS` | on when `tests/CMakeLists.txt` and `bench/micro` both exist | Pass it explicitly when `bench/micro` is absent. |
| `STRATA_BUILD_CONVERSATION_TESTS` | OFF | The standalone conversation cache tests. |
| `STRATA_ISA_FLOOR` | `""` | `""` / `avx` / `none`. |
| `STRATA_PORTABLE` | OFF | AVX2 baseline, static MSVC runtime. |
| `STRATA_WERROR` | OFF | Treat compiler warnings as errors. |
| `STRATA_ORACLE_SOURCE_DIR` / `STRATA_ORACLE_BUILD_DIR` | `""` | An external llama.cpp checkout for the independent parity tests. |

## The targets

`strata` (the first engine), `strata-poly` (the poly engine — **CUDA branch only**), `strata-gguf`,
`strata-dequant`, `strata-plan`, `strata-requant`, `strata-device`, `strata-vision` (the picture child, built
from llama.cpp's image code), and the parity family.

## The parity family

`src/kernels/*_parity.cpp` holds the instruments; `src/poly/*_test.cu` and `src/poly/*_test.cpp` hold the poly
engine's own. Each is a CMake target of the same name as its file.

**A test is listed only if it links.** The Vulkan branch's list grows only with ported kernels — adding a test
against a stub would fail, or worse, pass against a stub and claim coverage that does not exist.

## A link is the only honest check

Archives are pulled on demand, so a small target linking proves nothing about the full symbol surface.

```bash
cmake --build build --target strata-poly -j"$(nproc)"
```

If the build claims success **without relinking**, suspect a stale zero-filled output from an interrupted link
and delete it.

## The installer

`setup.py` holds `FAMILIES` — the models the installer knows, each with its repository, file name, size and
SHA-256, its routed experts' size and the VRAM it takes before any expert. `OWN_PATH_FAMILIES` is for a user's
own GGUF.

`setup.py` writes a `strata-<name>.json` beside it: the engine's executable, its args, its cwd, the tokenizer
directory, the model's name and the port.

## The server

`polystrata.py` is the user's entry point: `--list-models`, `--model <name>`, `benchmark --prompt-tokens N`.
`serve/` is the Python server; `serve/server.py` is where the routes are.

## The llama.cpp checkout

`STRATA_GGML_DIR` names a llama.cpp checkout to take ggml from; empty means fetch the pinned commit.
`third_party/ggml/` holds the vendored `ggml-common.h` (the i-quant codebook tables) and is **not** a
submodule.

> **The Vulkan work brings a `third_party/llama.cpp` submodule** — see `docs/VULKAN.md`. The ggml-vulkan
> interop accessors are a **patch** (`patches/llama.cpp-vulkan-interop.patch`), applied at configure time next
> to the qwen3-coder parser patch, so a clone of the pin builds them.

## Warnings

`STRATA_WERROR` turns compiler warnings into errors. There is **no** `.clang-format`, `.clang-tidy` or
`.editorconfig` in this tree, and no CI — the warning flags are the whole of the lint story.

## The AMD cards

`docs/AMD_HIP.md` is the reference. The poly models that have been run on AMD: Qwen3.6 and Ornith on an RX 7600
under Linux, Tiel Coder on an RX 9070 XT under Windows. **Nothing else on either.**
