---
name: PolyStrata Engine
description: Work on PolyStrata's inference engines - the shared Engine class in src/poly/engine.hpp, the architecture classes derived from it, the kernels in src/poly/kernels.cu, and the first engine in src/core. Use when adding an architecture, adding a backend, changing a hook, or when a poly model's output or speed is wrong.
---

# The engines

PolyStrata has two engines and one server.

| Engine | Source | What it runs |
| --- | --- | --- |
| `strata` | `src/core/`, `src/kernels/`, `src/prefill/` | Qwen3.8-Flash-Next and its fine-tunes. The first engine, from Strata. |
| `strata-poly` | `src/poly/` | GLM 5.3 Flash, Qwen3.6-35B-A3B, Ornith 1.5, Gemma 4, Tiel Coder, MiniCPM-V, DeepSeek V4 Flash, LFM2.5-24B-A2B. |

The server (`serve/`, Python) starts an engine as a child process and talks to it over lines of text: `GEN` with
token ids in, `T` lines with tokens and a `DONE` line with timings out. **Any program that speaks those lines is
a backend to the server.** Changing the model stops the child and starts another, so its VRAM, its pinned memory
and its file mappings go back to the operating system when the process ends — not through cleanup code that
could miss something.

## The `Engine` class — the seam everything hangs on

`src/poly/engine.hpp` (~755 lines) and `src/poly/engine.cpp` (~3,576 lines). **Inside `strata-poly`, `Engine` is
what every architecture shares:**

- the GGUF, read in place, in one file or several;
- the dense weights in VRAM in the type the file has them;
- the routed experts: a VRAM cache filled by how much each is used, experts the cache lacks computed on the CPU
  while the card does its share, and the cache following what an answer uses;
- a prompt batch: every token's experts on the card with llama.cpp's batched kernels, through a staging buffer
  for the experts that are not cached;
- the windows of guessed tokens and their check, the draft block that guesses, and guesses from repeats;
- the conversation state: checkpoints, parked conversations;
- CUDA graphs for a token's fixed launches;
- the serve protocol, sampling, statistics.

**An architecture is a class derived from `Engine`** — `Glm5Next`, `Qwen35`, `Gemma4`, `DeepSeek4`, `Lfm2Moe`.
It supplies these hooks:

| Hook | What it does |
| --- | --- |
| `load_geometry()` | the model's shape from the metadata; `load_shape()` reads what all have |
| `load_layer(il)` | a layer's weights to VRAM; `load_moe()` loads a MoE layer's router and finds its experts |
| `recurrent_floats()`, `alloc_caches()`, `assign_scratch()`, `widest()` | its state, its attention caches, its scratch |
| `layer_head()`, `layer_tail()` | a layer up to its router, and what happens to the experts' sum |
| `route()`, `moe_shared()`, `router_scores()` | the routing rule, the shared expert, the router's input |
| `output_rows()`, `finish_logits()`, `embed_scale()` | the output norm, a cap on the logits, a scale on embeddings |
| `block_attend()`, `block_rows()` | the draft block's attention |
| `sees()`, `whole()` | whether it reads pictures, and how many of a prompt's rows the next call takes |
| `row_bytes()`, `save_rows()`, `load_rows()`, `extra_bytes()`, `save_extra()`, `load_extra()` | what a conversation keeps |

A layer's own weights live in the class's own `Block`, one a layer, beside the shared `Layer`.

> **The engine calls a hook once a layer or once a call, never inside a kernel**, so the calls cost nothing that
> can be measured: GLM writes at the same speed before and after the split (51.8, 50.6 and 46.1 tokens/s against
> 50.7, 49.5 and 45.8).

**Nothing in the shared code asks which model it runs.** What differs between architectures is a hook, or a value
the architecture sets (`gelu_` for the experts' activation, the distance between experts in a fused tensor).

## The architecture files

| File | Lines | Architecture |
| --- | ---: | --- |
| `src/poly/glm5next.cpp` | 614 | GLM 5.3 Flash |
| `src/poly/gemma4.cpp` | 645 | Gemma 4 |
| `src/poly/deepseek4.cpp` | 605 | DeepSeek V4 Flash |
| `src/poly/qwen35.cpp` | 548 | Qwen3.6-35B-A3B, Ornith 1.5, Tiel Coder, MiniCPM-V |
| `src/poly/lfm2moe.cpp` | 349 | LFM2.5-24B-A2B |
| `src/poly/cpu_experts.cpp` | 234 | the CPU side of the experts |

`src/poly/kernels.cu` (~2,724 lines) holds the kernels; `src/poly/kernels_dsv4.cu` (~598) the DeepSeek ones;
`src/poly/fattn.cu` (~631) and `src/poly/fattn_mma.cuh` (~1,940) the fused attention, of which the
`fattn_mma` half is the tensor-core path.

## Adding an architecture

`docs/BACKENDS.md` §"Adding an architecture" is the recipe. In short:

1. **Read llama.cpp's graph for it** (`src/models/<arch>.cpp`) and the ggml ops it builds. That is the reference.
2. **Derive a class from `Engine`.** `qwen35.cpp` is the shortest example with both a recurrent layer and
   attention; `gemma4.cpp` shows the hooks for a model that departs further.
3. **Put what the layers need that no kernel does yet in `src/poly/kernels.cu`**, next to the others. Keep a
   kernel's result for a token independent of the tokens beside it when it runs in a window of guessed tokens.
4. **Name the class in `Engine::open()`** and the file in `CMakeLists.txt`.
5. **Check it against llama.cpp on the same file before any tuning**: tokens and probabilities on short and long
   prompts, with every expert on the CPU, and with and without guessed tokens.

## Adding a model of an architecture that is there

Also in `docs/BACKENDS.md`. The short version: check `general.architecture` is one the table names; write the
tokenizer directory with `tools/strata_tokenizer.py`; write `strata-<name>.json` beside `setup.py`; check
`polystrata.py --list-models`; **check it against llama.cpp before trusting it** — the same prompt's token ids
into both, greedy, and compare the tokens that come out.

## The first engine's families

The first engine (`src/core/`) has its own, table-driven family abstraction:

- `include/strata/artifact/model_family.hpp` — the `ModelFamily` enum, and `model_family_from_arch` reading
  `general.architecture`. **A recognised family the engine does not run yet is refused with
  `model_family_not_supported`, not mis-run.**
- `include/strata/core/family_layout.hpp` — `geometry_from_gguf`, and the `layer_tensors` / `global_tensors`
  layout tables: every tensor of a layer, by name and GGUF shape, checked against a model file's directory. For
  a family whose table is complete, no `blk.*` tensor the table does not name is allowed.
- `src/core/arch_defaults.cpp` — the per-family defaults.

## What to read first for a bug

- The architecture's own file (`src/poly/<arch>.cpp`) — a wrong answer in one model is usually there.
- `src/poly/engine.cpp` — the shared code. It is the same for every model, so a bug there shows in all of them.
- `src/poly/kernels.cu` — the kernels.

## The rule about numbers

`docs/` keeps measured numbers with **what they were measured on**: the card, the CPU, the RAM, the context, the
number of runs. A number without its conditions is not a measurement, and the docs do not carry one. The same
applies to a claim in a commit message: **no claims without a measurement.**
