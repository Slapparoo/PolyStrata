# The Vulkan backend

PolyStrata is gaining a third backend. This page says what it is, where the code comes from, how it gets into
this tree, what has to happen afterwards, and what is not measured yet.

The goal is the models of `strata-poly` — GLM 5.3 Flash, Qwen3.6-35B-A3B, Ornith 1.5, Gemma 4, Tiel Coder,
MiniCPM-V, DeepSeek V4 Flash and LFM2.5-24B-A2B — on the AMD cards the HIP build does not reach, and on the
cards the HIP build reaches without their own build of the poly engine.

## What exists today

Two backends, and a third to come.

| Backend | How | What it runs |
| --- | --- | --- |
| CUDA | `STRATA_ENABLE_CUDA=ON` | everything |
| HIP | `STRATA_ENABLE_HIP=ON` (wave32 RDNA) or `STRATA_HIP_GFX906=ON` (wave64) | `strata` and some of `strata-poly`: Qwen3.6 and Ornith on an RX 7600 under Linux, Tiel Coder on an RX 9070 XT under Windows |
| Vulkan | `STRATA_ENABLE_VULKAN=ON` | nothing yet |

[BACKENDS.md](BACKENDS.md) §"Limits" has the card list, and it is the place this page's outcome has to be
written down.

## Where the code comes from

The Vulkan backend exists in another project, **UnbiasedStrata** (`Slapparoo/UnbiasedStrata`), which is this
tree's ancestor with the Vulkan work on top. It is a port of the first engine's CUDA kernels to Vulkan, done by
replacing each kernel's `.cu` with a host wrapper that records the work into a Vulkan command buffer, and by
compiling the kernels' arithmetic again as GLSL compute shaders.

What it is, counted on 2026-10-21:

| | Files | Lines |
| --- | ---: | ---: |
| `src/vulkan/` — the runtime that the `cuda*` API maps onto | 10 | 6,167 |
| `src/kernels/vulkan/` — one host wrapper per CUDA kernel file | 31 | 4,398 |
| `include/strata/vulkan/` — the `cuda*` surface, the registry, the shims | 5 | 813 |
| `shaders/` — 79 compute shaders and 4 shared GLSL includes | 83 | 5,176 |
| **Total** | **129** | **16,554** |

It replaces 45 CUDA kernel files (20,495 lines) with 31 host wrappers. The engine's own sources are **textually
unchanged** between backends: the same `src/core/layer.cpp` runs under CUDA and under Vulkan, because the seam
is the runtime, not the engine.

Both projects are MIT-licensed with the same copyright line, so the code moves freely.

## The submodule: a pin plus a patch

The port needs `third_party/llama.cpp` as a submodule, and the accessors that hand a Vulkan buffer to ggml —
the tensor's VkBuffer, the device handles, a buffer over a foreign VkBuffer.

Those were three commits on a **local branch** of the submodule, which made the tree single-machine: a clone
that could not fetch that branch would not build. **They are a patch now** —
`patches/llama.cpp-vulkan-interop.patch`, 159 lines over three files — and the build applies it at configure
time next to the qwen3-coder parser patch, with the same reverse dry-run so an already-applied patch is
harmless.

**A clone of the pin builds them.** Checked on 2026-10-21: a pristine checkout of the pinned commit, both
patches applied at configure, prints `interop accessor present`, and the interop spike target builds. The patch
reproduces that branch's tree exactly — the whole-tree hash of the pin plus the patch matches the branch's.

The gitlink still points at the branch, so `git submodule update` wants a commit it cannot fetch. **The pin plus
the patch is the portable form**, and it is what comes over with the merge.

This tree vendors only `third_party/ggml/ggml-common.h` — the i-quant codebook tables — and that stays.

## The merge

The two trees have **unrelated git histories**: neither contains the other's import of upstream Strata. They do
share a **content base** — upstream Strata at `6f32ec0` — so the merge is done with a **synthetic common
ancestor**: a commit whose tree is upstream's, with the two sides as its two children. Git then does an
ordinary three-way merge with the right base.

Measured on the trees as of 2026-10-21, on Linux, with `git merge --no-commit`:

| | |
| --- | ---: |
| Conflicted files | 225 |
| Conflict hunks | 221 |
| — trivial: `docs/`, the Python `serve/`, `tools/`, `sycl/`, the READMEs | 191 |
| — **substantive: `src/`, `include/strata/`, `CMakeLists.txt`** | **34 files, about 180 hunks** |
| **Conflicts inside the Vulkan port itself** | **0** |

The port's files are new on one side only, so they transplant clean. The work is in the files **both** sides
changed.

The substantive conflicts fall in four files, and the rest are short:

| File | Hunks |
| --- | ---: |
| `src/program/generate.cpp` | 41 |
| `src/prefill/prefill.cpp` | 23 |
| `src/core/verify.cpp` | 23 |
| `src/kernels/cuda/qsa_prompt_attn.cu` | 12 |
| `src/core/mtp.cpp` | 11 |
| `CMakeLists.txt` | 10 |
| `src/kernels/cuda/qsa_decode_attn.cu` | 9 |
| `src/core/layer.cpp` | 4 |
| `src/core/expert_source.cpp` | 3 |
| `src/core/session.cpp` | 1 |

