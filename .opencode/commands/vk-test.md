---
description: Build, deploy and run the parity suite on the real AMD card
---

Run the parity suite **on the real card**.

**Targets:** `$1` — a space-separated list of test targets (default: the whole parity family).

## Why this is not just "run the test"

**Linux here enumerates a software rasteriser only.** It initialises fine and then **segfaults inside the
rasteriser on the first compute dispatch**, on a driver worker thread, with no engine frames on the stack. A
crash like that is the driver, not the kernel. **Never debug a shader against it.**

The AMD card is only visible to Windows binaries.

## The protocol

```bash
cmake --build build-win --target <targets> -j"$(nproc)"
cp build-win/*.exe /mnt/c/Source/strata-win/
powershell.exe -NoProfile -Command "cmd /c \"cd C:\Source\strata-win && <test>.exe --selftest > out.txt 2>&1\""
```

**Copy first.** For anything whose exit code or output you need to trust, running the exe through WSL interop
mangles the exit code — an ACCESS_VIOLATION (`0xC0000005`) surfaces as `exit=5`, which reads like "five failures"
instead of "crashed".

**Redirect with cmd's `>`, not PowerShell's.** PowerShell writes UTF-16 and `cat` shows every other byte as a
space.

**A crashing test prints nothing** — stdio is block-buffered when redirected, so the trace is lost with the
process. A non-zero exit with empty output means *crashed*, not *printed nothing*.

## Before running

Check for a **stale engine** — two engine processes from cancelled runs (one at 12.3 GB host commit) held the card
and made even 351 slots unallocatable:

```bash
cmd.exe /c 'tasklist /NH' 2>/dev/null | grep -iE "unbiased-strata|strata-poly|strata\.exe"
```

Kill any that are running before measuring anything.

## Reading the result

- Exit **0** = pass. Exit **77 / 3 / 32** = skip — say which arm did not run and why; a skip is not a pass.
- Exit **1** = a check failed. Read the failure shape before reaching for the shader: **all variants agree with
  each other but not with the reference** is a layout or convention disagreement; **stale or previous output** is
  an ordering bug, not a math bug.
- A scale that is fp16 on both sides differing by one ulp is **expected** — only the codes are exact.

Report each test's verdict, its exit code, and the failing check verbatim if any failed.
