# Architecture — HMI Visualization Application

> Module decomposition and integration with `ADAS_Framework`. `design.md` covers *what*; this file covers *how*.

---

## 1. Position in the larger system

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                  Snapdragon 8650P+ (QNX hypervisor)                          │
│                                                                              │
│  ┌────────────────────────────┐         ┌──────────────────────────────┐   │
│  │   ASIL-B partition         │         │   QM partition               │   │
│  │   (perception-app)         │         │   (APP_HMI_Visualization)  ← this codebase │   │
│  │                            │         │                              │   │
│  │   - NPU pipeline (Hexagon) │  ipc    │   - Adreno GPU rendering     │   │
│  │   - perception heads       │ ─────►  │   - V4L2 capture             │   │
│  │   - planner (or separate)  │  via    │   - Display output           │   │
│  │                            │ adas    │                              │   │
│  │   Publishes:               │ -core   │   Subscribes:                │   │
│  │   - perception.snapshot    │         │   - perception.snapshot      │   │
│  │   - perception.calibration │         │   - perception.calibration   │   │
│  │   - planner.snapshot       │         │   - planner.snapshot         │   │
│  │   - vehicle.ego_motion     │         │   - vehicle.ego_motion       │   │
│  └────────────────────────────┘         └──────────────────────────────┘   │
│                                                                              │
│  Both partitions link `ADAS_Framework` (built per-partition with same source).   │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
                                       ▲
                                       │ ipc::transport::SharedMemory
                                       │ (read-only from HMI side; enforced
                                       │  by hypervisor memory permissions)
                                       │

   ┌───────────────────────┐
   │ Camera ISP (4×)       │ ── MIPI CSI-2 teed → both partitions
   │ Fisheye sensors       │
   └───────────────────────┘

   ┌───────────────────────┐
   │ USS ECU               │ ── vehicle bus (CAN-FD or Ethernet) → ADAS_Framework::ipc::transport::CanFd or SomeIp
   │ (12 PDC zones)        │
   └───────────────────────┘

   ┌───────────────────────┐
   │ HMI control surface   │ ── input bus → ADAS_Framework::ipc topic "hmi.view_mode" and "hmi.haptic"
   │ (touchscreen + haptic)│
   └───────────────────────┘