Two of the sides made different, non-overlapping work on the same CUDA kernels, and that is where the hunks
are. The other side templated `native_mmvq.cu` for the cards (a `SmallTraits` shape, a per-architecture tuning
path for Pascal, Volta and MXFP4); this side made the integer weight types run as INT8 dot products instead of
float dequantization. Both are wanted.

**Resolving a CUDA kernel's conflict in favour of one side can leave that side's Vulkan wrapper stale** — the
wrapper encodes the other side's arithmetic. Every such pair has to be re-checked after the merge.

## After the merge

**The merge is not done when it compiles.** The backend has to be re-validated on the card:

- the device probe and the memory plan, which must agree with each other;
- the parity family — 45 instruments in this tree, plus the poly engine's own thirteen;
- a decode that produces the right text, which is the only check that says the whole thing works.

The traps that cost time once are in the port's own notes, and they are worth reading before the first run:

- **the null stream is real** — the runtime maps `nullptr` to a stream of its own, and a `!stream` check in a
  canonical wrapper once made every parity test return early: green, and having run nothing;
- **Vulkan does not order your commands** — a dispatch followed by a transfer has no implicit dependency, and a
  kernel that is provably correct reporting stale output is an ordering bug, not a math bug;
- **a command buffer is reused, never allocated per submission**;
- **a captured graph's segments come from the global command pool**, not the stream's transient one;
- **the descriptors a graph can reference are pinned at capture end**.

## The poly engine on Vulkan

This is the larger half, and it is not done by the merge.

`src/poly/` is **raw CUDA**: 79 `__global__` kernels in `src/poly/kernels.cu`, `src/poly/kernels_dsv4.cu` and
`src/poly/fattn.cu`, and a **cuBLAS dependency** — `lin()` multiplies a prompt batch by cuBLAS, and
`attend_batch` uses the handle `blas_`.

The seam maps the `cuda*` **API** onto the runtime. It does not reach raw kernels. So the poly engine needs its
own work: the kernels become recorded dispatches, and cuBLAS needs a Vulkan replacement.

Two things make that more tractable than the kernel count suggests.

**`Engine`'s hook table is the seam.** The hooks `load_geometry`, `load_layer`, `layer_head`, `route`,
`moe_shared` and `output_rows` are called once a layer or once a call, never inside a kernel, so the calls cost
nothing that can be measured. A Vulkan backend is a **subclass**, not a change threaded through a monolith.

**The kernel bodies already exist.** The first engine's port already has the dense projections, the attention,
the router, the expert GEMMs, the norms, the rotary embedding and the SwiGLU. The poly engine should call those
rather than translate `src/poly/kernels.cu` line by line.

And one path is dead weight on an AMD card: `src/poly/fattn_mma.cuh` is 1,940 lines of tensor-core intrinsics, and
the `m16n8k16` step needs Turing. The scalar path is the one that runs.

## The order: LFM2.5 first

One architecture, and the smallest of them — 24B total with 2B active. Its layer is the simplest too: a gated
short convolution of three taps in 30 of the 40 layers, attention in the other 10, a dense FFN in the first two,
routed experts with **sigmoid** gates and a **selection bias** in the rest, and **no shared expert**.

Each step has to be checkable on its own, against a reference that is already in the tree:

1. the host side — a subclass of `Engine` that opens the GGUF and reads its geometry;
2. the dense path — the projections, and the cuBLAS replacement;
3. the recurrent path — the short convolution and its state, against `src/poly/shortconv_test.cu`;
4. the attention — the normed and turned queries and keys, and the caches;
5. the router — the sigmoid gates, the bias added after the sigmoid and used only for the selection, the weight
   from the unbiased gate, and the renormalisation, against `src/poly/route_test.cu`;
6. the experts, and then the whole model.

The oracle for the whole is llama.cpp: **the same prompt's token ids into both, greedy, and the tokens
compared.** Then probabilities, on short and long prompts, with every expert on the CPU, and with and without
guessed tokens.

The numbers to beat are in [LFM.md](LFM.md), each with the card it was measured on. On a laptop with 8 GB of
VRAM (RTX 4070 Laptop, i7-14700HX, 32 GB of RAM, Windows 10, CUDA 13.0), UD-Q4_K_XL gave 8.3 to 8.4 s to the
first token, 3,575 to 3,601 tokens/s reading a 29,804-token prompt, and 52% to 53% of the experts from VRAM.

## The license

LFM2.5-24B-A2B is Liquid AI's LFM Open License v1.0, not an open-source one: commercial use is licensed only to
companies under 10 million US dollars of yearly revenue. [LFM.md](LFM.md) says so, and any page that offers the
model says so.

## What is not measured yet

Everything above about the merge is measured, on the trees of 2026-10-21, on Linux. Everything about the poly
engine on Vulkan is **not**: no kernel of it has been written, no number has been taken, and no card has run it.

The claims this page makes are the ones the measurements carry. The rest is the plan.
