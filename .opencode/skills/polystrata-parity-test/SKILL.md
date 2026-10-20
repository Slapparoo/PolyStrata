---
name: PolyStrata Parity Test
description: Write, extend or audit an instrument for PolyStrata - the kernel parity tests in src/kernels/*_parity.cpp and the poly engine's own in src/poly/*_test.cu. Use when a kernel needs a reference to judge it, when a test fails, when choosing a tolerance, or when registering a test in CMakeLists.txt.
---

# The instruments

A kernel change is **not done** until an instrument judges it against a reference, with **negative controls** for
the wrong readings it guards against.

Every parity binary:

- takes `--selftest`;
- prints its checks;
- exits **0** on success.

The poly engine's own tests take the same contract: `poly_shortconv_test` is "exit 0: everything within a few
units in the last place".

## Where the files live

**`src/kernels/<name>_parity.cpp`** — the kernel instruments (54 of them). The target name is the file stem.

**`src/poly/<name>_test.cu`** — the poly engine's own, thirteen of them: `shortconv_test`, `route_test`,
`attend_test`, `fattn_test`, `dsv4_test`, `kv_q8_test`, `mxfp4_test`, `norm_q8_test`, `gemm_small_test`,
`draft_race_test`, `guesses_test`, `workers_test`, `conversations_test`. Plus `src/poly/expert_parity.cpp`.

A test that needs the poly engine's headers adds
`target_include_directories(... ${CMAKE_CURRENT_SOURCE_DIR}/include ${CMAKE_CURRENT_SOURCE_DIR}/src/poly)`.

## The oracle rule

> **Check it against llama.cpp before trusting it: the same prompt's token ids into both, greedy, and compare
> the tokens that come out.**

For a kernel, the reference is the arithmetic the CUDA kernel does — the test carries a **host implementation of
the same arithmetic** in the same file, so a mismatch is the kernel's, not the reference's.

`docs/BACKENDS.md` §"Adding an architecture" step 5 is the model: tokens and probabilities on short and long
prompts, with every expert on the CPU, and with and without guessed tokens.

## The four ways an instrument lies

### 1. It returns early

A test that printed nothing has not proven anything. **Print what ran** — the case count, the shape, the format
— unconditionally, before the verdict.

### 2. The property under test is not observable

If two things the test compares were indistinguishable, every green number would be vacuous. Close it with an
explicit **observability assertion**: perturb the thing and require the check to notice.

`src/poly/shortconv_test.cu` is the model here: it checks the outputs, **the state it leaves**, and the
snapshots of a window's tokens, and it runs one long call against the same tokens in two calls.

### 3. The tolerance is wrong for the contract

| Quantity | Tolerance |
| --- | --- |
| Bitwise contracts (codes, scales, permutations, exact int sums) | **bit-exact** |
| Summation-order differences (fp32 reductions) | a relative bound, and the test says which |
| A scale that is fp16 on both sides | **one fp16 ulp**, never bitwise |

A loose tolerance over many elements will accept a **permutation**. A permutation, a scale and an off-by-one each
need their own check.

### 4. The negative control is missing

**Require:** a deliberately wrong input the check must reject, per reading guarded. If nothing would make the
check fail, it is not a check.

## Also audit

- **The reference's provenance.** Is the host reference a transcription of the real source, or something
  written from the implementation's output? A reference transcribed from the thing it judges is circular.
- **Separation of the checks.** One end-to-end comparison needs one loose tolerance for everything and hides the
  interesting failures.
- **The registration.** A test is listed **only if it links**. Is the target in `CMakeLists.txt`, and does what
  it exercises actually exist there (not a stub)?
- **Unbuffered stdout.** Does `main` call `setvbuf(stdout, nullptr, _IONBF, 0)`? A crashing test prints nothing
  otherwise, and "empty output with a non-zero exit" then reads as "printed nothing" rather than "crashed".
- **The measurement's conditions.** A number a test prints is a measurement: say what it was measured on.

## Reading a red test

A red test has so far meant a **runtime or layout bug**, not a kernel that is difficult to port. Read the
failure shape before reaching for the shader:

- **All variants agree with each other but not with the reference** → a layout or convention disagreement.
- **Stale or previous output** → an ordering bug (Vulkan does not order your commands; CUDA does).
- **A number that is absurdly large** → a fabricated or uninitialised layout.

## The exit codes

| Code | Meaning |
| --- | --- |
| `0` | pass |
| `1` | a check failed |
| `2` | bad usage, or a die-by-name stub was reached |
| `77` | skip — no device, no fixture, no pack |
| `3` | skip — a dependency is missing |

A skip is **not** a pass. Say which arm did not run and why.
