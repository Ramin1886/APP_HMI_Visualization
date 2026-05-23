# Design — HMI Visualization Application

> Functional specification. Distilled from the project roadmap §3.19 (Turn 38) and §3.19 implementation block (Turn 39). What the application must do.

---

## 1. Scope

The HMI Visualization application renders driver-facing video for an ADAS parking product. It produces Full HD (1920×1080) @ 30 FPS multi-view video from four fisheye cameras with selectable view modes and parking-task overlays.

The application is **stateless w.r.t. user-visible state** and architecturally separate from the perception software stack. It consumes the `ADAS_Framework` framework for scheduling, IPC, logging, configuration, XCPlite, diagnostics, and time.

### 1.1 Inputs (the entire input contract)

| Input | Source | Transport (via framework) |
|---|---|---|
| Camera frames | Camera ISP (teed) | Direct V4L2 capture (per-camera thread, framework `scheduler` periodic task) |
| Camera calibration | Perception | IPC pub/sub topic `perception.calibration` |
| PDC zones | USS ECU via vehicle bus | IPC pub/sub topic `vehicle.pdc` |
| View mode + subview requests | HMI control layer | IPC pub/sub topic `hmi.view_mode` |
| Parking slot corners + count | Perception | IPC pub/sub topic `perception.snapshot` |
| Selected slot, target pose, intermediate pose, steering tube | Planner | IPC pub/sub topic `planner.snapshot` |
| Ego motion delta | Perception or vehicle dead-reckoning | IPC pub/sub topic `vehicle.ego_motion` |
| Haptic input | HMI input device | IPC pub/sub topic `hmi.haptic` |

Schemas live in `schemas/` (per-topic FlatBuffers files); transport selected per topic via `ADAS_Framework::ipc` config.

All world-frame coordinates: ego rear-axle center frame.

### 1.2 Output

A single 1920×1080 framebuffer at 30 FPS delivered to the in-vehicle display via DRM/KMS (QNX) or display compositor.

---

## 2. Framework dependencies

The application **uses** the following `ADAS_Framework` primitives:

| Need | `ADAS_Framework` primitive |
|---|---|
| Render loop scheduling | `scheduler::Scheduler` periodic task at 30 Hz |
| Camera capture threads | `scheduler::Scheduler` periodic tasks (one per camera, 30 Hz) |
| Input topic consumption | `ipc::Subscriber<T>` per topic |
| Live calibration of rendering parameters (e.g., overlay colors, BEV extent, blend factors) | `xcp::ADAS_XCP_CAL(...)` |
| Live measurement (e.g., frame time, dropped frames, GPU usage) | `xcp::ADAS_XCP_VAR(...)` |
| Logging | `log::ADAS_LOG_<LEVEL>(...)` |
| Config loading | `config::Store` |
| Health reporting | `diag::Counter`, `diag::Gauge`, heartbeats |
| Time | `time::MonotonicClock`, `time::Duration` |
| Error handling | `error::Result<T, E>` |
| Math (fisheye projection, transforms) | `common::math` |
| Strong-typed units | `common::units` |
| Test fixtures + mocks | `test::FrameworkFixture`, `test::MockClock`, `test::MockTransport` |

The application **does not** implement scheduling, IPC, XCP, logging, config, diagnostics, time, error handling, or common math.

---

## 3. View modes

### 3.1 Top-level mode

`off` / `single` / `split`. Transitions are instantaneous (stateless).

### 3.2 Available views (10)

| View | Description |
|---|---|
| `bev` | Bird's Eye View — top-down composite from all 4 fisheyes, ground-plane homography, blended at overlaps. Haptic rotate / zoom. |
| `pano_front` | Cylindrical panorama from left-mirror + front + right-mirror fisheyes |
| `pano_rear` | Cylindrical panorama from left-mirror + rear + right-mirror fisheyes |
| `curb_l` | Curbstone view, left side: hybrid projection emphasizing the left curb |
| `curb_r` | Curbstone view, right side: symmetric |
| `single_front` | Fisheye-to-rectilinear remap of the front fisheye |
| `single_left` | Fisheye-to-rectilinear remap of the left-mirror fisheye |
| `single_right` | Fisheye-to-rectilinear remap of the right-mirror fisheye |
| `single_rear` | Fisheye-to-rectilinear remap of the rear fisheye |
| `floating_3d` | 3D scene: ego mesh + textured ground; haptic rotate / zoom. **No overlays.** |

Split mode: any two of the above; no nested splits.

Front pinhole camera is **not** used by HMI — fisheyes only.

---

## 4. Per-view rendering

Per `roadmap §3.19` rendering specifics. Same as the standalone HMI bundle previously; recapped here briefly.

| View | Approach |
|---|---|
| BEV | Inverse perspective mapping per fisheye onto ground plane + soft-mask blending in overlaps |
| Panorama front/rear | Cylindrical projection from 3 contributing fisheyes |
| Curbstone left/right | Hybrid: BEV-like ground + near-vertical plane projection |
| Single | Fisheye-to-rectilinear remap with Brown-Conrady or KB model (whichever the calibration uses) |
| 3D floating | 3D scene with ego mesh + textured ground; orbital haptic-controlled camera |

