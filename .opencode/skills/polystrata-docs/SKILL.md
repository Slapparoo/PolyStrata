---
name: PolyStrata Docs
description: Keep PolyStrata's documents true - which document owns which fact, the style rule the docs follow, and the same-change rule for keeping docs/BACKENDS.md, docs/LFM.md, docs/AMD_HIP.md and the other pages in line with the tree. Use when a change makes a document stale, before stating a measured number, or when adding a model, a backend or an architecture.
---

# The documents

PolyStrata's docs are the project's memory. `AGENTS.md` is a short pointer to them; the pages under `docs/` carry
the detail.

| Document | Owns |
| --- | --- |
| `docs/BACKENDS.md` | Which model runs on which engine; how a backend works; how a model is recognized; the registry; how to add a model and how to add an architecture; the limits; every benchmark. |
| `docs/DETAILS.md` | How the engine works, every measured number, the API, all the settings. |
| `docs/LFM.md` | LFM2.5-24B-A2B: the sizes, the speed on two cards, its license. |
| `docs/GLM.md`, `docs/DEEPSEEK.md` | The two large poly models, in their own pages. |
| `docs/AMD_HIP.md`, `docs/AMD_HIP_PERFORMANCE.md` | The AMD cards, the HIP build and its validation. |
| `docs/INTEL_ARC.md` | The SYCL engine. |
| `docs/AI_SETUP.md`, `docs/INSTALL.md` | Installing for a user. |
| `docs/MCP_SERVER.md` | The MCP server. |
| `docs/COMMUNITY_BENCHMARKS.md` | Runs from other people's machines. |
| `AGENTS.md` | Installing for a user; working on the code; contributing a change. |
| `README.md` (+ the translations) | The front page. |

## The style rule

The project's own words, from `AGENTS.md`:

> **Keep the docs' style: plain words, measured numbers with what they were measured on, no claims without a
> measurement.**

Concretely, a number carries its conditions: the card, the CPU, the RAM, the context, the number of runs, and
whether the output was correct. Seven identical probes measuring 2.50–4.12 tok/s (mean 3.43, sd 0.65) is a
measurement; a single run is a sample.

**No claims without a measurement** cuts both ways: do not write a performance claim that was not measured, and
do not write a limitation that was not found.

## The same-change rule

**A change that alters behaviour a document describes updates that document in the same change.** The docs are
not a follow-up task.

- A new model or a new architecture → `docs/BACKENDS.md` §"What runs" and §"Adding an architecture", and the
  model's own page if it has one.
- A new backend → `docs/BACKENDS.md` §"How a backend works", and the Limits section if the card coverage changes.
- A measured number → the table it belongs to, with its conditions.
- A new limit → the Limits section, and the page of the thing it limits.
- A change to the installer → `docs/AI_SETUP.md`, and `setup.py`'s own `--help`.

## Verify a citation before quoting it

A `file:line` reference must be checked against the current tree before it is written down. Line numbers drift,
and a stale citation is worse than none because it looks authoritative.

```bash
grep -n "load_geometry" src/poly/engine.hpp
```

## Commit messages

Lowercase, imperative, and prefixed with the area: `poly:`, `setup:`, `docs:`, `bench:`, `tools:`. The subject
says what changed; the body explains why, and carries the measurement if there is one.

Examples from the tree:

```
setup: refuse an own GGUF whose experts' gate and up differ in type
poly: --check-batch gives the attention caches their VRAM at load
docs: a user's Tiel Coder UD-Q4_K_XL runs on an RTX 4070 Ti SUPER
```

## The tree is the truth

When a document and the code disagree, the code is right and the document is a bug. Fix the document.

## The setup's own tests

`tools/test_setup_<name>.py` run without a GPU or downloads — for example `tools/test_setup_amd.py`,
`tools/test_setup_choices.py`. A change to `setup.py` should keep them passing.

## The license note

A model's license is part of its page. **LFM2.5-24B-A2B is Liquid AI's LFM Open License v1.0, not an open-source
one**: commercial use is licensed only to companies under 10 million US dollars of yearly revenue. Any page that
offers the model says so.
