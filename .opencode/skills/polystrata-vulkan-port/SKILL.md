---
name: PolyStrata Vulkan Port
description: Port a CUDA kernel to the Vulkan backend of PolyStrata - the seam that maps the cuda* API onto src/vulkan/runtime.cpp, the two-edit shader rule, the ordering model, the host-node rules and the die-by-name stubs. Use when adding a shader, adding a host wrapper, replacing a stub, or debugging a Vulkan-only kernel.
---

# The Vulkan backend

PolyStrata is gaining a Vulkan backend: the port that lets the engines run on an AMD card through Vulkan, rather
than through HIP, and that reaches the cards the HIP build does not.

> **This backend does not exist in the tree yet.** It arrives with the merge described in `docs/VULKAN.md`. This
> skill describes the conventions it will follow, so that a kernel written today lands in it correctly.

## The seam

| Piece | Where |
| --- | --- |
| The CUDA runtime, mapped onto Vulkan | `include/strata/vulkan/shim/cuda_runtime.h` → `include/strata/vulkan/cuda_compat.hpp` → `src/vulkan/runtime.cpp` |
| The whole `cuda*` surface | `include/strata/vulkan/runtime.hpp` (~550 lines: streams, events, graph capture, host funcs, `cudaMemGetInfo`, `adopt_device`, `wrap_buffer`) |
| The kernel registry (descriptor pools) | `include/strata/vulkan/kernels.hpp`, `src/vulkan/kernels.cpp` |
| One-time shader registration | `src/vulkan/shaders.cpp` (`ensure_kernels_ready()`) |
| One host wrapper per CUDA kernel file | `src/kernels/vulkan/<name>.cpp` — same signature, same header as the `.cu` it replaces |
| The shaders | `shaders/*.comp` (+ the shared `shaders/*.glsl` includes) |
| The shader registry | `src/vulkan/CMakeLists.txt` → `SHADER_SPECS` |
| The die-on-name stubs | `src/kernels/vulkan/stubs_exact.cpp` |
| Availability gates | `src/kernels/vulkan/backend_gates.cpp` |

**Engine sources are textually unchanged between backends.** That is the point of the seam: the same
`src/core/layer.cpp` runs under CUDA and under Vulkan.

## The two-edit rule

A shader needs **two** edits, or it is never compiled:

1. The `.comp` file in `shaders/`. The file stem is the **registry kernel name** the host wrapper looks up.
2. A line in `SHADER_SPECS` in `src/vulkan/CMakeLists.txt`:

```
name.comp|bindings|push_bytes|local_x|local_y|local_z
```

- `bindings` — the number of storage-buffer bindings.
- `push_bytes` — the push-constant block size (Vulkan guarantees 128 max).
- `local_x|local_y|local_z` — the workgroup size the shader declares.

The list is **whitespace-separated and must contain NO COMMAS** — a comma is a configure error at the wrong line.
A **seventh** field specialises the shader to one weight format (`-DSPEC_TYPE=N`); the variant is registered as
`<stem>_t<N>`.

**Third edit:** if you add or change a shared `.glsl` include, add it to `SHADER_INCLUDES`. Without that, the
shaders that include it are **not rebuilt** — a change to a shared include silently does nothing.

## fp16 is converted as bits

**Never through `unpack/packHalf2x16`.** This driver flushes subnormal fp32→fp16 results to zero, and a scale
that goes subnormal becomes every code zero. Use the sanctioned forms: `f16_val` (the i-quant dequantizer),
`f16e` (the expert one), `f16b` (the q8_1 block), `q81_scale`.

`fma()` lowers to a separate mul+add in this driver. Do not assume fused semantics.

## The ordering rules

- **The null stream is real.** The runtime maps the engine's `nullptr` stream to a stream of its own. Canonical
  CUDA kernels accept a null stream and synchronize after (`sync_if_default()` mirrors that). **Do not add
  `!stream` argument checks** to canonical wrappers — a null stream is a real stream here, and the check that was
  added once made every parity test return early: green, and having run nothing.
- **Vulkan does not order your commands; CUDA does.** `stream_cmd()` emits a full pipeline barrier before every
  recorded operation. A dispatch followed by a transfer in one command buffer has **no implicit dependency**.
  The engine is written against CUDA and assumes ordering everywhere.
  *Symptom to recognise: a kernel that is provably correct reporting stale or previous output is an ordering
  bug, not a math bug.* An explicit `cudaDeviceSynchronize` before the readback will appear to fix it — do not
  accept that as a fix, and do not go looking for a shader error.
- **A command buffer is REUSED, never allocated per submission.** Allocating a fresh one per flush retained
  ~47 KB of host memory per iteration that the pool reset never gave back.
- **A captured graph's segments come from the GLOBAL command pool**, never the stream's transient one. A reset of
  the transient pool destroys commands a graph still replays.
- **A descriptor pool a graph can reference is PINNED at capture end.** `pin_live_pools()` records every pool live
  during a capture; a graph references those sets on every replay.
- **Every helper called from inside a critical section must be the `_locked` form** —
  `drain_inflight_locked`, `flush_stream_locked`, `take_graph_workers_locked`. A stream-keyed call from the drain
  self-deadlocks on a non-recursive mutex.
- **`cudaMemset` writes BYTES**, and its range is byte-exact at any offset and size.
- **`cudaMemGetInfo` is the DEVICE_LOCAL heap(s)**, never the sum of every heap. A discrete GPU also lists its
  shared system memory as `HOST_VISIBLE`; summing over-reports badly.

## The poly engine is not yet a Vulkan citizen

`src/poly/` is **raw CUDA**: 79 `__global__` kernels in `src/poly/kernels.cu`, `kernels_dsv4.cu` and `fattn.cu`,
plus a **cuBLAS dependency** for the batched GEMMs (`lin()` multiplies a prompt batch by cuBLAS; `attend_batch`
uses `blas_`).

The seam maps the `cuda*` **API** onto Vulkan; it does not reach raw kernels. So the poly engine needs its own
work: the kernels become recorded dispatches, and cuBLAS needs a Vulkan replacement.

Two things make that more tractable than it sounds:

- **`Engine`'s hook table is the seam.** `load_geometry`, `load_layer`, `layer_head`, `route`, `moe_shared`,
  `output_rows` and the rest are called once a layer or once a call — so a Vulkan backend is a **subclass**, not
  a patch through a monolith.
- **The kernel bodies already exist in the first engine's Vulkan wrappers.** The dense projections, the attention,
  the router, the expert GEMMs, the norms, the rotary embedding and the SwiGLU are all ported there. The poly
  engine should call those rather than translate `src/poly/kernels.cu` line by line.

**`fattn_mma.cuh` is dead weight on an AMD card** — the tensor cores' `m16n8k16` step needs Turing, and AMD
builds have neither. The scalar path is the one that runs.

## Never fabricate

Never accept a plausible return value for a path that is not implemented. A decode that returns fabricated
expert layouts produces fluent wrong text, which is worse than stopping. Die by name — `stubs_exact.cpp` is
exactly that, and a stub's signature must be **exact**: an ellipsis stub mangles as `...z`, satisfies nothing, and
fails the link far from its cause.
