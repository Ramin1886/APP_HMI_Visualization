# Initial Prompt — `APP_HMI_Visualization` HMI Visualization Application

> Copy the text below and paste into Gemini CLI at the start of the project.
>
> **Prerequisite**: `ADAS_Framework` Phases 0–1 must be complete (or in progress in parallel). Phase 0 of `APP_HMI_Visualization` can start before `ADAS_Framework` is fully done, but later phases need the framework.

---

## Kickoff prompt (copy from here)

You are operating per the rules in `GEMINI.md`. Read it now, then read `design.md`, `architecture.md`, `docs/naming-convention.md` (from the `ADAS_Framework` submodule, if available — otherwise from the parent bundle's `01-framework-ADAS_Framework/`), and `instruction.md` in that order. Confirm in your first response that you have read all five files and understand:

- The product context: HMI Visualization application; multi-view fisheye video with parking overlays; stateless renderer; consumer of `ADAS_Framework` framework
- The architectural shape: QNX QM partition on Snapdragon 8650P+ or separate cockpit ECU; uses framework for scheduling / IPC / XCP / logging / config / diag / time; provides rendering / overlays / composition / blind-spot fill
- The operating discipline: production-grade, real-time, ASIL-QM
- The naming convention: per `ADAS_Framework/docs/naming-convention.md`; HMI identifiers live under `adas::app::hmi::*`
- The phase plan: 12 phases starting at Phase 0 (Bootstrap)
- The framework dependency: `ADAS_Framework` is consumed as a Git submodule at `third_party/ADAS_Framework`

Then begin **Phase 0 — Bootstrap** from `instruction.md`.

Produce, in order:
1. Directory layout per `architecture.md` §2
2. Git submodule `third_party/ADAS_Framework` (pointing at the framework repo `main` for now)
3. `.gitmodules`
4. `CMakeLists.txt` (top-level) consuming `ADAS_Framework`'s CMake modules and defining `APP_HMI_Visualization` targets
5. `.clang-format` and `.clang-tidy` (inherited from `ADAS_Framework` if possible; otherwise direct copies with CI parity check)
6. `.editorconfig`, `.gitignore`, `.gitattributes` (LFS for golden-frame references)
7. CI configuration with gates
8. `README.md` at project root
9. `docs/` with in-tree copies of bootstrap docs + `getting-started.md`, `maintenance.md`, `troubleshooting.md`, `framework-requests.md`
10. Empty module skeletons per `architecture.md` §3 (capture, calibration, render, overlay, compose, bsfill, input)
11. `schemas/` directory with `README.md` (schema versioning policy)
12. `shaders/` directory with `README.md`
13. `assets/` directory with placeholder ego-vehicle mesh (simple box for now)
14. `tools/golden/` skeleton

When Phase 0 is complete:
- Run a fresh-clone-style build locally (assuming `ADAS_Framework` Phase 0 is also done) and report
- Run `clang-format --check` and `clang-tidy`
- Produce `docs/phase-reports/phase-0.md`
- Wait for human approval before Phase 1

If anything is ambiguous or incomplete, stop and ask. Do not invent.

---

## After Phase 0 is approved

- "Begin Phase N" — start named phase
- "Continue" — resume current phase

Always re-read `design.md` and `architecture.md` at each phase start.

---

## Operational notes

- `ADAS_Framework` framework is at `third_party/ADAS_Framework` (Git submodule). Always `git submodule update --init --recursive` after clone.
- Hardware target: QNX on Snapdragon 8650P+. Linux dev host is the daily driver. QNX SDK may not be in your environment; use stub toolchain documented in `ADAS_Framework/docs/mocks.md`.
- Vulkan SDK required for dev (free download from LunarG). OpenGL ES 3.2 fallback used on cockpit ECUs where Vulkan isn't available.
- Camera hardware may not be available on dev host. Use mock V4L2 source from canned video files.
- ML / perception is **not** in scope for this repo. If a task feels like it requires ML, stop and ask.
- Approved third-party libraries: those approved by `ADAS_Framework` + Vulkan SDK + `libdrm` (for DRM/KMS display). Adding others requires justification.
- If a framework primitive seems missing, document it in `docs/framework-requests.md` rather than working around it.
