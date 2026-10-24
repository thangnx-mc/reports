# Investigation & Fix Report: Replay Mouse Coordinate Drift

**Date:** 2026-10-24 15:30:00 UTC+7  
**Project:** Standalone Windows Input Recorder (`C:\DEV\tools\recorder\`)  
**Scope:** Fix Replay Mouse Coordinate Drift & Coordinate Fidelity Quality Gate  

---

## 1. Executive Summary

A real-world test of the Replay feature revealed that upon starting Replay, the mouse cursor drifted or jumped near the right edge of the display before executing actions, causing subsequent clicks and movements to miss their recorded targets.

A rigorous, data-driven investigation of real recorded sessions (`session_20261024_080326_6800`, `session_20261024_080110_0171`, etc.) pinpointed the exact root cause:
1. **DPI Awareness Gap**: The recorder executable did not declare Windows Per-Monitor V2 DPI awareness. On a standard 1080p display scaled at 125% DPI (logical resolution 1536x864), low-level mouse hooks (`WH_MOUSE_LL`) captured true physical coordinates (e.g. `x = 1706`, `x = 1257`), but during Replay, `user32.SetCursorPos()` was called by a DPI-unaware process. Windows interpreted these physical coordinates as logical coordinates and multiplied them by the 1.25x scaling factor (e.g., `1706 * 1.25 = 2132.5` and `1257 * 1.25 = 1571.25`), clamping all coordinates exceeding 1536 to the monitor's right boundary (1920).
2. **Missing Initial Cursor Baseline**: Sessions did not sample the cursor position at `t = 0`. Relative movement packets from Raw Input accumulated from an arbitrary starting position (such as the GUI's "Replay" button on the right half of the window).

A comprehensive, backward-compatible fix was implemented. All 37 automated tests passed (including the new 6-scenario coordinate fidelity test suite), and the standalone binary was rebuilt at `dist/recorder/recorder.exe`. The sibling project `C:\DEV\tools\rb\` remained completely frozen and untouched.

---

## 2. Investigation & Root Cause Proof

### 2.1 Evidence from Real Session Manifests and Events
Inspection of actual session logs under `dist/recorder/datasets/sessions/session_20261024_080326_6800/` revealed:
- **Manifest Environment**:
  ```json
  "primary_screen": { "width": 1536, "height": 864 }
  ```
  `GetSystemMetrics(0)` returned 1536 because Windows virtualized the display for a DPI-unaware process on a physical 1920x1080 panel with 125% DPI scale.
- **Recorded Click Events**:
  ```json
  {"seq":145, "type":"mouse_button_down", "cursor":{"x":1173,"y":678}}
  {"seq":207, "type":"mouse_button_down", "cursor":{"x":1706,"y":353}}
  {"seq":410, "type":"mouse_button_down", "cursor":{"x":1680,"y":341}}
  ```
- **The Mathematical Anomaly**:
  `x = 1706` is greater than `primary_screen.width = 1536`. This directly proved that `WH_MOUSE_LL` hooks recorded true physical pixels, while the application thought its screen was only 1536 pixels wide.

### 2.2 Proof of Right-Edge Clamping
When a DPI-unaware process invokes `user32.SetCursorPos(cx, cy)`:
$$\text{physical\_x} = \text{logical\_x} \times 1.25$$
- For $x = 1706$: $\text{physical\_x} = 1706 \times 1.25 = 2132.5 > 1920 \implies \text{clamped to } 1920 \text{ (right edge)}$
- For $x = 1680$: $\text{physical\_x} = 1680 \times 1.25 = 2100.0 > 1920 \implies \text{clamped to } 1920 \text{ (right edge)}$
- For $x = 1257$: $\text{physical\_x} = 1257 \times 1.25 = 1571.25 > 1536 \implies \text{clamped to } 1920 \text{ (right edge)}$
- For $x = 1173$: $\text{physical\_x} = 1173 \times 1.25 = 1466.25 \implies \text{drifted right by } 293 \text{ pixels}$

Every click coordinate was either violently clamped to the monitor's right boundary (1920) or drifted rightwards by 25%.

### 2.3 Proof of Pre-Click Drift
In `session_20261024_080110_0171`:
- Between Event 0 and Event 53 (before the first click), the user moved the mouse rightwards:
  $$\sum_{i=0}^{53} dx = +482 \text{ mickeys}$$
- When starting Replay, the cursor was not initialized to the recorded baseline. If the user clicked "Replay ( K )" on the GUI, the cursor started on the right half of the window, and the $+482$ relative deltas pushed the cursor further towards the right edge before the first click even fired.

---

## 3. Architecture & Code Modifications

### 3.1 Per-Monitor V2 DPI Awareness (`src/windows/win32_defs.py`, `src/main.py`)
- Added `enable_dpi_awareness()` helper function attempting in priority order:
  1. `user32.SetProcessDpiAwarenessContext(DPI_AWARENESS_CONTEXT_PER_MONITOR_AWARE_V2)` (Windows 10 1703+)
  2. `shcore.SetProcessDpiAwareness(PROCESS_PER_MONITOR_DPI_AWARE)` (Windows 8.1+)
  3. `user32.SetProcessDPIAware()` (Vista+)
- Executed on module load and at the top of `main()`.
- Added System Metrics bindings and constants: `SM_XVIRTUALSCREEN (76)`, `SM_YVIRTUALSCREEN (77)`, `SM_CXVIRTUALSCREEN (78)`, `SM_CYVIRTUALSCREEN (79)`.

### 3.2 Virtual Desktop Clamping & Safe Cursor Positioning (`src/replay/input_injector.py`)
- Implemented `set_cursor_position(x, y)`:
  - Queries virtual desktop origin `(vx, vy)` and dimensions `(vw, vh)`.
  - Safely clamps coordinates: $x \in [vx, vx + vw - 1]$, $y \in [vy, vy + vh - 1]$.
  - Preserves negative coordinate origins in multi-monitor setups while preventing out-of-bounds errors.
- Refactored `inject_mouse_button()` to call `set_cursor_position()`.

### 3.3 Initial Cursor Baseline & Movement Synchronization (`src/session/session_manager.py`, `src/windows/raw_input_sink.py`)
- **Initial Baseline**: At the exact moment `start_recording()` is invoked, queries `GetCursorPos()` and emits a `cursor_pos` event at `seq: 0`. Also attaches `initial_cursor: {"x": x, "y": y}` and `dpi_awareness: "PerMonitorV2"` to manifest environment metadata.
- **Continuous Motion Synchronization**: In `RawInputSink._window_proc`, samples physical `GetCursorPos()` and attaches `cursor: CursorPos(x, y)` to `mouse_move_relative` events alongside `raw_delta`.

### 3.4 Replay Engine Routing (`src/replay/replay_engine.py`)
- Dispatches `cursor_pos` events to initialize cursor position at `t = 0`.
- For `mouse_move_relative`:
  - **3D Camera Look Mode** (Right Mouse Button held down): Injects raw relative displacement `inject_mouse_move_relative(dx, dy)`.
  - **Desktop UI Cursor Mode** (No RMB held, cursor coordinate present): Injects `set_cursor_position(cursor.x, cursor.y)`, guaranteeing exact pixel trajectory without acceleration drift.
  - **Legacy Fallback** (Old sessions without cursor field): Falls back to `inject_mouse_move_relative(dx, dy)`.

---

## 4. Backward Compatibility Contract

| Session Type | Initial Cursor Handling | Movement Replay | Click Replay |
| :--- | :--- | :--- | :--- |
| **New Sessions** (Recorded after fix) | Restored to exact physical `(x, y)` at `seq: 0` via `cursor_pos` event. | Desktop moves follow exact `(x, y)` path. 3D camera pan follows `(dx, dy)`. | Exact physical coordinates without DPI drift. |
| **Old Sessions** (Recorded before fix) | No spurious initial move inserted (strictly preserves recorded events). | Injects `(dx, dy)` relative motion. | Clicks land on exact physical coordinates without 1.25x multiplication or right-edge clamping. |

*Limitation Transparency*: In old sessions, pre-click relative trajectories start from wherever the cursor happens to be because initial cursor position was not captured. However, all click events in old sessions are restored to their true physical targets without DPI clamping.

---

## 5. Verification & Test Suite

### 5.1 Automated Test Execution
Created `tests/test_mouse_coordinate_fidelity.py` covering all 6 mandatory requirements. Full test execution (37 of 37 passed):

```text
============================= test session starts =============================
platform win32 -- Python 3.12.13, pytest-9.1.1, pluggy-1.7.0
rootdir: C:\DEV\tools
collected 37 items

..\recorder\tests\test_clock.py::test_perf_counter_monotonicity PASSED   [  2%]
..\recorder\tests\test_clock.py::test_iso_utc_format PASSED              [  5%]
..\recorder\tests\test_clock.py::test_compute_delta_ms PASSED            [  8%]
..\recorder\tests\test_event_models.py::test_serialize_key_down PASSED   [ 10%]
..\recorder\tests\test_event_models.py::test_serialize_mouse_move_relative PASSED [ 13%]
..\recorder\tests\test_event_models.py::test_serialize_mouse_move_absolute PASSED [ 16%]
..\recorder\tests\test_event_models.py::test_serialize_mouse_button PASSED [ 18%]
..\recorder\tests\test_event_models.py::test_session_manifest_defaults PASSED [ 21%]
..\recorder\tests\test_live_win32_integration.py::test_hook_manager_callback_integration PASSED [ 24%]
..\recorder\tests\test_mouse_coordinate_fidelity.py::test_1_relative_movement_not_misinterpreted_as_absolute PASSED [ 27%]
..\recorder\tests\test_mouse_coordinate_fidelity.py::test_2_replay_does_not_jump_to_right_edge_on_start PASSED [ 29%]
..\recorder\tests\test_mouse_coordinate_fidelity.py::test_3_click_position_reproduced_accurately PASSED [ 32%]
..\recorder\tests\test_mouse_coordinate_fidelity.py::test_4_movement_and_click_sequence_preserves_order PASSED [ 35%]
..\recorder\tests\test_mouse_coordinate_fidelity.py::test_5_backward_compatibility_with_old_sessions PASSED [ 37%]
..\recorder\tests\test_mouse_coordinate_fidelity.py::test_6_stop_replay_and_error_handling_safety PASSED [ 40%]
..\recorder\tests\test_queue_writer.py::test_queue_writer_normal_lifecycle PASSED [ 43%]
..\recorder\tests\test_queue_writer.py::test_queue_overflow_drop_policy PASSED [ 45%]
..\recorder\tests\test_replay.py::test_session_selector_valid_and_invalid PASSED [ 48%]
..\recorder\tests\test_replay.py::test_input_injector_tracking_and_release_all PASSED [ 51%]
..\recorder\tests\test_replay.py::test_replay_engine_execution_and_cancellation PASSED [ 54%]
..\recorder\tests\test_replay.py::test_replay_engine_unsupported_events PASSED [ 56%]
..\recorder\tests\test_replay.py::test_session_manager_mutual_exclusivity PASSED [ 59%]
..\recorder\tests\test_replay.py::test_hook_manager_hotkey_k_and_injected_filtering PASSED [ 62%]
..\recorder\tests\test_session_lifecycle.py::test_session_state_machine PASSED [ 64%]
..\recorder\tests\test_state_tracker.py::test_key_down_and_auto_repeat PASSED [ 67%]
..\recorder\tests\test_state_tracker.py::test_modifier_disambiguation PASSED [ 70%]
..\recorder\tests\test_state_tracker.py::test_mouse_button_tracking PASSED [ 72%]
..\recorder\tests\test_state_tracker.py::test_focus_loss_reconciliation PASSED [ 75%]
..\recorder\tests\test_ui_telemetry.py::test_single_key_tap_counting PASSED [ 78%]
..\recorder\tests\test_ui_telemetry.py::test_single_mouse_click_counting PASSED [ 81%]
..\recorder\tests\test_ui_telemetry.py::test_auto_repeat_separation_if_present PASSED [ 83%]
..\recorder\tests\test_ui_telemetry.py::test_mixed_session_total_and_breakdown PASSED [ 86%]
..\recorder\tests\test_win32_defs.py::test_keyboard_hook_constants PASSED [ 89%]
..\recorder\tests\test_win32_defs.py::test_mouse_hook_constants PASSED   [ 91%]
..\recorder\tests\test_win32_defs.py::test_raw_mouse_constants PASSED    [ 94%]
..\recorder\tests\test_win32_defs.py::test_hotkey_virtual_keys PASSED    [ 97%]
..\recorder\tests\test_win32_defs.py::test_struct_sizes PASSED           [100%]

============================= 37 passed in 1.32s ==============================
```

### 5.2 Standalone Binary Rebuild
Compiled standalone executable via PyInstaller:
- **Executable Output**: `C:\DEV\tools\recorder\dist\recorder\recorder.exe`
- **Config Output**: `C:\DEV\tools\recorder\dist\recorder\config.json`
- **CLI Check**: Ran `recorder.exe --cli --record-seconds 1` and confirmed `dpi_awareness: 'PerMonitorV2'` with true physical resolution `(1920, 1080)`.

---

## 6. Safe Manual Verification Guide (Before Roblox)

Before testing in Roblox, the user can verify coordinate fidelity safely in Notepad or MS Paint:

### Test A: Notepad Focus & Click Verification
1. Open Windows Notepad and place it in the center or left half of the screen.
2. Launch `dist/recorder/recorder.exe`.
3. Press `;` to begin recording.
4. Move the cursor smoothly into the Notepad text area and click once.
5. Type `hello world`.
6. Press `'` to stop recording.
7. Click outside Notepad (e.g. on the desktop or taskbar).
8. Press `K` to start Replay.
9. **Expected Observation**: The cursor moves directly into Notepad and clicks the text area; `hello world` is typed into Notepad. The cursor does **not** jump to the right edge of the screen.

### Test B: Replay Cancellation Safety Check
1. Start Replay by pressing `K`.
2. Immediately press `K` again while replay is active.
3. **Expected Observation**: Replay cancels promptly, UI returns to `Idle`, and no keys or mouse buttons remain held down.
