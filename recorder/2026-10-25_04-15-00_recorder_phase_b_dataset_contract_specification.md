# RECORDER PHASE B — DATASET CONTRACT & ARCHITECTURE SPECIFICATION
**Version:** 1.0  
**Phase:** B — Dataset Contract & Architecture Specification (Documentation Only)  
**Date:** 2026-10-25 04:15:00 +07:00  
**Project root:** `C:\DEV\tools\recorder\`  
**Protected project:** `C:\DEV\tools\rb\` (Strictly Frozen)  
**Reports repository:** `https://github.com/thangnx-mc/reports`, branch `main`  
**Report path:** `recorder/2026-10-25_04-15-00_recorder_phase_b_dataset_contract_specification.md`  

---

## 1. Executive Summary & Design Decisions

### 1.1 Context and Mission
Following the Phase A Architecture & Dataset Readiness Audit (`recorder/2026-10-25_03-15-00_recorder_master_plan_phase_a_architecture_and_dataset_readiness_audit.md`), the standalone Input Recorder has achieved verified, high-fidelity input recording and deterministic replay on Windows 11 under Per-Monitor V2 DPI awareness `[FACT]`.

The objective of Phase B is to define the **Dataset Contract and Architecture Specification** for multimodal behavioral recording: combining serialized keyboard/mouse input events with time-synchronized screen frames to support downstream Imitation Learning (Behavioral Cloning, Action Chunking, and Diffusion Policy training) `[PROPOSAL]`.

Phase B is strictly **documentation-only**:
- Zero modifications to source files, tests, configurations (`config.json`), dependencies (`requirements.txt`), session datasets, or binaries (`dist/recorder/`) in `C:\DEV\tools\recorder\` `[FACT]`.
- Zero interaction with `C:\DEV\tools\rb\` `[FACT]`.

### 1.2 Evidence Classification Standard
Throughout this specification, statements are strictly tagged:
- `[FACT]`: Directly observed in the existing codebase, live test sessions, or Windows OS APIs.
- `[INFERENCE]`: Deduced from architectural analysis, queue dynamics, or performance characteristics.
- `[PROPOSAL]`: Recommended design, schema, or policy for future implementation phases.
- `[UNKNOWN]`: Requires empirical profiling, prototype measurement, or real-world testing.

### 1.3 Key Architectural Decisions Summary
1. **Decoupled Asynchronous Dual-Pipeline Architecture `[PROPOSAL]`**:
   Input ingress (`LowLevelKeyboardHook`, `LowLevelMouseHook`, and `RawInputSink`) and Screen Capture (`DXGI Desktop Duplication` / `Windows Graphics Capture`) run on isolated OS threads, feed separate ring/bounded queues, and flush through separate worker threads. A drop or slowdown in visual capture/encode/disk-write **must never** introduce jitter, drop, or delay in the Windows input hook callbacks.
2. **Strict Monotonic Timeline Synchronization `[PROPOSAL]`**:
   All events (input and frames) are timestamped using `time.perf_counter_ns()` `[FACT]`. Wall-clock UTC ISO 8601 timestamps are reserved solely for human-readable session boundaries `[PROPOSAL]`.
3. **Realistic Synchronization Contract `[PROPOSAL]`**:
   Visual capture cannot be instantaneous. Frame metadata records distinct timestamps: trigger time ($T_{\text{trigger}}$), hardware acquisition/readback time ($T_{\text{acquired}}$), encode completion time ($T_{\text{encoded}}$), and disk-write time ($T_{\text{written}}$). Frame metadata includes `latest_input_seq` as an atomic correlation pointer, explicitly acknowledging that it denotes the most recent serialized input event at or before $T_{\text{acquired}}$, without claiming microsecond simultaneity.
4. **Schema Versioning & Backward Compatibility `[PROPOSAL]`**:
   The manifest schema advances from `1.1.0` to `1.2.0`. Sessions lacking visual capture remain valid input datasets. Offline validators and ML export pipelines evaluate explicit `training_eligibility` criteria without breaking legacy replay consumers.
5. **Storage Budget & Hard Quota Safety `[PROPOSAL]`**:
   Target capture rate is default 10 FPS (configurable 5–30 FPS) at JPEG Q80 or WebP Q80. A session storage quota (default 2.0 GB) triggers an automatic graceful capture stop while permitting input recording to finish cleanly without data corruption.

---

## 2. Deliverable A — Dataset Directory Contract

### 2.1 Proposed Directory Layout
The session directory contract preserves the proven v1.1 structure while cleanly nesting visual artifacts:

```text
C:\DEV\tools\recorder\datasets\sessions\session_<YYYYMMDD_HHMMSS_RAND>/
├── manifest.json       # Session metadata, integrity, metrics, capture configuration
├── events.jsonl        # Input events stream (JSON Lines format)
├── frames.jsonl        # Visual frames metadata & temporal synchronization stream
└── frames/             # Raw image files (flat directory)
    ├── frame_000001_1368927500000.jpg
    ├── frame_000002_1368927600000.jpg
    └── ...
