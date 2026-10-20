---
name: PolyStrata AMD GPU
description: Run PolyStrata's Windows binaries on a real AMD card from WSL - the copy-and-run protocol, the exit-code and redirect traps, the unbuffered-stdout pattern, and the software-rasteriser trap. Use when a parity test must actually run on the card, when a Windows exe crashes or prints nothing, or when the device is queried.
---

# Running on the AMD card

**The AMD card is only visible to Windows binaries.** Linux here enumerates a software rasteriser: it initialises
fine and then **segfaults inside the rasteriser on the first compute dispatch**, on a driver worker thread, with
no engine frames on the stack. A crash like that is the driver, not the kernel — **do not debug shaders against
it.**

The card this work targets: a **Radeon Pro 580X**, AMD proprietary driver, **warp 64**, and **no fp16 / bf16 /
dp4a / matrix cores** — so the fp32 reduction paths are not a temporary shortcut, that is the hardware. The HIP
build in this tree covers wave32 RDNA (gfx1100, gfx1201); the Vulkan backend is what reaches gfx906.

## The protocol

Build in `build-win`, then **copy the `.exe` to `C:\Source\strata-win` and run it from there**:

```bash
cmake --build build-win --target rope_parity -j"$(nproc)"
cp build-win/*.exe /mnt/c/Source/strata-win/
powershell.exe -NoProfile -Command "cmd /c \"cd C:\Source\strata-win && rope_parity.exe --selftest > out.txt 2>&1\""
```

For anything whose **exit code or output you need to trust**, the copy is not optional.

## The three traps

### 1. Running the exe through WSL interop mangles the exit code

An ACCESS_VIOLATION (`0xC0000005`) surfaces as `exit=5`, which reads like "five failures" instead of "crashed".
Run from `C:\Source\strata-win` and the true `-1073741819` comes back.

### 2. Redirect with `cmd /c`'s `>`, not PowerShell's

PowerShell writes UTF-16, so `cat` shows every other byte as a space. If you must use PowerShell, read back with
`iconv -f UTF-16LE -t UTF-8`.

### 3. A crashing test prints nothing

stdio is block-buffered when redirected, so the trace is lost with the process. **A non-zero exit with empty
output means *crashed*, not *printed nothing*.**

The pattern is one line in `main`:

```cpp
setvbuf(stdout, nullptr, _IONBF, 0);
```

## The device query

```bash
cp build-win/strata-device.exe /mnt/c/Source/strata-win/
powershell.exe -NoProfile -Command "cmd /c \"cd C:\Source\strata-win && strata-device.exe --selftest > out.txt 2>&1\""
```

`strata-device --selftest` prints the memory plan. It and the probe must agree.

**`cudaMemGetInfo` is the DEVICE_LOCAL heap(s), never the sum of every heap.** A discrete GPU also lists its
shared system memory as `HOST_VISIBLE`, and summing all heaps over-reports badly — on an 8 GiB card it reported
23.69 GiB. A VRAM plan that places experts in memory the card does not have is the consequence of getting this
wrong. An integrated GPU is the one case that takes every heap, because it has no `DEVICE_LOCAL`-flagged heap.

## A stale engine is the first suspect

Two engine processes from cancelled runs (one at 12.3 GB host commit) held the card and made even 351 slots
unallocatable. `taskkill /F /IM unbiased-strata.exe` until none remain, then start.

The front end's auto tier is also capped by the Windows COMMIT (`ullAvailPageFile - 2 GiB`): VRAM free is not
enough, a device allocation is charged to RAM + page file, and a failed open closes its partial arena before the
retry shrinks.

## The expert tier on this card

**A GPU HIT is the slow path here.** A hit is ~7× slower per expert than a parallel AVX-2 miss (16 threads
against one scalar INT8 dot), because the expert kernels are fp32 and this card has no fp16/bf16/dp4a/matrix
units. The trade is **arithmetic, not memory**: fewer slots means more CPU misses and the GPU waits on them;
more slots means more VRAM hits, and a hit is slow.

Measured on an 8 GiB card: 400 slots 2.86 tok/s, 800 slots (14.9% hits) 4.72, `auto` (42% hits) 1.73.

**A tier claim needs repeats, and the arms must be interleaved.** Seven identical 800-slot probes measured
2.50–4.12 tok/s (mean 3.43, sd 0.65). One serve per arm is a sweep, not a measurement.

## The host arena is what pages, not the card

The card is 8 GiB. The engine's resident expert arena is `VirtualLock`ed on a 7 GB RAM box — that is the
"swapping to system RAM", and the child's private commit is ~19.5 GB with it. `--mmap-experts` removes it: commit
5.0 GB, measured 3.69 tok/s against 4.54/4.69 with the arena. The page cache is the price.

## Never hand the GPU a host pointer without asking

Vulkan cannot alias an arbitrary host allocation into a buffer on this driver, so the register refuses and says
so. A NaN through a quantizer surfaces as a plausible answer, not as a crash.