Calibration source: `ADAS_Framework::ipc` subscriber on `perception.calibration` topic. Updates flow live; renderer recomputes per-view projection matrices on calibration change.

---

## 5. Overlay rendering

### 5.1 Overlay applicability matrix

| Overlay | bev | pano_f/r | curb_l/r | single_front | single_rear | single_l/r | floating_3d |
|---|---|---|---|---|---|---|---|
| Steering tube | ✓ | — | — | ✓ | ✓ | — | — |
| Intermediate pose (ghost-car) | ✓ | — | — | — | — | — | — |
| Target pose (ghost-car) | ✓ | — | — | — | — | — | — |
| All parking slots | ✓ | — | — | — | — | — | — |
| Selected parking slot | ✓ | — | — | — | — | — | — |
| PDC zones | ✓ (line sections) | — | — | ✓ (3D plane sections) | ✓ (3D plane sections) | — | — |

### 5.2 Overlay specifics

- **Steering tube**: polyline in ego-frame, projected per view
- **Intermediate / Target pose**: 2D ghost ego-car silhouette with distinct colors (configurable via XCP calibration)
- **Parking slots**: 4-corner polygons (all in white outline; selected highlighted)
- **PDC**: in BEV as line sections around ego footprint; in single front/rear as 3D plane sections perpendicular to optical axis at zone distance; colored by proximity (configurable thresholds via XCP)

---

## 6. Stateless design semantics

The renderer is stateless w.r.t. user-visible state. Each frame's overlays / view-mode behavior is a pure function of current inputs.

**One exception — bounded local pixel ring** for blind-spot fill:
- Per-camera ring of ~2–4 frames (~120–250 ms)
- Warped forward each frame using `ego_motion_delta`
- Fills the dark zone around the ego footprint occluded by body / wheels
- Not user-visible state; bounded; documented in `architecture.md`

On HMI restart, blind-spot fill degrades gracefully and recovers in 2–4 frames.

---

## 7. Camera ISP teeing

Camera ISP path tees raw fisheye frames to:
- ASIL-B perception partition (image features for the NPU pipeline; different format)
- QM HMI partition (ISP-processed YUV for rendering; this application)

Hardware-integration requirement on the SoC platform.

---

## 8. Latency and compute budgets

| Stage | Target |
|---|---|
| Camera capture → ISP output | ~33 ms |
| ISP → HMI partition | < 2 ms |
| Render + composite | ~12–20 ms |
| Output → display | ~16–33 ms |
| **Total camera-to-display** | **~65–90 ms** |

| Resource | Budget |
|---|---|
| Adreno GPU | ~40–50% |
| CPU (in QM partition) | 2 cores |
| Memory | ~200–300 MB |
| Hexagon NPU | 0 |

---

## 9. ASIL classification

Entirely QM. Cannot drive safety-critical actuation. Receives safety-relevant data (PDC, slot positions, planner outputs) for display only. Hypervisor isolation enforces that HMI cannot write into the ASIL-B perception partition (via `ADAS_Framework::ipc` read-only shared-memory transport from the HMI side).

---

## 10. Failure modes (recap)

| Failure | Required behavior |
|---|---|
| One camera frame missing | Render with last frame for up to ~100 ms; then mask region or omit contribution |
| Calibration invalid | Use factory calibration only; log warning |
| PDC zones invalid | Do not render PDC overlays |
| Planner inputs missing | Do not render those overlays; render valid ones |
| Ego motion delta unavailable | Skip blind-spot fill; recover when delta resumes |
| Haptic input unavailable | Defaults: rotation = 0, zoom = 1.0 |
| Display path stalled | Drop oldest framebuffer rather than block render loop |
| Render thread crash | Framework scheduler restarts the task; capture continues |

No failure causes: partition crash, ASIL-B partition writes, misleading overlays (better to omit than mislead).

---

## 11. Implementation language and toolchain

Per roadmap Turn 39:

- C++17 (C++20 if `ADAS_Framework` is built with C++20)
- Vulkan primary, OpenGL ES 3.2 fallback
- CMake (consumes `ADAS_Framework` CMake modules)
- GLSL → SPIR-V shaders
- glm for math (via `ADAS_Framework::common::math`)
- FlatBuffers via `ADAS_Framework::ipc` (schemas in `schemas/`)
- V4L2 capture wrapped in C++ RAII
- GoogleTest + golden-frame regression harness in `tools/golden/`

Rejected alternatives (per roadmap): Rust, C, Qt for Automotive, Kanzi, Crank Storyboard, Slint, wgpu.

---

## 12. Cross-references

- Architectural module structure: `architecture.md`
- Naming convention: `ADAS_Framework/docs/naming-convention.md` (canonical)
- Phase plan: `instruction.md`
- Operational rules: `GEMINI.md`
- Project roadmap: parent `ADAS_Perception_Startup_Roadmap.md` §3.19