```

---

## 2. Repository structure

```
APP_HMI_Visualization/
├── include/adas/app/hmi/        # public headers
│   ├── capture/
│   ├── calibration/
│   ├── render/
│   │   ├── bev/
│   │   ├── pano/
│   │   ├── curbstone/
│   │   ├── single/
│   │   └── floating3d/
│   ├── overlay/
│   ├── compose/
│   ├── bsfill/
│   └── input/
├── src/                         # implementation; mirrors include/
├── test/                        # tests; mirrors include/
├── shaders/                     # GLSL → SPIR-V shader sources
├── schemas/                     # FlatBuffers schemas owned by this app (e.g., hmi.view_mode_v1.fbs)
├── assets/                      # ego mesh (glTF), default config, fonts
├── tools/                       # golden-frame test harness, replay tools
├── docs/                        # in-tree docs
│   ├── design.md
│   ├── architecture.md
│   ├── naming-convention.md     # copy of ADAS_Framework's; framework version is canonical
│   ├── getting-started.md
│   ├── maintenance.md
│   ├── troubleshooting.md
│   ├── framework-requests.md    # gaps requested from ADAS_Framework
│   ├── api-reference.md         # generated
│   └── phase-reports/
├── third_party/                 # pinned third-party (or git submodules)
│   └── ADAS_Framework/               # git submodule pointer (added in Phase 0)
├── CMakeLists.txt
├── .clang-format               # references ADAS_Framework's via include
├── .clang-tidy                 # references ADAS_Framework's via include
├── .editorconfig
├── .gitignore
├── .gitattributes
├── .gitmodules
├── GEMINI.md
└── README.md
```

The `ADAS_Framework` submodule is the canonical source. The application's `.clang-format` and `.clang-tidy` are symlinks or include-directives that pull from the submodule, ensuring consistency.

---

## 3. Module decomposition

```
                          ┌──────────┐
                          │   main   │
                          └────┬─────┘
                               │ (wires app modules to ADAS_Framework::scheduler)
        ┌──────────────────────┼─────────────────────┐
        │                      │                     │
   ┌────▼────┐         ┌───────▼─────┐         ┌─────▼─────┐
   │ capture │         │ calibration │         │   input   │
   │ (V4L2)  │         │ (subscriber)│         │(subscriber│
   └────┬────┘         └──────┬──────┘         └─────┬─────┘
        │                     │                      │
        └─────────────────────┼──────────────────────┘
                              │
                       ┌──────▼──────┐
                       │   render    │
                       │   ├─bev     │
                       │   ├─pano    │
                       │   ├─curbstn │
                       │   ├─single  │
                       │   └─float3d │
                       └──────┬──────┘
                              │
       ┌──────────────────────┼──────────────────────┐
       │                      │                      │
   ┌───▼────┐           ┌─────▼────┐           ┌─────▼─────┐
   │overlay │           │  bsfill  │           │  compose  │
   └────────┘           └──────────┘           └───────────┘
                              │
                       ┌──────▼──────┐
                       │   display   │
                       │ (DRM/KMS)   │
                       └─────────────┘

   ALL modules depend on:  ADAS_Framework (scheduler, ipc, log, config, xcp, diag,
                                       time, error, common::math, common::units)
```

### 3.1 Module responsibilities

| Module | Responsibility | Framework primitives used |
|---|---|---|
| `capture` | V4L2 frame capture, one task per camera (via framework scheduler); publishes captured frames to local in-process topic | `scheduler::PeriodicTask`, `ipc::Publisher<CameraFrame>` (in-process) |
| `calibration` | Subscribes to `perception.calibration`; provides thread-safe accessor; emits notification on calibration update | `ipc::Subscriber`, `common::TripleBuffer<Calibration>` |
| `render` | View renderers (one submodule per view); GPU resource management; Vulkan / OpenGL ES backend selection | `scheduler::PeriodicTask` (render loop @ 30 Hz), `xcp::ADAS_XCP_CAL` for tunable parameters, `common::math` |
| `overlay` | Overlay primitives: steering tube, ghost car, slot polygons, PDC sections | `common::math` for projections |
| `compose` | Multi-view compositor (single, split, off modes) | (none beyond render) |
| `bsfill` | Bounded blind-spot pixel ring + ego-motion-warp shader | `common::math::Se2` for warping |
| `input` | Subscribes to `hmi.view_mode` and `hmi.haptic`; provides latest snapshots | `ipc::Subscriber`, `common::TripleBuffer` |
| (`main`) | Wires everything to the framework scheduler; runs the application lifecycle | `scheduler::Scheduler`, `config::Store`, `log`, `diag::HealthMonitor` |

### 3.2 Dependency rules

- All HMI modules depend on `ADAS_Framework`
- HMI modules may depend on each other within the layered structure (capture / calibration / input → render → overlay / bsfill / compose)
- No cyclic dependencies; CI enforces

---

## 4. Scheduler-driven thread layout (via `ADAS_Framework::scheduler`)

| Task | Type | Rate / Trigger | Priority band |
|---|---|---|---|
| `hmi.capture.front` | Periodic | 30 Hz | Real-time (150) |
| `hmi.capture.left` | Periodic | 30 Hz | Real-time (150) |
| `hmi.capture.right` | Periodic | 30 Hz | Real-time (150) |
| `hmi.capture.rear` | Periodic | 30 Hz | Real-time (150) |
| `hmi.render` | Periodic | 30 Hz | Normal (80) |
| `hmi.calibration.sub` | Event | on `perception.calibration` | Normal (60) |
| `hmi.input.sub` | Event | on `hmi.view_mode` / `hmi.haptic` | Normal (60) |
| `hmi.display` | Periodic | 30 Hz (display vsync if available) | Normal (70) |

Priorities are config-driven; defaults here. All tasks register with `ADAS_Framework::scheduler::Scheduler` at startup.

---

## 5. IPC topology

### 5.1 Topics consumed (subscriptions)

| Topic | Type | Transport | QoS |
|---|---|---|---|
| `perception.snapshot` | `adas.ipc.v1.PerceptionSnapshot` | shared_memory | latest |
| `perception.calibration` | `adas.ipc.v1.CalibrationUpdate` | shared_memory | latest |
| `planner.snapshot` | `adas.ipc.v1.PlannerSnapshot` | shared_memory | latest |
| `vehicle.pdc` | `adas.ipc.v1.PdcSnapshot` | shared_memory or some/ip | latest |
| `vehicle.ego_motion` | `adas.ipc.v1.EgoMotionDelta` | shared_memory | latest |
| `hmi.view_mode` | `adas.ipc.v1.HmiViewMode` | in_process | latest |
| `hmi.haptic` | `adas.ipc.v1.HmiHapticInput` | in_process | latest |

### 5.2 Topics published

Internal in-process topics for fan-out between HMI modules:

| Topic | Type | Transport | QoS | Producer | Consumer |
|---|---|---|---|---|---|
| `hmi.internal.frame.front` | `adas.ipc.v1.CameraFrame` | in_process | latest | capture.front | render |
| `hmi.internal.frame.left` | same | in_process | latest | capture.left | render |
| `hmi.internal.frame.right` | same | in_process | latest | capture.right | render |
| `hmi.internal.frame.rear` | same | in_process | latest | capture.rear | render |
| `hmi.diag.frame_drops` | `adas.ipc.v1.Counter` | in_process | latest | render | diag aggregator |
| `hmi.diag.frame_time_ms` | `adas.ipc.v1.Gauge` | in_process | latest | render | diag aggregator |

### 5.3 Schemas owned

HMI-specific FlatBuffers schemas live in `schemas/`:
- `hmi_view_mode_v1.fbs`
- `hmi_haptic_input_v1.fbs`
- `hmi_internal_camera_frame_v1.fbs`

Cross-component schemas (perception, planner, vehicle, calibration) live in the **integration repo** `ADAS_Integration/schemas/` (per Bundle 03). Both this repo and the perception repo consume them via the integration repo path.

---

## 6. XCP exposure

Tunable rendering parameters (calibration via `ADAS_XCP_CAL`):

| Variable | Default | Range | Unit | Description |
|---|---|---|---|---|
| `bev_extent_m` | 20.0 | [5, 50] | m | BEV render extent (square side) |
| `bev_default_zoom` | 1.0 | [1.0, 4.0] | 1 | Default BEV zoom factor |
| `pdc_threshold_green_m` | 1.0 | [0.5, 2.0] | m | PDC green threshold |
| `pdc_threshold_yellow_m` | 0.5 | [0.3, 1.0] | m | PDC yellow threshold |
| `pdc_threshold_orange_m` | 0.3 | [0.1, 0.5] | m | PDC orange threshold |
| `overlay_target_color_r/g/b/a` | 0/255/0/180 | [0, 255] | 1 | Target-pose ghost-car color |
| `overlay_intermediate_color_r/g/b/a` | 100/200/255/180 | [0, 255] | 1 | Intermediate-pose color |
| `bsfill_ring_depth` | 3 | [2, 8] | 1 | Blind-spot ring frame count |

Measurement signals via `ADAS_XCP_VAR`:

| Variable | Unit | Description |
|---|---|---|
| `render_frame_time_ms` | ms | Per-frame render time |
| `dropped_frames_total` | 1 | Cumulative dropped frames |
| `current_view_mode` | enum | Active view mode |
| `gpu_usage_pct` | % | GPU utilization (when available from driver) |
| `calibration_update_count` | 1 | Cumulative calibration updates received |

XCP server runs in `ADAS_Framework::xcp` thread; macros declare in the application's source files.

---

## 7. Configuration

`APP_HMI_Visualization` consumes config via `ADAS_Framework::config::Store`. Application-specific schema:

```toml
[hmi]
log_level = "INFO"

[hmi.render]
target_fps = 30
target_width = 1920
target_height = 1080
graphics_api = "vulkan"            # vulkan | gles32

[hmi.capture]
camera_front_device = "/dev/video0"
camera_left_device = "/dev/video1"
camera_right_device = "/dev/video2"
camera_rear_device = "/dev/video3"
frame_format = "yuv420"

[hmi.bev]
extent_m = 20.0
zoom_min = 1.0
zoom_max = 4.0
zoom_default = 1.0

[hmi.bsfill]
ring_depth = 3
ring_max_age_ms = 250

[hmi.display]
device = "/dev/dri/card0"          # DRM/KMS device
connector_id = 0                   # selected connector (0 = first)
```

Loaded once at startup via `ADAS_Framework::config::Store::initialize(...)`. Hot-reloadable subset: `log_level`. Critical fields (devices, FPS, resolution) require restart.

---

## 8. Render pipeline (per frame)

```
1.  scheduler invokes hmi.render task at 30 Hz
2.  read latest frames from capture/* (in-process pub/sub, latest QoS)
3.  read latest calibration (TripleBuffer in calibration module)
4.  read latest perception/planner/vehicle/pdc snapshots
5.  read latest view_mode + haptic input
6.  for each active subview (single | split):
       a. render the view (BEV / pano / curbstone / single / floating_3d)
       b. apply blind-spot fill if applicable
       c. composite overlays (steering tube, ghost car, slots, PDC)
7.  composite subviews into final framebuffer
8.  submit framebuffer to display
9.  emit metrics (frame_time_ms, dropped_frames) via diag + XCP
```

No allocation in steps 2–9 (all buffers pre-allocated at init). Bounded execution time.

---

## 9. Vulkan/OpenGL ES backend selection

Build-time and runtime:

- CMake option: `HMI_GRAPHICS_API=vulkan|gles32` (default `vulkan`)
- Runtime fallback: if Vulkan initialization fails (driver missing, version too old), log error and exit (no silent fallback). Fallback is build-time, not runtime.

Shader cross-compilation:
- GLSL source written once
- `glslc` compiles to SPIR-V (Vulkan target)
- `glslc` also compiles to ESSL 3.20 (OpenGL ES 3.2 target) via cross-output

---

## 10. Display path

DRM/KMS direct submission on QNX and Linux (via libdrm). Configurable device + connector via config.

Alternative paths supported but out of v1 scope: Wayland, OEM display compositor protocols.

---

## 11. Camera ISP teeing

The camera ISP path must tee raw fisheye frames to both partitions (ASIL-B perception, QM HMI). Standard MIPI CSI-2 / V4L2 multi-consumer routing under QNX. Documented and supported on 8650P+ Ride Flex.

The HMI application opens V4L2 devices directly from its QM partition; the hypervisor / device mapping makes the cameras visible to both partitions. Per-camera capture task in the HMI application uses `v4l2_buffer` mmap mode for zero-copy DMA.

---

## 12. Testing strategy

| Layer | Tool |
|---|---|
| Unit | GoogleTest + `ADAS_Framework::test::FrameworkFixture` |
| Integration | GoogleTest with `MockTransport` simulating perception/planner inputs |
| Golden-frame regression | Custom harness in `tools/golden/` with canned camera inputs and reference framebuffer hashes; PNG-perceptual diff |
| Performance | `ADAS_Framework` GoogleBenchmark hooks; per-view render-time benchmark |
| Soak | Multi-hour run scripted in `tools/soak/`; CI nightly |

---

## 13. Cross-references

- Functional spec: `design.md`
- Operational rules: `GEMINI.md`
- Phase plan: `instruction.md`
- Naming convention: `docs/naming-convention.md` (copied from framework; canonical version is `ADAS_Framework/docs/naming-convention.md`)
- Project roadmap: `ADAS_Perception_Startup_Roadmap.md` §3.19
- Framework: `01-framework-ADAS_Framework/`
- Integration: `03-integration-ADAS_Integration/`
