---
description: Spike one poly architecture on the Vulkan runtime - LFM2.5 first
---

Spike a poly architecture on the Vulkan runtime.

**Architecture:** `$1` — default `lfm2moe` (LFM2.5-24B-A2B). `qwen35moe`, `glm5-next`, `gemma4` and `deepseek4`
are the others.

## Why this is the shape of the work

`src/poly/` is **raw CUDA**: 79 `__global__` kernels in `src/poly/kernels.cu`, `kernels_dsv4.cu` and `fattn.cu`,
plus a **cuBLAS dependency** for the batched GEMMs — `lin()` multiplies a prompt batch by cuBLAS, and
`attend_batch` uses `blas_`.

The Vulkan seam maps the `cuda*` **API** onto `src/vulkan/runtime.cpp`. It does not reach raw kernels. So the
poly engine needs its own work: the kernels become recorded dispatches, and cuBLAS needs a Vulkan replacement.

Two things make that more tractable than it sounds:

- **`Engine`'s hook table is the seam.** `load_geometry`, `load_layer`, `layer_head`, `route`, `moe_shared`,
  `output_rows` and the rest are called once a layer or once a call, never inside a kernel. A Vulkan backend is
  a **subclass**, not a patch through a monolith.
- **The kernel bodies already exist in the first engine's Vulkan wrappers.** The dense projections, the
  attention, the router, the expert GEMMs, the norms, the rotary embedding and the SwiGLU are all ported there.
  The poly engine should call those rather than translate `src/poly/kernels.cu` line by line.

**`fattn_mma.cuh` is dead weight on an AMD card** — the tensor cores' `m16n8k16` step needs Turing, and AMD
builds have neither. The scalar path is the one that runs.

## The spike

Do it on **one** architecture. LFM2.5 is the smallest of them (24B total, 2B active) and the one with the
simplest layer: a gated short convolution of three taps in 30 of 40 layers, attention in the other 10, a dense
FFN in the first two, routed experts with **sigmoid** gates and a **selection bias** in the rest, and **no shared
expert**.

### The order that keeps each step checkable

1. **The host side first.** A subclass of `Engine` that opens the GGUF and reports its geometry, with the
   kernels stubbed. If that does not open the file and read the shapes, nothing after it matters.
2. **The dense path.** The projections (`lin()`) — the cuBLAS replacement. Check it against llama.cpp's
   logits on one token.
3. **The recurrent path.** The short convolution, with its state. `src/poly/shortconv_test.cu` is the reference
   for the arithmetic — the outputs, **the state it leaves**, and the snapshots of a window's tokens.
4. **The attention.** The queries and keys normed a head and turned by the position, the keys and values into
   the caches.
5. **The router.** Sigmoid gates, the bias added **after** the sigmoid and used **only for the selection**, the
   weight from the **unbiased** gate, then the renormalisation. `src/poly/route_test.cu` is the reference.
6. **The experts**, and then the whole model.

### The oracle

> **Check it against llama.cpp on the same file before any tuning: the same prompt's token ids into both, greedy,
> and compare the tokens that come out.**

Then probabilities, on short and long prompts, with every expert on the CPU, and with and without guessed
tokens.

### The measurements

`docs/LFM.md` has the numbers to beat, each with the card it was measured on: on a laptop with 8 GB of VRAM
(RTX 4070 Laptop, i7-14700HX, 32 GB RAM, Windows 10, CUDA 13.0), UD-Q4_K_XL gave 8.3–8.4 s to the first token,
3,575–3,601 tokens/s reading a 29,804-token prompt, and 52–53% of the experts from VRAM.

**A number without its conditions is not a measurement.** Say the card, the CPU, the RAM, the context, the runs,
and whether the output was correct.

## Report

What was built, what was checked against what, the measured numbers with their conditions, and what is still
missing. A spike that half works is worth reporting as a half — with the exact step it stopped at.