```

### 2.2 File Roles and Isolation Principles
1. **`manifest.json` `[PROPOSAL]`**:
   - Single point of truth for session lifecycle, hardware/OS environment, total counts, aggregate latency metrics, integrity validation flags, and ML training eligibility.
   - Written partially at session startup (`status: "incomplete"`), finalized and flushed atomically at session stop (`status: "complete"`).
2. **`events.jsonl` `[FACT]`**:
   - Serialized stream of keyboard, mouse, and focus events.
   - Isolated from visual capture: disk write errors in `frames/` or buffer overflow in the frame queue **never** alter or append to `events.jsonl`.
3. **`frames.jsonl` `[PROPOSAL]`**:
   - Dedicated index of visual frames.
   - Each line represents one attempted frame capture cycle, recording timestamp telemetry, relative file paths, and failure/drop diagnostics.
   - Stored in append-only JSONL format to guarantee recovery even if the recording process crashes mid-session.
4. **`frames/` Directory `[PROPOSAL]`**:
   - Flat directory containing encoded frame files (e.g., JPEG or WebP).
   - Filename convention: `frame_{frame_seq:06d}_{capture_complete_ns}.{ext}`.
   - Storing `capture_complete_ns` in the filename allows quick manual auditing and prevents collision.

### 2.3 Session State Classification Rules
An offline validator, replay runner, or ML dataset builder classifies any session into one of four distinct states based on file presence and manifest content:

| Session Category | File System State | Manifest Indicators | Consumer Action |
|---|---|---|---|
| **Legacy Input-Only (v1.1.0)** | `manifest.json` present<br>`events.jsonl` present<br>`frames.jsonl` absent<br>`frames/` absent | `schema_version`: `"1.1.0"`<br>`capture`: absent or `null`<br>`status`: `"complete"` | Valid for Input Replay `[FACT]`. Ineligible for visual imitation learning `[PROPOSAL]`. |
| **Complete Multimodal (v1.2.0)** | `manifest.json` present<br>`events.jsonl` present<br>`frames.jsonl` present<br>`frames/` populated | `schema_version`: `"1.2.0"`<br>`capture.enabled`: `true`<br>`status`: `"complete"`<br>`integrity.integrity_status`: `"VALID"` | Fully eligible for ML training and visual replay validation `[PROPOSAL]`. |
| **Complete Input / Capture Failed** | `manifest.json` present<br>`events.jsonl` present<br>`frames.jsonl` present or empty<br>`frames/` empty or missing files | `schema_version`: `"1.2.0"`<br>`capture.enabled`: `true`<br>`capture.frames_captured`: `0`<br>`capture.stop_reason`: error code | Valid for Input Replay `[PROPOSAL]`. Flagged as `training_eligibility.eligible = false` with explicit failure reason `[PROPOSAL]`. |
| **Incomplete / Interrupted Session** | Any files present or partial | `status`: `"incomplete"` or manifest missing/corrupted | Rejected by Replay and ML export until offline recovery is executed `[FACT]`. |

---

## 3. Deliverable B — Schema Specification

### 3.1 `manifest.json` Schema (v1.2.0)

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "title": "RecorderSessionManifest_v1_2_0",
  "type": "object",
  "required": [
    "schema_version",
    "session_id",
    "status",
    "session_start_iso_utc",
    "session_end_iso_utc",
    "session_start_perf_ns",
    "session_end_perf_ns",
    "duration_seconds",
    "environment",
    "metrics",
    "integrity",
    "replay_contract"
  ],
  "properties": {
    "schema_version": { "type": "string", "enum": ["1.2.0"] },
    "session_id": { "type": "string", "pattern": "^session_[0-9]{8}_[0-9]{6}_[0-9]{4}$" },
    "status": { "type": "string", "enum": ["incomplete", "complete", "complete_input_only", "error"] },
    "session_start_iso_utc": { "type": "string", "format": "date-time" },
    "session_end_iso_utc": { "type": "string", "format": "date-time" },
    "session_start_perf_ns": { "type": "integer", "minimum": 0 },
    "session_end_perf_ns": { "type": "integer", "minimum": 0 },
    "duration_seconds": { "type": "number", "minimum": 0.0 },
    "environment": {
      "type": "object",
      "required": [
        "os", "architecture", "python_version", "keyboard_layout_tested",
        "target_process_filtered", "target_process_name", "dpi_awareness",
        "initial_cursor", "primary_screen", "virtual_desktop"
      ],
      "properties": {
        "os": { "type": "string" },
        "architecture": { "type": "string" },
        "python_version": { "type": "string" },
        "keyboard_layout_tested": { "type": "string" },
        "target_process_filtered": { "type": "boolean" },
        "target_process_name": { "type": "string" },
        "dpi_awareness": { "type": "string" },
        "initial_cursor": {
          "type": "object",
          "required": ["x", "y"],
          "properties": { "x": { "type": "integer" }, "y": { "type": "integer" } }
        },
        "primary_screen": {
          "type": "object",
          "required": ["width", "height"],
          "properties": { "width": { "type": "integer" }, "height": { "type": "integer" } }
        },
        "virtual_desktop": {
          "type": "object",
          "required": ["left", "top", "width", "height"],
          "properties": {
            "left": { "type": "integer" },
            "top": { "type": "integer" },
            "width": { "type": "integer" },
            "height": { "type": "integer" }
          }
        }
      }
    },
    "capture": {
      "type": "object",
      "description": "Visual screen capture configuration and execution telemetry",
      "required": [
        "enabled", "backend", "target_mode", "target_fps", "actual_fps",
        "capture_resolution", "saved_resolution", "format", "compression_quality",
        "frames_captured", "frames_dropped", "capture_errors", "write_errors",
        "latency_stats_ms", "stop_reason"
      ],
      "properties": {
        "enabled": { "type": "boolean" },
        "backend": { "type": "string", "enum": ["dxgi", "wgc", "gdi", "none"] },
        "target_mode": { "type": "string", "enum": ["desktop", "window", "region"] },
        "target_window_title": { "type": ["string", "null"] },
        "target_fps": { "type": "integer", "minimum": 1, "maximum": 60 },
        "actual_fps": { "type": "number", "minimum": 0.0 },
        "capture_resolution": {
          "type": "object",
          "required": ["width", "height"],
          "properties": { "width": { "type": "integer" }, "height": { "type": "integer" } }
        },
        "saved_resolution": {
          "type": "object",
          "required": ["width", "height"],
          "properties": { "width": { "type": "integer" }, "height": { "type": "integer" } }
        },
        "format": { "type": "string", "enum": ["jpeg", "webp", "png"] },
        "compression_quality": { "type": "integer", "minimum": 1, "maximum": 100 },
        "frames_captured": { "type": "integer", "minimum": 0 },
        "frames_dropped": { "type": "integer", "minimum": 0 },
        "capture_errors": { "type": "integer", "minimum": 0 },
        "write_errors": { "type": "integer", "minimum": 0 },
        "max_frame_queue_depth": { "type": "integer", "minimum": 0 },
        "total_storage_bytes": { "type": "integer", "minimum": 0 },
        "latency_stats_ms": {
          "type": "object",
          "required": ["mean_capture_ms", "p95_capture_ms", "mean_encode_ms", "mean_write_ms"],
          "properties": {
            "mean_capture_ms": { "type": "number" },
            "p95_capture_ms": { "type": "number" },
            "mean_encode_ms": { "type": "number" },
            "mean_write_ms": { "type": "number" }
          }
        },
        "stop_reason": {
          "type": "string",
          "enum": ["normal_stop", "storage_quota_exceeded", "backend_failure", "access_lost", "user_abort"]
        }
      }
    },
    "metrics": {
      "type": "object",
      "required": [
        "total_events_serialized", "event_breakdown", "dropped_events_count",
        "write_errors_count", "max_queue_depth_reached", "unreleased_inputs_at_stop"
      ],
      "properties": {
        "total_events_serialized": { "type": "integer" },
        "event_breakdown": { "type": "object", "additionalProperties": { "type": "integer" } },
        "dropped_events_count": { "type": "integer" },
        "write_errors_count": { "type": "integer" },
        "max_queue_depth_reached": { "type": "integer" },
        "unreleased_inputs_at_stop": {
          "type": "object",
          "required": ["keys", "buttons"],
          "properties": {
            "keys": { "type": "array", "items": { "type": "string" } },
            "buttons": { "type": "array", "items": { "type": "string" } }
          }
        }
      }
    },
    "integrity": {
      "type": "object",
      "required": ["flush_confirmed", "hook_failures", "clean_shutdown", "integrity_status"],
      "properties": {
        "flush_confirmed": { "type": "boolean" },
        "hook_failures": { "type": "integer" },
        "clean_shutdown": { "type": "boolean" },
        "integrity_status": { "type": "string", "enum": ["VALID", "CORRUPT", "DEGRADED"] }
      }
    },
    "training_eligibility": {
      "type": "object",
      "description": "Gate evaluation for feeding session into ML imitation pipelines",
      "required": ["eligible", "reasons"],
      "properties": {
        "eligible": { "type": "boolean" },
        "reasons": { "type": "array", "items": { "type": "string" } }
      }
    },
    "replay_contract": {
      "type": "object",
      "required": ["requires_release_all_on_start", "requires_release_all_on_stop"],
      "properties": {
        "requires_release_all_on_start": { "type": "boolean" },
        "requires_release_all_on_stop": { "type": "boolean" }
      }
    }
  }
}
```

