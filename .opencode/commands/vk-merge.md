---
description: Run the 3-way merge of the Vulkan port into this tree and report the conflicts that need a decision
---

Merge the Vulkan port in and report what needs a decision.

**Source of the port:** the `UnbiasedStrata` repository (the Vulkan backend), on the branch that holds it.
**Destination:** this tree, on the current branch.

## Why a 3-way merge and not a patch

The two trees have **unrelated git histories** — neither contains the other's import commit — so `git merge`
would refuse. But they share a **content base**: upstream Strata at `6f32ec0`, which is the first commit of the
Vulkan tree's history.

So the merge is done with a **synthetic common ancestor**: a commit whose tree is upstream's, with the two sides
as its two children. Git then does an ordinary 3-way merge with the right base.

```bash
# in a scratch clone of this tree, with the other tree fetched as a remote
BASE_TREE=$(git rev-parse <upstream-import-commit>^{tree})
BASEC=$(git commit-tree "$BASE_TREE" -m "synthetic base: upstream Strata 6f32ec0")
UNC=$(git rev-parse main^{tree});  POLYC=$(git rev-parse poly/main^{tree})
UNB=$(git commit-tree "$UNC" -p "$BASEC" -m "UnbiasedStrata side (vulkan port)")
POL=$(git commit-tree "$POLYC" -p "$BASEC" -m "PolyStrata side (poly engine)")
git update-ref refs/heads/xpoly "$POL"; git update-ref refs/heads/xunb "$UNB"
git checkout xunb && git merge --no-commit --no-ff xpoly
```

**Do this in a scratch clone**, never in the working tree: the merge leaves the index full of conflicts.

## What the measurement says

The merge was run once already, on the trees as of 2026-10-21. **Re-measure it** — the numbers move with every
commit — but this is the shape to expect:

| | |
| --- | --- |
| Conflicted files | 225 |
| Conflict hunks | 221 |
| Trivial (`docs/`, `serve/` Python, `tools/`, `sycl/`, the READMEs) | 191 |
| **Substantive (`src/`, `include/strata/`, `CMakeLists.txt`)** | **34 files, ~180 hunks** |
| **Conflicts inside the Vulkan port itself** | **0** |

The Vulkan port's files — `src/vulkan/`, `src/kernels/vulkan/`, `shaders/`, `include/strata/vulkan/` — are new
on one side only, so they transplant clean. The work is in the files **both** sides changed.

The substantive conflicts concentrate hard. Expect roughly: `src/program/generate.cpp` 41 hunks,
`src/prefill/prefill.cpp` 23, `src/core/verify.cpp` 23, `src/kernels/cuda/qsa_prompt_attn.cu` 12,
`src/core/mtp.cpp` 11, `CMakeLists.txt` 10, `src/kernels/cuda/qsa_decode_attn.cu` 9 — and `src/core/layer.cpp`
only 4.

## The order to resolve in

1. **Trivial conflicts first**, to get them out of the way: take this tree's version for the files this fork
   removed (the Python `serve/`, `tools/vision/`, `sycl/`), and PolyStrata's for the ones it owns.
2. **`CMakeLists.txt`** — the two branches are exclusive (`STRATA_ENABLE_VULKAN` against `STRATA_ENABLE_SYCL`),
   so the merged file has to carry both.
3. **The CUDA kernels the Vulkan wrappers mirror.** When a conflict is resolved in favour of one side's `.cu`,
   **the matching `src/kernels/vulkan/<name>.cpp` may be stale** — it encodes the other side's semantics. List
   every one of those at the end and re-check each against its kernel.
4. **The four big files** (`generate.cpp`, `prefill.cpp`, `verify.cpp`, `qsa_prompt_attn.cu`) last, once the
   shape of the tree is settled.

## The prerequisite

The Vulkan port needs `third_party/llama.cpp` as a submodule, on the `strata/ggml-interop` branch — three commits
past the pin, for the ggml-vulkan interop accessors. **That branch is local-only until it is pushed to a fork
that a clone can reach**, so `git submodule update` fails on it. Push it first.

The patch `patches/llama.cpp-qwen3-coder-any-argument-order.patch` is applied to the submodule's working tree at
configure time, and is deliberately not committed there.

## After the merge

**The merge is not done when it compiles.** The Vulkan backend has to be re-validated on the card: the device
probe, the memory plan, and the parity family. See `/vk-build` and `/vk-test`.

Report: the conflict count by category, the list of substantive files with their hunk counts, the
CUDA-kernel/Vulkan-wrapper pairs that need re-checking, and what the build says afterwards.
