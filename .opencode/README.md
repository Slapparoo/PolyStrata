# `.opencode/` — project configuration for PolyStrata

This directory is **project-specific**. It encodes the procedures this repository repeats, and the traps it has
already paid for, in a form an agent can load on demand.

`AGENTS.md` remains the short operating manual (installing for a user, working on the code, contributing). The
longer reference is `docs/` — `BACKENDS.md` for the engines and how to add to them, `LFM.md` for LFM2.5,
`AMD_HIP.md` for the AMD cards. Nothing here contradicts them; the skills are the *procedures* those documents
state as rules, written out step by step.

## Layout

```
.opencode/
├── opencode.jsonc          permissions, formatter off
├── skills/                 procedures, loaded on demand via the skill tool
├── agents/                 specialist subagents
└── commands/               slash commands
```

## Skills

| Skill | What it is for |
| --- | --- |
| `polystrata-engine` | The `Engine` hook table: what every architecture shares, what a class supplies, and where a new backend or architecture goes. |
| `polystrata-build` | Which build command produces which target; the CMake options; the CUDA / HIP / SYCL split; the installer's families. |
| `polystrata-parity-test` | Writing or extending an instrument in `src/kernels/*_parity.cpp` or `src/poly/*_test.cu`: the `--selftest` contract, tolerances, negative controls, registering a test. |
| `polystrata-vulkan-port` | Porting a kernel to the Vulkan backend: the seam, the two-edit shader rule, the ordering model, the host-node rules, the die-by-name stubs. |
| `polystrata-amd-gpu` | Running the real AMD card from WSL: the copy-and-run protocol, the exit-code and redirect traps, the unbuffered-stdout pattern. |
| `polystrata-docs` | Which document owns which fact, the style rule the docs follow, and the same-change rule. |

## Agents

| Agent | What it is for |
| --- | --- |
| `vulkan-port-reviewer` | Reviews a change to the Vulkan backend against the CUDA reference and the port conventions. Read-only. |
| `parity-auditor` | Audits an instrument for vacuous passes, wrong tolerances and missing negative controls. Read-only. |

## Commands

| Command | What it does |
| --- | --- |
| `/vk-merge` | Run the 3-way merge of the Vulkan port in, and report the conflicts that need a decision. |
| `/vk-build` | Configure and build the Vulkan tree. |
| `/vk-test` | Build, deploy and run the parity suite on the real AMD card. |
| `/poly-spike` | Spike one poly architecture on the Vulkan runtime, LFM2.5 first. |
| `/doc-sync` | Bring the documents in line with the current tree. |

## Conventions used here

- Paths are relative to the repository root.
- Real target names, real file names, real command lines. No placeholders that do not exist.
- The docs' own rule applies here too: **plain words, measured numbers with what they were measured on, no
  claims without a measurement.**