### 3.2 `frames.jsonl` Record Schema

Each line in `frames.jsonl` is an independent, valid JSON object representing one frame cycle:

```json
{
  "frame_seq": 1,
  "file": "frames/frame_000001_1368927500000.jpg",
  "status": "success",
  "capture_start_ns": 1368927495000,
  "capture_complete_ns": 1368927500000,
  "encode_complete_ns": 1368927503500,
  "write_complete_ns": 1368927505200,
  "capture_latency_ms": 5.0,
  "encode_latency_ms": 3.5,
  "write_latency_ms": 1.7,
  "latest_input_seq": 42,
  "width": 1920,
  "height": 1080,
  "format": "jpeg",
  "byte_size": 84120,
  "error": null
}
```

#### Field Specification Table: `frames.jsonl`

| Field | Type | Required | Units / Values | Description & Rules | Null Condition |
|---|---|---|---|---|---|
| `frame_seq` | `integer` | Yes | 1-based index | Monotonically increasing sequence number per session. Must never skip or duplicate. | Never |
| `file` | `string` or `null` | Yes | Relative POSIX path | Relative path within the session directory (`frames/frame_000001_....jpg`). | `null` if frame was dropped or failed before write. |
| `status` | `string` | Yes | `"success"`, `"dropped"`, `"error"` | Outcome of the frame acquisition and write cycle. | Never |
| `capture_start_ns` | `integer` | Yes | Nanoseconds (`perf_counter_ns`) | Timestamp immediately prior to issuing frame grab to the capture API. | Never |
| `capture_complete_ns`| `integer` | Yes | Nanoseconds (`perf_counter_ns`) | Timestamp when the surface/texture buffer is safely acquired in CPU memory. | Never |
| `encode_complete_ns` | `integer` or `null`| No | Nanoseconds (`perf_counter_ns`) | Timestamp when image compression finishes. | `null` if frame dropped before encode. |
| `write_complete_ns` | `integer` or `null`| No | Nanoseconds (`perf_counter_ns`) | Timestamp when disk write finishes. | `null` if frame dropped before disk write. |
| `capture_latency_ms`| `number` | Yes | Milliseconds ($ms$) | Calculated as $(T_{\text{complete}} - T_{\text{start}}) / 10^6$. | Never |
| `encode_latency_ms` | `number` or `null`| No | Milliseconds ($ms$) | Calculated as $(T_{\text{encoded}} - T_{\text{complete}}) / 10^6$. | `null` if dropped |
| `write_latency_ms` | `number` or `null`| No | Milliseconds ($ms$) | Calculated as $(T_{\text{written}} - T_{\text{encoded}}) / 10^6$. | `null` if dropped |
| `latest_input_seq` | `integer` or `null`| Yes | Sequence ID | The sequence ID of the latest event in `events.jsonl` at timestamp $T_{\text{complete}}$. | `null` if no input events have occurred yet (`seq < 0`). |
| `width` | `integer` | Yes | Pixels ($px$) | Saved horizontal image resolution. | Must match actual image header. |
| `height` | `integer` | Yes | Pixels ($px$) | Saved vertical image resolution. | Must match actual image header. |
| `format` | `string` | Yes | `"jpeg"`, `"webp"`, `"png"` | Image encoding format. | Never |
| `byte_size` | `integer` | Yes | Bytes | On-disk file size of the image. | `0` if dropped. |
| `error` | `string` or `null` | Yes | Error string or `null` | Diagnostic reason if `status != "success"` (e.g. `"queue_full_oldest_dropped"`, `"DXGI_ERROR_ACCESS_LOST"`). | `null` on success. |

