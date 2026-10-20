---
description: Bring PolyStrata's documents in line with the current tree
---

Keep PolyStrata's documents true.

**Scope:** `$1` — a path or a topic to focus on (default: the whole tree).

## The style rule

The project's own words, from `AGENTS.md`:

> **Keep the docs' style: plain words, measured numbers with what they were measured on, no claims without a
> measurement.**

A number carries its conditions: the card, the CPU, the RAM, the context, the number of runs, and whether the
output was correct.

**No claims without a measurement** cuts both ways: do not write a performance claim that was not measured, and
do not write a limitation that was not found.

## The same-change rule

**A change that alters behaviour a document describes updates that document in the same change.** The docs are
not a follow-up task.

| Change | Document |
| --- | --- |
| A new model or architecture | `docs/BACKENDS.md` §"What runs" and §"Adding an architecture" |
| A new backend | `docs/BACKENDS.md` §"How a backend works" |
| A new card, or a card's coverage changing | `docs/BACKENDS.md` §"Limits", and `docs/AMD_HIP.md` / `docs/VULKAN.md` |
| A measured number | the table it belongs to, with its conditions |
| The installer | `docs/AI_SETUP.md`, and `setup.py`'s own `--help` |
| A user's run | `docs/COMMUNITY_BENCHMARKS.md` |

## The procedure

1. **Read the change** — `git status --short`, `git diff`, and the recent commits. The scope is what the tree now
   says, not what the last message claims.
2. **Find the affected claims.** Search the docs for the behaviour that changed: a new command, a changed
   default, a new model, a new limit, a new measurement.
3. **Verify every `file:line` citation before repeating it.** Line numbers drift; a stale citation is worse than
   none because it looks authoritative.
   ```bash
   grep -n "load_geometry" src/poly/engine.hpp
   ```
4. **Keep the markers true.** The docs distinguish what runs now from what is planned; never leave a statement
   reading as present tense when the code has moved on.
5. **Keep the setup's own tests passing** — `tools/test_setup_<name>.py` run without a GPU or downloads.

## The tree is the truth

When a document and the code disagree, the code is right and the document is a bug. Fix the document.

## Report

List each document you changed and what you changed in it, with the old and the new wording. List any claim you
could not verify, and any document you believe is stale but did not change.

**Do not silently rewrite prose.** If a claim is wrong but its correction is a judgement call, say so and ask.
