# Instruction — HMI Visualization Application Implementation Plan

> Phase-by-phase work plan. Do not advance to the next phase until the current phase passes its acceptance criteria and is approved by a human reviewer.

> **Prerequisite**: `ADAS_Framework` Phase 1 must be complete (foundation layer: common, error, time, log). Phase 0 of this application repo can proceed in parallel with `ADAS_Framework` Phase 0; later phases need progressively more of `ADAS_Framework`.

---

## How to use this file

Operating per `GEMINI.md`. Before any code:
1. Re-read `design.md` and `architecture.md`
2. Re-read `docs/naming-convention.md` (from `ADAS_Framework` submodule)
3. Confirm which phase you are starting; do not skip
4. At end of each phase: `docs/phase-reports/phase-<N>.md`

---

## Phase 0 — Bootstrap

### Goal
Project skeleton + build system + `ADAS_Framework` submodule pointer.

### Deliverables
- Directory layout per `architecture.md` §2
- Git submodule: `third_party/ADAS_Framework` pointing at the framework repo (initially `main` branch; will be pinned to a tagged release later)
- `CMakeLists.txt` (top-level) that:
  - Includes `third_party/ADAS_Framework/CMakeLists.txt` as a subdirectory (or finds it via `find_package(ADAS_Framework)` if installed)
  - Defines the `APP_HMI_Visualization` library target linking against `ADAS_Framework`
  - Includes `ADAS_Framework`'s CMake modules (`adas-compiler-flags`, `adas-clang-tidy`, etc.)