---

## 4. Deliverable C — Temporal Synchronization Contract

### 4.1 Clock Reference & Monotonic Origin
- The unified timeline is governed exclusively by Python's `time.perf_counter_ns()` `[FACT]`.
- Windows maps `perf_counter` directly to `QueryPerformanceCounter` (QPC), backed by the CPU Invariant TSC (Time Stamp Counter) `[FACT]`.
- `events.jsonl` records `timestamp_ns` via `perf_counter_ns()` `[FACT]`.
- `frames.jsonl` records all lifecycle stages using the same `perf_counter_ns()` `[PROPOSAL]`.
- **Wall-clock UTC** (`session_start_iso_utc`) is recorded strictly for human indexing and session directory naming `[FACT]`. Wall-clock time **must never** be used to calculate delta times or pair inputs with frames, as NTP adjustments, leap seconds, and daylight savings introduce non-monotonic discontinuities `[INFERENCE]`.

### 4.2 Timeline Lifecycle Stages
Visual frame acquisition is fundamentally an interval operation, unlike an input event which can be treated as a point event.

```text
Input Pipeline:
[Event Hook Callback] ──> T_input (perf_counter_ns) ──> [Queue Push] ──> [Disk Writer]

Frame Pipeline:
T_trigger ──> [DXGI AcquireNextFrame / CopySurface] ──> T_acquired ──> [Queue Push]
                                                            │
                                                            ▼ (Asynchronous Worker)
                                            T_encoded ──> [Encode JPEG]
                                                            │
                                                            ▼
                                            T_written ──> [Write File] ──> [Append frames.jsonl]
```

1. **Trigger Time ($T_{\text{trigger}}$)**:
   The timestamp immediately before invoking the capture API. In periodic capture, this corresponds to the timer tick.
2. **Hardware Acquisition Time ($T_{\text{acquired}}$)**:
   The timestamp when the GPU texture has been resolved, copied to the CPU-accessible staging surface, and mapped into RAM. **This is the true visual reference timestamp for synchronization.**
3. **Encode Completion Time ($T_{\text{encoded}}$)**:
   The timestamp when TurboJPEG or Pillow finishes compressing the RGB buffer into JPEG/WebP bytes.
4. **Disk Write Completion Time ($T_{\text{written}}$)**:
   The timestamp when the compressed bytes are flushed to the OS file cache / disk.

### 4.3 Mathematical Latency Definitions
$$\text{capture\_latency\_ms} = \frac{T_{\text{acquired}} - T_{\text{trigger}}}{1\,000\,000}$$
$$\text{encode\_latency\_ms} = \frac{T_{\text{encoded}} - T_{\text{acquired}}}{1\,000\,000}$$
$$\text{write\_latency\_ms} = \frac{T_{\text{written}} - T_{\text{encoded}}}{1\,000\,000}$$
$$\text{total\_pipeline\_latency\_ms} = \frac{T_{\text{written}} - T_{\text{trigger}}}{1\,000\,000}$$

### 4.4 Disambiguation of `latest_input_seq`
- **Definition `[PROPOSAL]`**: `latest_input_seq` is obtained by an atomic memory read of an internal sequence counter maintained by the input pipeline at the moment $T_{\text{acquired}}$ is reached.
- **Strict Limitation `[PROPOSAL]`**:
  `latest_input_seq` does **not** assert that the input event happened simultaneously with the screen presentation.
  Because OS display pipelines (DWM compositing, monitor refresh rates, frame buffers) introduce 8–25 ms of display lag, while input hooks execute at 1–2 ms latency, claiming exact concurrency is a technical fallacy.
  `latest_input_seq` serves purely as an indexing anchor to determine the latest user action registered by the OS before the frame buffer was retrieved.

### 4.5 Frame-to-Input Pairing Policy for ML Imitation Learning
For training behavioral cloning models (such as ACT, Diffusion Policy, or Video-Action Transformers):
- The model policy predicts an action chunk $A_t = [a_t, a_{t+1}, \dots, a_{t+H}]$ given an observation image $O_t$ observed at frame $k$.
- The observation state timestamp is assigned as $T_{\text{acquired}, k}$.
- The active input window associated with observation $O_t$ spans the interval:
  $$[T_{\text{acquired}, k},\; T_{\text{acquired}, k+1})$$
