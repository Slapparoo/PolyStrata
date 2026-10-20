---
description: Configure and build the Vulkan backend, and check that a change links
---

Build the Vulkan backend of PolyStrata.

**Target:** `$1` — a CMake target (default `strata-poly`), or `all` for the tree's default target.
**Tree:** `$2` — `linux` (default) or `win` for the Windows cross build.

## The tree

```bash
cmake -S . -B build-vulkan -DSTRATA_ENABLE_VULKAN=ON -DCMAKE_BUILD_TYPE=Release -DSTRATA_BUILD_TESTS=ON
cmake --build build-vulkan --target <target> -j"$(nproc)"
```

`STRATA_VULKAN` is **exclusive** with `STRATA_ENABLE_CUDA`, `STRATA_ENABLE_HIP` and `STRATA_ENABLE_SYCL` — the
configure refuses a tree with more than one.

`STRATA_BUILD_TESTS` must be passed **explicitly**: its default is on only when both `tests/CMakeLists.txt` and
`bench/micro` exist.

The build also needs `glslc` (the `shaderc` package) — it compiles the shaders to SPIR-V at build time and
embeds them as C++ byte arrays. Without it the configure says so by name.

## The Windows cross build

This is the path that sees the card.

```bash
cmake -S . -B build-win -DCMAKE_TOOLCHAIN_FILE=tmp/mingw-windows.cmake \
      -DSTRATA_ENABLE_VULKAN=ON -DCMAKE_BUILD_TYPE=Release -DSTRATA_BUILD_TESTS=ON
cmake --build build-win --target <target> -j"$(nproc)"
```

`tmp/mingw-windows.cmake` and `tmp/mingw-vulkan/` arrive with the merge. glslc runs on **Linux** even for the
cross build — SPIR-V is target-independent.

## The shader registry

Every shader needs **two** edits or it is never compiled: the `.comp` file, and a line in `SHADER_SPECS` in
`src/vulkan/CMakeLists.txt`:

```
name.comp|bindings|push_bytes|local_x|local_y|local_z
```

**Whitespace-separated, no commas** — a comma is a configure error at the wrong line. A shared `.glsl` include
also has to be in `SHADER_INCLUDES`, or the shaders that include it are not rebuilt.

## After the build

**A link of the full target is the only honest check of kernel coverage.** Archives are pulled on demand, so a
small target linking proves nothing about the full symbol surface.

If the build claims success **without relinking**, suspect a stale zero-filled output from an interrupted link
and delete it.

Report: the target built, the configure options in effect, the submodule state the configure printed (commit,
branch, and whether the ggml interop accessor is present or ABSENT), and any warning output. If the build
failed, report the first error with its file and line — and read it as a runtime or layout bug before assuming
the shader is at fault.