- `.clang-format` and `.clang-tidy`: thin files that include / inherit from `ADAS_Framework/.clang-format` and `.clang-tidy` (or are direct copies if include syntax isn't supported, with a CI check that they match)
- `.editorconfig`, `.gitignore`, `.gitattributes` (LFS for golden-frame reference images)
- `.gitmodules`
- CI configuration with gates: format / tidy / build / test / golden-frame regression
- `README.md`, `docs/getting-started.md`, `docs/maintenance.md`, `docs/troubleshooting.md`, `docs/framework-requests.md`
- In-tree copies of bootstrap docs under `docs/` (`design.md`, `architecture.md`, `GEMINI.md`); naming-convention.md copied from `ADAS_Framework/docs/`
- Empty module skeletons per `architecture.md` §3 (each module: `include/adas/app/hmi/<module>/`, `src/<module>/`, `test/<module>/`, `README.md`)
- `schemas/` directory with `README.md` describing schema versioning policy
- `shaders/` directory with `README.md`
- `assets/` directory with placeholder ego-vehicle mesh (simple box for now)
- `tools/golden/` skeleton

### Acceptance criteria
- Fresh-clone `git submodule update --init && cmake -B build && cmake --build build && ctest` succeeds on Ubuntu LTS dev host (after `ADAS_Framework` Phase 0 is also complete on the framework side)
- `clang-format --check` and `clang-tidy` pass
- CI is green
- `docs/getting-started.md` walks a new developer from clone → passing build in < 30 min

---

## Phase 1 — Capture (V4L2)

### Goal
Camera frame capture via framework scheduler.

### Pre-req
`ADAS_Framework` Phases 0–3 (bootstrap + common/error/time/log + config/diag + in-process IPC) complete.

### Deliverables
- `capture::Camera`: V4L2 device wrapper (open, configure format, start streaming, dequeue, requeue, stop, close) wrapped in RAII
- Per-camera periodic task registered with `ADAS_Framework::scheduler::Scheduler` at 30 Hz; publishes to in-process topic `hmi.internal.frame.<camera>`
- `CameraFrame` type (FlatBuffers schema): YUV data pointer + dimensions + timestamp + camera-id
- Mock camera source for dev hosts without hardware (reads from canned video file)
- Configuration via `ADAS_Framework::config::Store`
- `ADAS_XCP_VAR(uint64_t, capture_<camera>_frame_count, ...)` per camera

### Acceptance criteria
- Real-hardware path: 30 FPS sustained for >10 minutes without drops on a USB UVC camera on dev host
- Mock path: deterministic replay
- TSan/ASan/UBSan clean
- Coverage ≥ 85%

---

## Phase 2 — Calibration ingestion

### Goal
Subscribe to `perception.calibration` and expose to renderers.

### Pre-req
`ADAS_Framework` Phase 3 (in-process IPC) + Phase 4 (shared-memory IPC, if using cross-partition transport).

### Deliverables
- `calibration::Calibration` type: factory K/R/t per camera + residual correction
- `calibration::Subscriber`: `ADAS_Framework::ipc::Subscriber<CalibrationUpdate>` wrapper publishing into a TripleBuffer
- `calibration::Provider::get_current()` thread-safe accessor
- Cache invalidation hook (downstream renderers re-compute homography when calibration updates)
- Mock calibration publisher in test harness

### Acceptance criteria
- Round-trip projection test: factory + residual project + unproject within tolerance
- Stress test: 100k concurrent reads + writes, TSan-clean
- Calibration update propagates to renderer within < 5 ms (measured end-to-end)

---

## Phase 3 — Render scaffolding + first view (BEV)

### Goal
GPU bring-up + first view.

### Pre-req
`ADAS_Framework` Phase 5 (scheduler) complete.

### Deliverables
- Vulkan instance / device / swapchain bring-up
- Shader pipeline (GLSL → SPIR-V via `ADAS_Framework/cmake/adas-shader.cmake`)
- `render::IView` interface
- `render::bev::BevRenderer`:
  - Ground-plane homography from each fisheye using calibration
  - Soft-mask blending in overlaps
  - Haptic zoom/rotation parameters as `ADAS_XCP_CAL(float, bev_zoom, ...)`, etc.
  - Configurable extent (`bev_extent_m` XCP cal)
- Display path: DRM/KMS framebuffer submission (X11 fallback on dev host for visual inspection)
- Render task registered with `ADAS_Framework::scheduler::Scheduler` at 30 Hz

### Acceptance criteria
- 30 FPS sustained on dev host (mock or real camera input)
- Latency < 90 ms capture-to-display (measured)
- No visible stitch seams on canned data
- Golden-frame regression test (first reference image generated)
- GPU memory use within `design.md` §10 envelope

---

## Phase 4 — Remaining views

### Goal
Implement all remaining views.

### Pre-req
Phase 3 complete.

### Deliverables (each with golden-frame test)
- `render::pano::PanoFrontRenderer`, `PanoRearRenderer`
- `render::curbstone::CurbstoneLRenderer`, `CurbstoneRRenderer`
- `render::single::SingleFrontRenderer`, `SingleLeftRenderer`, `SingleRightRenderer`, `SingleRearRenderer`
- `render::floating3d::Floating3dRenderer` (with placeholder ego mesh)
- `compose::Compositor` for split + single + off modes

### Acceptance criteria
- Each view passes its golden-frame regression test
- View switching observably instantaneous (frame N+1 shows new view)
- Compute budget within `design.md` §10

---

## Phase 5 — Overlays

### Goal
Implement all overlays per `design.md` §5.

### Pre-req
Phase 4 complete; `ADAS_Framework` Phase 4 (shared-memory IPC) complete so perception/planner inputs flow.

### Deliverables
- `overlay::SteeringTube`
- `overlay::GhostCar` (intermediate + target with colors via XCP cal)
- `overlay::ParkingSlots` (all)
- `overlay::SelectedSlot`
- `overlay::PdcLineSections` (BEV)
- `overlay::PdcPlaneSections` (single front/rear)
- Per-view applicability per `design.md` §5.1
- Coordinate-frame projection shared logic in `common::projections` (small helper module)

### Acceptance criteria
- Each overlay passes golden-frame test in each applicable view
- Overlay rendering within GPU budget
- Visual inspection: overlays do not lag image content when inputs are fresh

---

## Phase 6 — Blind-spot fill

### Goal
Bounded pixel ring + ego-motion warp.

### Pre-req
Phase 3 complete.

### Deliverables
- `bsfill::BsFiller` per camera with bounded ring (depth from `ADAS_XCP_CAL` `bsfill_ring_depth`)
- Ego-motion-warp shader (uses SE(2) delta to project past pixels forward)
- Integration into BEV
- Graceful behavior when `ego_motion_delta` is unavailable (skip fill)

### Acceptance criteria
- Blind spot under car appears filled in BEV when ego is moving
- No ghosting beyond ~250 ms of motion
- Bounded ring (no growth)
- Recovery after HMI restart: 2–4 frames

---

## Phase 7 — Input handling

### Goal
View-mode + haptic input subscribers.

### Deliverables
- `input::ViewModeSubscriber` consuming `hmi.view_mode` topic
- `input::HapticSubscriber` consuming `hmi.haptic` topic
- Provides latest snapshots to render module via TripleBuffer

### Acceptance criteria
- View-mode change propagates to render within 1 frame (< 33 ms)
- Haptic input drives BEV / 3D floating view interactively
- TSan/ASan-clean

---

## Phase 8 — Diagnostics & XCP wiring

### Goal
Surface all metrics via diagnostics + XCP.

### Pre-req
`ADAS_Framework` Phase 7 (XCP) complete.

### Deliverables
- All `ADAS_XCP_VAR` measurement points per `architecture.md` §6
- All `ADAS_XCP_CAL` calibration points per `architecture.md` §6
- A2L file generation integrated (via `ADAS_Framework::adas-xcp-a2l.cmake`)
- Health heartbeats from render task
- `diag::Counter` / `diag::Gauge` for: dropped frames, frame time, GPU usage, calibration update count
- DEM events: render-thread crash → DEM event reported via `ADAS_Framework::diag`

### Acceptance criteria
- A2L generated and parseable by `xcp_lite` reference master
- Live tuning via `FakeXcpHost` demonstrated (change `bev_zoom`, observe BEV updates)
- All declared XCP variables observable

---

## Phase 9 — Cross-ECU variant

### Goal
Validate separate-cockpit-ECU deployment.

### Pre-req
`ADAS_Framework` Phase 8 (Some/IP transport) complete.

### Deliverables
- Build target for at least one cockpit ECU (SA8295P preferred; or document why deferred)
- Camera frame streaming over Ethernet (replaces local V4L2 capture; consumes from a remote `vehicle.camera.<id>` topic instead)
- Perception/planner inputs over Ethernet (same FlatBuffers schemas, different transport)
- CMake option `HMI_DEPLOYMENT=same_soc|separate_ecu`

### Acceptance criteria
- Same source tree builds for both deployments
- Latency budget still met (< 100 ms camera-to-display) for separate-ECU
- Documented in `docs/deployment-variants.md`

---

## Phase 10 — Stress testing + performance hardening

### Goal
Sustained production-grade performance.

### Deliverables
- Performance test harness in `tools/perf/`
- Frame-time p99 jitter < 5 ms measured
- ASan / TSan / UBSan clean across full app
- 2-hour soak test passes (no memory growth, no frame drops)

### Acceptance criteria
- All `design.md` §8/§10 targets met
- CI perf benchmarks gate regressions

---

## Phase 11 — Documentation finalization

### Goal
Production-ready docs.

### Deliverables
- `docs/api-reference.md` (Doxygen)
- `docs/architecture.md` updated to as-built
- `docs/design.md` updated to as-built
- `docs/getting-started.md` final
- `docs/maintenance.md`: adding a view, adding an overlay, schema updates, calibration debugging, performance investigation
- `docs/troubleshooting.md`: common failure modes + resolutions
- Per-module README final

### Acceptance criteria
- New developer onboards in < 30 min
- Doxygen coverage ≥ 95% public API

---

## Per-phase report template

Same as `ADAS_Framework/docs/phase-reports/`. See `01-framework-ADAS_Framework/instruction.md`.

---

## Stop rules

Stop and ask the human if:
- An acceptance criterion can be met only by violating a rule in `GEMINI.md` or naming convention
- A phase has consumed more than 3x its expected effort
- A `design.md` or `architecture.md` decision proves wrong in light of implementation
- `ADAS_Framework` needs a new primitive that doesn't exist
- The hardware path proves unworkable

Do not silently work around. Do not silently downgrade scope.