- Input events falling strictly inside this temporal window represent the human reactions and commands executed while perceiving screen state $O_t$.
- By storing exact nanosecond timestamps on both streams, offline dataset exporters can adjust alignment offsets (e.g., negative shift for human reaction time or positive shift for monitor presentation lag) dynamically without modifying raw recorded files `[PROPOSAL]`.

### 4.6 Synchronization Precision Bounds
- **Temporal Resolution**: 1 nanosecond (integer counter) `[FACT]`.
- **Ingress Sampling Uncertainty**:
  - Raw Input mouse: $\pm 1.0\text{ ms}$ (at 1000 Hz polling rate) `[FACT]`.
  - Windows low-level hooks: $\pm 2.0\text{ ms}$ (bounded by OS dispatch latency) `[INFERENCE]`.
  - DXGI Desktop Duplication: $\pm 4.0\text{ ms}$ (GPU copy and DWM sync) `[INFERENCE]`.
- **Claimed Dataset Accuracy `[PROPOSAL]`**:
  The dataset specification guarantees monotonic ordering and temporal alignment accuracy within $\pm 10\text{ ms}$ under standard 60 Hz desktop conditions. Sub-millisecond synchronization claims are explicitly disclaimed as unprovable without external hardware sensor instrumentation `[PROPOSAL]`.

---

## 5. Deliverable D — Capture Policy & Privacy

### 5.1 Capture Backend Evaluation

| Backend Option | Mechanism | Latency Profile | Window-Only Capability | Pros & Cons | Feasibility / Status |
|---|---|---|---|---|---|
| **DXGI Desktop Duplication** | Direct3D 11 surface copy from Desktop Window Manager | Low (1–3 ms) `[INFERENCE]` | Full desktop/monitor only. Requires post-capture cropping for window targets. | High performance, direct GPU access. Fails during UAC/secure desktop transitions (`DXGI_ERROR_ACCESS_LOST`). | Recommended Primary for Desktop `[PROPOSAL]`. |
| **Windows Graphics Capture (WGC)** | WinRT `GraphicsCaptureItem` (Windows 10 1803+) | Low–Medium (3–6 ms) `[INFERENCE]` | Native per-window and per-monitor capture. | Automatically clips occluding windows; no yellow border in modern Win 11 APIs. Requires WinRT Python bindings. | Recommended Primary for Target Window `[PROPOSAL]`. |
| **GDI `BitBlt` / `PrintWindow`** | Win32 GDI screen scraping | High (15–30 ms) `[INFERENCE]` | Native per-window via `PW_RENDERFULLCONTENT`. | Universal compatibility across all Windows versions. High CPU readback overhead; can miss hardware-accelerated DirectX surfaces. | Recommended Fallback `[PROPOSAL]`. |

### 5.2 Edge Cases and Target Scenarios
1. **Windowed / Borderless / Exclusive Fullscreen `[PROPOSAL]`**:
   - Modern games (including Roblox) run borderless or windowed. WGC and DXGI capture these transparently.
   - For legacy exclusive fullscreen, DXGI Desktop Duplication captures the output adapter directly.
2. **Occlusion, Minimization, and Focus Loss `[PROPOSAL]`**:
   - If the target window is minimized, WGC freezes updates or outputs black frames; GDI `PrintWindow` fails or produces stale bitmaps.
   - Policy: The capture engine detects window minimization via `IsIconic(hwnd)` or zero-size rectangle, sets frame `status: "occluded"`, drops frame image generation, and increments `frames_dropped`. Input recording continues uninterrupted.
3. **DPI Awareness and Resolution Switching `[PROPOSAL]`**:
   - The standalone recorder already enforces `PerMonitorV2` DPI awareness `[FACT]`.
   - All capture coordinates and frame dimensions are recorded in unscaled physical screen pixels, ensuring 1:1 mathematical alignment with `events.jsonl` mouse cursor coordinates `[FACT]`.
   - If the display mode or monitor resolution changes mid-recording, DXGI returns `DXGI_ERROR_ACCESS_LOST`. The engine initiates an automatic teardown and reconnect cycle, logging a `reconnect` event in `frames.jsonl`.

### 5.3 Privacy & Data Governance Policy
1. **Strict Window Bounding `[PROPOSAL]`**:
   - When configured in `target_mode: "window"`, the recorder restricts capture strictly to the target application's client rect. Secondary monitors and background windows (browsers, chat apps) are excluded.
2. **User Consent & Visual Feedback `[PROPOSAL]`**:
   - Full desktop recording requires an explicit confirmation or clear indicator in the UI.
   - The UI status bar must display an active capture indicator (e.g. `"REC [CAM + INP]"`).
3. **Emergency Capture Abort `[PROPOSAL]`**:
   - In addition to the global Stop hotkey (`'`), the UI provides a dedicated button: **"Abort Capture (Keep Inputs Only)"**.
   - If pressed, the visual capture engine flushes and stops instantly, while input recording proceeds.
4. **Zero Intentional PII Storage `[PROPOSAL]`**:
   - Manifest files must never record Windows user account names, personal folder paths, or machine MAC addresses. Paths are strictly relative.

---

## 6. Deliverable E — Buffer, Drop Policy & Storage Budget

### 6.1 Architecture Isolation
The input pipeline and visual capture pipeline must remain completely isolated:

```text
┌────────────────────────────────────────────────────────┐
│                   INPUT PIPELINE                       │
│  Hooks / RawInput  ──>  Queue (10k items)  ──>  Disk   │
└────────────────────────────────────────────────────────┘
                           ▲ NO BLOCKING
                           │ NO SHARED LOCKS
┌────────────────────────────────────────────────────────┐
│                  CAPTURE PIPELINE                      │
│  Capture Engine    ──>  Queue (30 frames)  ──>  Disk   │
└────────────────────────────────────────────────────────┘
```

