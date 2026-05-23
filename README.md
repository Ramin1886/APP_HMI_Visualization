# `APP_HMI_Visualization` Bootstrap Bundle (for Gemini CLI)

This folder contains the bootstrap context for the **HMI Visualization application repository**.

## Files

| File | Purpose | Read order |
|---|---|---|
| `GEMINI.md` | Agent operating context, discipline rules | 1st |
| `design.md` | Functional spec — what the application must do | 2nd |
| `architecture.md` | Module decomposition, framework integration, IPC topology | 3rd |
| `instruction.md` | 12-phase work plan | 4th |
| `initial-prompt.md` | Copy-pasteable kickoff prompt | last |
| `README.md` | This file | — |

**Note**: The naming convention is owned by `ADAS_Framework` (Bundle 01). The HMI repo references it via the Git submodule. During Phase 0, the framework's `naming-convention.md` is copied into `docs/` for in-repo reference.

## How to use

1. Create a fresh Git repository named `APP_HMI_Visualization` (or your preferred name; search-replace).
2. Copy these files into the repository root.
3. Open the repo with Gemini CLI: `cd APP_HMI_Visualization && gemini`
4. Open `initial-prompt.md`. Copy the kickoff prompt.
5. Paste into Gemini as the first message.
6. Gemini reads the context and starts Phase 0.

## Prerequisites

- `ADAS_Framework` framework repository must exist (see Bundle 01)
- For Phase 0: `ADAS_Framework` Phase 0 (skeleton) sufficient
- For later phases: progressively more of `ADAS_Framework` (see `instruction.md` per-phase prerequisites)

## What this produces

A production-grade C++ HMI Visualization application:

- **Capture**: V4L2 camera capture with mock-source fallback for dev
- **Calibration**: subscriber consuming `perception.calibration` topic; live updates
- **Render**: 10 view modes (BEV, panorama F/R, curbstone L/R, single front/L/R/rear, 3D floating); Vulkan / OpenGL ES 3.2
- **Overlays**: steering tube, ghost-car target/intermediate poses, parking slots, PDC zones
- **Composer**: split / single / off modes
- **Blind-spot fill**: bounded per-camera pixel ring with ego-motion warp
- **Input**: view-mode + haptic input subscribers
- **XCP wiring**: ~10 measurement signals + ~10 calibration parameters exposed live
- **Diagnostics**: counters, gauges, DEM event reporting
- **Cross-ECU variant**: separate-cockpit-ECU deployment over Some/IP Ethernet

## What this does NOT produce

- Perception / planner / control / state-management logic — those are other applications in their own repos
- Framework primitives (scheduling, IPC, XCP, logging, config, diag, time, math) — those live in `ADAS_Framework`
- ML inference — out of scope; perception application's concern

## Repo dependency structure

```
APP_HMI_Visualization
  └── third_party/ADAS_Framework (git submodule)
```

When this repo is integrated into the system, it is managed as a Git submodule by `ADAS_Integration` (Bundle 03).

## Cross-references

- Project roadmap: `ADAS_Perception_Startup_Roadmap.md` §3.19 (Turn 38–39)
- Framework: `01-framework-ADAS_Framework/`
- Integration: `03-integration-ADAS_Integration/`