1. **Queue Isolation**: The `QueueWriter` for inputs and the `FrameWriter` for frames utilize completely independent `queue.Queue` instances and worker threads `[PROPOSAL]`.
2. **Zero Ingress Degradation**: Visual capture, compression, or disk-write operations are strictly prohibited from holding locks or invoking synchronous calls inside the Windows hook callbacks `[PROPOSAL]`.
3. **Independent Failure Handling**: If `frames/` disk writes fail due to permission errors or slow storage, the input pipeline writes `events.jsonl` without delay or error `[PROPOSAL]`.

### 6.2 Drop Policy and Buffer Overflow Handling
1. **Frame Queue Bounding**: The in-memory frame queue is bounded to a maximum of 30 frames (representing 3 seconds of buffer at 10 FPS) to prevent RAM exhaustion `[PROPOSAL]`.
2. **Drop Strategy**: If the frame queue reaches capacity (e.g. disk write stall), the oldest unwritten frame is popped and discarded (`"queue_full_oldest_dropped"`), and `frames_dropped` is incremented `[PROPOSAL]`.
3. **Zero Silent Drops**: Every discarded frame produces a record in `frames.jsonl` with `file: null`, `status: "dropped"`, and `error: "queue_overflow"`. The manifest records the total `frames_dropped` count `[PROPOSAL]`.
4. **Input Priority Rule**: Under no circumstances may an input event be dropped to conserve resources for image writing `[PROPOSAL]`.

### 6.3 Storage Budget Formula & Scenarios
Storage consumption is calculated as:
$$\text{Storage Rate (MB/min)} = \text{FPS} \times 60 \times \left(\frac{\text{Average Frame Size in KB}}{1024}\right)$$

#### Theoretical Storage Projections (1080p Desktop)

| Scenario | Resolution | FPS | Format & Quality | Est. Frame Size | Storage Rate | 5 min Session | 15 min Session | 30 min Session |
|---|---|---|---|---|---|---|---|---|
| **Low Overhead** | 1280x720 | 5 FPS | JPEG Q75 | ~35 KB `[INFERENCE]` | 10.3 MB/min | 51.5 MB | 154.5 MB | 309.0 MB |
| **Balanced (Default)**| 1920x1080 | 10 FPS | JPEG Q80 | ~95 KB `[INFERENCE]` | 55.7 MB/min | 278.5 MB | 835.5 MB | 1,671.0 MB |
| **High Fidelity** | 1920x1080 | 20 FPS | JPEG Q85 | ~140 KB `[INFERENCE]` | 164.1 MB/min | 820.5 MB | 2,461.5 MB | 4,923.0 MB |
| **WebP Efficient** | 1920x1080 | 10 FPS | WebP Q80 | ~65 KB `[INFERENCE]` | 38.1 MB/min | 190.5 MB | 571.5 MB | 1,143.0 MB |

*Note: Frame sizes are theoretical engineering estimates based on typical 3D game scene entropy `[INFERENCE]`. Exact empirical sizes will be measured in Phase C profiling `[UNKNOWN]`.*

### 6.4 Hard Storage Quota Safety Rule
- **Default Session Storage Quota**: 2,048 MB (2.0 GB) `[PROPOSAL]`.
- **Enforcement Policy**:
  When total session disk usage exceeds the configured quota:
  1. The frame capture engine logs `stop_reason: "storage_quota_exceeded"` and shuts down gracefully `[PROPOSAL]`.
  2. The frame worker flushes remaining queue items and closes `frames.jsonl` `[PROPOSAL]`.
  3. The input recording pipeline remains **fully active**, continuing to capture keyboard and mouse events until the user presses the Stop hotkey (`'`) `[PROPOSAL]`.
  4. The manifest records `status: "complete_input_only"` or `capture.stop_reason: "storage_quota_exceeded"` `[PROPOSAL]`.

---

## 7. Deliverable F — Versioning & Backward Compatibility

### 7.1 Schema Evolution Rules
- **Semantic Versioning**: Manifests and dataset schemas adhere to `MAJOR.MINOR.PATCH` `[PROPOSAL]`.
  - `v1.1.0`: Current production input-only schema `[FACT]`.
  - `v1.2.0`: Multimodal input + frame capture schema `[PROPOSAL]`.
- **Additive Compatibility**: All new fields in v1.2.0 (`capture`, `training_eligibility`) are optional additions to the top-level manifest. Consumers written for v1.1.0 can safely parse a v1.2.0 manifest using lenient JSON deserializers `[PROPOSAL]`.

### 7.2 Backward Compatibility with Existing Datasets
1. **Existing Sessions Untouched**: Real user sessions in `dist/recorder/datasets/sessions/` (e.g., `session_20261024_152713_3799`) have schema version `"1.1.0"` `[FACT]`.
2. **Replay Engine Invariance**: The replay subsystem (`src/replay/replay_engine.py`, `session_selector.py`) inspects only `events.jsonl` and input metrics `[FACT]`. It does not require `frames.jsonl` or image files. A v1.1.0 session replays identically to a v1.2.0 session `[FACT]`.
3. **Consumer Fallback**: If an ML dataset builder encounters a v1.1.0 session, it immediately flags `eligible: false, reasons: ["legacy_v1_1_input_only_session"]` without raising an unhandled exception `[PROPOSAL]`.

### 7.3 ML Training Eligibility Gate
An offline validator evaluates the following strict rule set to declare `training_eligibility.eligible = true`:
1. `manifest.status == "complete"` `[PROPOSAL]`.
2. `manifest.integrity.integrity_status == "VALID"` and `flush_confirmed == true` `[PROPOSAL]`.
3. `manifest.capture.enabled == true` `[PROPOSAL]`.
4. `manifest.capture.frames_captured >= 100` (rejects trivial or empty tests) `[PROPOSAL]`.
5. Frame Drop Ratio $\le 5.0\%$:
   $$\frac{\text{frames\_dropped}}{\text{frames\_captured} + \text{frames\_dropped}} \le 0.05$$
6. Input Drop Count $== 0$ (`metrics.dropped_events_count == 0`) `[PROPOSAL]`.
7. Zero missing files: Every `success` record in `frames.jsonl` must have a corresponding, non-empty image file in `frames/` matching its recorded `byte_size` `[PROPOSAL]`.

---

## 8. Deliverable G — Architecture & Minimal-Diff Plan

### 8.1 Module Architecture Boundary
To guarantee that the working Record and Replay systems remain 100% stable, all visual capture, validation, and export logic is partitioned into dedicated packages:

```text
src/
├── core/                  # [PROTECTED CORE] Clock, event models, queue writer, win32 defs
├── replay/                # [PROTECTED CORE] Replay engine, session selector, state reconciler
├── session/               # [PROTECTED CORE] Session manager, state tracker
├── ui/                    # [PROTECTED CORE] Tkinter GUI app, hotkey manager
├── capture/               # [NEW MODULE - Phase C] Capture backends, frame models, frame writer
├── validation/            # [NEW MODULE - Phase E] Offline integrity & frame sync validator
└── export/                # [NEW MODULE - Phase F] HDF5 / WebDataset converter for ML training
```

### 8.2 Minimal-Diff Plan Across Master Plan Phases

| File | Add/Modify | Phase | Technical Justification | Can Modification Be Avoided? | Targeted Regression Test |
|---|---|---|---|---|---|
| `src/capture/__init__.py` | **Add** | Phase C | Package initializer for capture engine. | N/A (New file) | Unit test package import. |
| `src/capture/capture_models.py` | **Add** | Phase C | Data classes for `FrameEvent`, `FrameManifestConfig`, and serialization contracts. | N/A (New file) | `tests/test_capture_models.py` |
| `src/capture/capture_engine.py` | **Add** | Phase C | Implementation of DXGI Desktop Duplication & GDI fallback capture loops. | N/A (New file) | `tests/test_capture_engine_mock.py` |
| `src/capture/frame_writer.py` | **Add** | Phase C | Asynchronous worker thread compressing frames and writing `frames.jsonl` and `frames/`. | N/A (New file) | `tests/test_frame_writer.py` |
| `src/core/event_models.py` | **Modify** | Phase D | Add optional `capture` and `training_eligibility` fields to `SessionManifest`. | **No**, if manifest schema is kept unified in `event_models.py`. **Yes**, if subclassed in `capture_models.py`. | `tests/test_event_models.py` (verify v1.1.0 manifest serialization remains unchanged). |
| `src/session/session_manager.py` | **Modify** | Phase D | Hook capture engine start/stop into session lifecycle (`start_session` and `stop_session`). | **Yes**, by wrapping `SessionManager` in a `CompositeRecorderManager` or using observer callbacks. | `tests/test_session_lifecycle.py` |
| `src/ui/app.py` | **Modify** | Phase D | Display visual capture status pill and capture telemetry metrics in GUI. | **Yes**, by keeping capture headless until Phase D integration. | `tests/test_ui_telemetry.py` |
| `config.json` | **Modify** | Phase D | Add `"capture"` configuration section (defaults disabled or 10 FPS). | **Yes**, by supplying fallback defaults inside `capture_models.py` if missing. | Verify recorder loads seamlessly with legacy `config.json`. |
| `src/validation/session_validator.py` | **Add** | Phase E | Read-only offline validation tool verifying frame continuity and timestamp monotonicity. | N/A (New file) | `tests/test_session_validator.py` |
| `src/export/dataset_exporter.py` | **Add** | Phase F | Offline conversion utility packaging sessions into HDF5/WebDataset for ML training. | N/A (New file) | `tests/test_dataset_exporter.py` |
| `src/core/clock.py` | **Do Not Touch** | — | `perf_counter_ns` is already optimal. | **Avoided** | N/A |
| `src/core/queue_writer.py` | **Do Not Touch** | — | Input serialization queue must remain untouched. | **Avoided** | `tests/test_queue_writer.py` |
| `src/windows/hook_manager.py` | **Do Not Touch** | — | Low-level input hooks must remain untouched. | **Avoided** | `tests/test_live_win32_integration.py` |
| `src/windows/raw_input_sink.py` | **Do Not Touch** | — | Mouse movement ingress must remain untouched. | **Avoided** | `tests/test_mouse_coordinate_fidelity.py` |
| `src/replay/*` | **Do Not Touch** | — | Replay engine must remain untouched. | **Avoided** | `tests/test_replay.py` |

---

## 9. Deliverable H — Quantitative Acceptance Criteria for Future Phases

### 9.1 Phase C: Screen Capture Prototype & Benchmarking
1. **Sustained Capture FPS**: The engine must sustain $10.0\text{ FPS} \pm 0.5\text{ FPS}$ on a 1920x1080 display under active desktop motion `[PROPOSAL]`.
2. **CPU Utilization**: Capture, JPEG encoding, and frame queueing must consume $\le 10.0\%$ average CPU on a modern 4-core+ CPU `[PROPOSAL]`.
3. **Capture Latency**: Mean hardware acquisition latency ($T_{\text{acquired}} - T_{\text{trigger}}$) must be $\le 10.0\text{ ms}$ on DXGI `[PROPOSAL]`.
4. **Isolated Memory Footprint**: Bounded frame queue must not exceed 150 MB RAM under steady-state write conditions `[PROPOSAL]`.

### 9.2 Phase D: Dual-Pipeline Synchronization & UI Integration
1. **Zero Input Hook Degradation**:
   - `dropped_events_count == 0` during active 10 FPS screen capture `[PROPOSAL]`.
   - Windows input hook processing time must remain $\le 1.0\text{ ms}$ per callback `[FACT]`.
2. **Timestamp Monotonicity**:
   - In `frames.jsonl`, $T_{\text{trigger}, k} < T_{\text{trigger}, k+1}$ for all consecutive frames `[PROPOSAL]`.
   - In `events.jsonl`, `timestamp_ns` remains strictly monotonically increasing `[FACT]`.
3. **Deterministic Replay Preservation**:
   - Running Replay on a session recorded with visual capture active must reproduce input actions with identical coordinate fidelity ($\Delta x = 0, \Delta y = 0$ drift) as a baseline input-only session `[PROPOSAL]`.
4. **Graceful Quota Handling**:
   - Exceeding the 2.0 GB quota must terminate visual capture cleanly while recording of inputs continues without error `[PROPOSAL]`.

### 9.3 Phase E: Dataset Validator & Quality Gate
1. **Missing File Detection**: The offline validator must report 100% of missing, zero-byte, or corrupted JPEG files in `frames/` `[PROPOSAL]`.
2. **Temporal Anomaly Flagging**: Any frame gap exceeding $2.5 \times (1 / \text{target\_fps})$ must be detected and flagged as a capture hiccup `[PROPOSAL]`.
3. **Gate Decision Determinism**: The validator must output identical boolean eligibility decisions on repeated runs `[PROPOSAL]`.

### 9.4 Phase F: ML Training Export
1. **Data Alignment Integrity**: Exporter must pair each frame observation $O_t$ with its corresponding input sequence chunk without temporal leakage (no future inputs mapped to past frames) `[PROPOSAL]`.
2. **Coordinate Normalization**: Mouse positions must be deterministically mapped to $[0.0, 1.0]$ floating-point space relative to client dimensions `[PROPOSAL]`.

---

## 10. Deliverable I — List of UNKNOWNs & Decisions Requiring User Approval

The following items cannot be established by static code analysis and represent empirical questions to be resolved during Phase C prototyping:

### 10.1 Empirical UNKNOWNs
1. `[UNKNOWN]`: **DXGI vs WGC in PyInstaller Bundles**:
   Does Windows Graphics Capture (via `winrt-Windows.Graphics.Capture` or `roapi`) introduce packaging or DLL dependency complications when frozen into a standalone PyInstaller EXE on Windows 11?
   *Resolution Plan*: Prototype both backends in isolated scratch scripts in Phase C before selecting the default.
2. `[UNKNOWN]`: **TurboJPEG vs Pillow Compression Latency**:
   What is the exact compression latency on 1080p RGB frames between `libjpeg-turbo` (via `PyTurboJPEG`) and standard `PIL.Image.save()`?
   *Resolution Plan*: Measure encoding time in Phase C. If `libjpeg-turbo` is required for $<5\text{ ms}$ latency, verify binary distribution licensing.
3. `[UNKNOWN]`: **Real Scene Entropy & Actual Disk Throughput**:
   What is the actual compressed frame size of Roblox 3D gameplay at 1080p JPEG Q80? (Estimated between 70 KB and 120 KB, but empirical validation is required).

### 10.2 Decisions Requiring Explicit User Approval Before Phase C
1. **Target Capture Mode Preference**:
   Should Phase C prioritize **Full Primary Desktop** capture (DXGI Desktop Duplication) or **Target Window** capture (Windows Graphics Capture targeting `RobloxPlayerBeta.exe`) as the primary backend?
   *(Recommendation: Primary Desktop via DXGI for simplest low-latency capture, with bounding crop to the target window).*
2. **Third-Party Compression Dependencies**:
   Is the user willing to introduce `PyTurboJPEG` (which requires `turbojpeg.dll`) for ultra-low latency image compression, or should the engine start with pure Python `Pillow` (PIL) to keep the packaging footprint minimal?
   *(Recommendation: Begin Phase C with `Pillow` to test baseline feasibility without external DLL dependencies).*
3. **Session Integration Strategy**:
   Does the user approve modifying `src/session/session_manager.py` to trigger capture lifecycle events, or should we implement an external composite wrapper (`src/capture/capture_manager.py`) to keep `session_manager.py` strictly untouched?
   *(Recommendation: An external observer/adapter pattern to keep existing core files frozen).*

---

## 11. Deliverable J — Confirmation of Strict Read-Only Execution

Antigravity certifies that Phase B was executed in strict adherence to the project boundaries:
1. **Target Project (`C:\DEV\tools\recorder\`)**:
   - Zero files created, modified, deleted, moved, or formatted.
   - `config.json`, `requirements.txt`, source code, tests, dataset sessions, and PyInstaller binaries remain 100% identical to the pre-task baseline `[FACT]`.
2. **Protected Project (`C:\DEV\tools\rb\`)**:
   - Zero files touched or inspected `[FACT]`.
3. **Testing and Build Execution**:
   - Zero test suites run, zero capture experiments executed, zero dependencies installed, zero executables rebuilt `[FACT]`.
4. **Reporting Repository (`https://github.com/thangnx-mc/reports`)**:
   - The only file created is this documentation report under `recorder/` on branch `main` `[FACT]`.

---

**STATUS: PHASE B SPECIFICATION COMPLETE. AWAITING USER REVIEW AND APPROVAL BEFORE COMMENCING PHASE C.**
