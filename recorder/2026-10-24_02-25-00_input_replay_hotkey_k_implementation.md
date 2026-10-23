# Input Replay Subsystem Implementation Report (Hotkey `K`)

**Date:** 2026-10-24 02:25:00 UTC+7  
**Project:** Antigravity Standalone Windows Input Recorder (`C:\DEV\tools\recorder`)  
**Scope:** Scope Extension — Input Replay with Global Hotkey `K` & Execution Engine  

---

## 1. Executive Summary

This scope extension adds hardware-level, timing-accurate input replay to the standalone Input Recorder project at `C:\DEV\tools\recorder\`. The replay subsystem reads recorded `events.jsonl` streams and synthesizes low-level input using the Win32 `SendInput` API while maintaining strict safety gates, sequence fidelity, relative delta-time delays, mutual exclusivity with recording, and a mandatory stuck-key release contract.

All automated tests passed (31 of 31 tests in `tests/`), and the standalone executable was rebuilt into `dist/recorder/recorder.exe`. The sibling project `C:\DEV\tools\rb\` remained completely frozen and untouched throughout the implementation.

---

## 2. Requirements & Verification Matrix

| Requirement | Implementation Detail | Status |
| :--- | :--- | :--- |
| **Global Replay Hotkey (`K`)** | `VK_K` (0x4B) registered in `HookManager`. Triggers replay of most recent valid session when IDLE/SAVED/REPLAYED. | **Verified** |
| **Cancellation via Hotkey (`K`)** | Pressing `K` while REPLAYING requests engine cancellation and releases all held inputs. | **Verified** |
| **Physical vs Injected Hotkey Filter** | `HookManager` checks `not bool(flags & LLKHF_INJECTED)`. Synthetic `K` replay events cannot cancel active replay. | **Verified** |
| **Recursive Recording Echo Prevention** | Injected events (`LLKHF_INJECTED` / `LLMHF_INJECTED`) are dropped during active recording. | **Verified** |
| **Strict Session Selection Gate** | Replay requires `status == 'complete'`, `integrity_status == 'VALID'`, `flush_confirmed == True`, 0 drops, 0 write errors, and non-empty `events.jsonl`. Corrupt/incomplete sessions rejected. | **Verified** |
| **Precise Timing & Speed Multiplier** | `ReplayEngine` reproduces inter-event delays (`delta_ms / playback_speed`) using a responsive hybrid sleep loop that reacts to cancellation. | **Verified** |
| **Mutual Exclusivity** | `SessionManager` rejects `start_replay()` while recording, rejects `start_recording()` while replaying, and prevents duplicate replay sessions. | **Verified** |
| **Mandatory Safety Release Contract** | `InputInjector.release_all()` tracks all pressed keys and buttons; called on completion, cancellation, error, and shutdown. | **Verified** |
| **Dual UI & CLI Controls** | Dedicated Tkinter buttons `Replay ( K )` and `Stop Replay ( K )` alongside CLI flags (`--replay`, `--replay-session`, `--speed`). | **Verified** |
| **Project Boundary Freeze** | `C:\DEV\tools\rb\` remained untouched. Zero source or configuration mutations. | **Verified** |

---

## 3. Architecture & Module Design

### 3.1 Win32 Low-Level Input Definitions (`src/windows/win32_defs.py`)
Extended low-level Windows API definitions:
- **Constants**: `VK_K = 0x4B`, `INPUT_MOUSE = 0`, `INPUT_KEYBOARD = 1`, `INPUT_HARDWARE = 2`.
- **Keyboard Flags**: `KEYEVENTF_EXTENDEDKEY = 0x0001`, `KEYEVENTF_KEYUP = 0x0002`, `KEYEVENTF_SCANCODE = 0x0008`.
- **Mouse Flags**: `MOUSEEVENTF_MOVE = 0x0001`, `MOUSEEVENTF_LEFTDOWN = 0x0002`, `MOUSEEVENTF_LEFTUP = 0x0004`, `MOUSEEVENTF_RIGHTDOWN = 0x0008`, `MOUSEEVENTF_RIGHTUP = 0x0010`, `MOUSEEVENTF_MIDDLEDOWN = 0x0020`, `MOUSEEVENTF_MIDDLEUP = 0x0040`, `MOUSEEVENTF_XDOWN = 0x0080`, `MOUSEEVENTF_XUP = 0x0100`, `MOUSEEVENTF_WHEEL = 0x0800`, `MOUSEEVENTF_HWHEEL = 0x01000`.
- **Structures**: Defined 64-bit aligned ctypes structures: `MOUSEINPUT`, `KEYBDINPUT`, `HARDWAREINPUT`, `INPUT_UNION`, and `INPUT`.
- **Function Bindings**: Bound `user32.SendInput` and `user32.SetCursorPos`.

### 3.2 Session Quality Gate & Selector (`src/replay/session_selector.py`)
- `is_session_valid_and_complete(session_dir)`:
  1. Verifies existence and readability of `manifest.json` and `events.jsonl`.
  2. Ensures `manifest["status"] == "complete"`.
  3. Validates `manifest["integrity"]["flush_confirmed"] is True` and `manifest["integrity"]["integrity_status"] == "VALID"`.
  4. Enforces zero drops (`metrics["dropped_events_count"] == 0`) and zero write errors (`metrics["write_errors_count"] == 0`).
  5. Verifies `events.jsonl` is non-empty.
- `find_most_recent_valid_session(base_dir)`:
  - Discovers candidate sessions and filters valid ones.
  - Sorts descending by ISO UTC timestamp (`session_start_iso_utc`) to pick the latest valid recording.

### 3.3 Hardware Input Injector (`src/replay/input_injector.py`)
- Manages `_held_keys: Set[int]` and `_held_buttons: Set[str]`.
- `inject_key(vk, scan_code, extended, is_up)`: Updates tracking set and invokes `SendInput(1, INPUT_KEYBOARD)`.
- `inject_mouse_button(button, is_up, cursor_x, cursor_y)`: Moves cursor if specified, updates tracking set, and invokes `SendInput(1, INPUT_MOUSE)`.
- `inject_mouse_move_relative(dx, dy)`: Injects camera displacement via `MOUSEEVENTF_MOVE`.
- `inject_mouse_wheel(wheel_delta, wheel_axis)`: Injects scroll ticks via `MOUSEEVENTF_WHEEL` / `MOUSEEVENTF_HWHEEL`.
- `release_all()`: Traverses active held sets and synthesizes key-up and button-up events. Returns count of released inputs.

### 3.4 Replay Engine Worker (`src/replay/replay_engine.py`)
- Threaded execution worker parsing `events.jsonl` sequentially.
- Calculates per-event sleep duration: `(delta_ms / 1000.0) / playback_speed`.
- Implements `_precise_sleep()` in 5ms slices polling `_stop_requested` to ensure responsive cancellation within < 10ms.
- Enforces policy on unhandled absolute movements: `mouse_move_absolute` is counted as unsupported and safely skipped when `support_absolute_mouse=False`.
- Guarantees `injector.release_all()` in `finally:` block.

### 3.5 Ingress Hook Filter & Hotkey Routing (`src/windows/hook_manager.py`)
- Added `on_replay_hotkey` callback and `replay_vk = VK_K`.
- In keyboard callback:
  ```python
  if nCode >= 0 and (wParam in (WM_KEYDOWN, WM_SYSKEYDOWN)):
      if kb.vkCode == self.replay_vk and not injected:
          if self.on_replay_hotkey:
              self.on_replay_hotkey()
          if self.suppress_hotkeys:
              return 1
  ```
- Replay echo filter:
  ```python
  if injected:
      return user32.CallNextHookEx(None, nCode, wParam, lParam)
  ```

### 3.6 Session State Machine & Mutual Exclusivity (`src/session/session_manager.py`)
- Added states: `RecorderState.REPLAYING`, `RecorderState.STOPPING_REPLAY`, `RecorderState.REPLAYED`.
- State transitions:
  - `start_recording()` strictly blocked if in `REPLAYING` or `STOPPING_REPLAY`.
  - `start_replay()` strictly blocked if in `RECORDING` or `STOPPING`.
  - Duplicate `start_replay()` strictly blocked if already replaying.
- Hotkey `K` toggles replay when IDLE/SAVED/REPLAYED, or requests cancellation when REPLAYING.

### 3.7 UI & CLI Extensions (`src/ui/app.py`, `src/main.py`)
- **Tkinter GUI**: Added `Replay ( K )` and `Stop Replay ( K )` controls with dynamic state-badge styling (`#17a2b8` for `REPLAYING`, `#6c757d` for `STOPPING REPLAY`, `#28a745` for `REPLAYED`).
- **CLI Options**: Added `--replay`, `--replay-session <ID>`, and `--speed <FLOAT>`.

---

## 4. Test Verification Results

Automated test execution across all 31 test cases passed cleanly:

```text
============================= test session starts =============================
platform win32 -- Python 3.12.13, pytest-9.1.1, pluggy-1.7.0
rootdir: C:\DEV\tools
collected 31 items

..\recorder\tests\test_clock.py::test_perf_counter_monotonicity PASSED   [  3%]
..\recorder\tests\test_clock.py::test_iso_utc_format PASSED              [  6%]
..\recorder\tests\test_clock.py::test_compute_delta_ms PASSED            [  9%]
..\recorder\tests\test_event_models.py::test_serialize_key_down PASSED   [ 12%]
..\recorder\tests\test_event_models.py::test_serialize_mouse_move_relative PASSED [ 16%]
..\recorder\tests\test_event_models.py::test_serialize_mouse_move_absolute PASSED [ 19%]
..\recorder\tests\test_event_models.py::test_serialize_mouse_button PASSED [ 22%]
..\recorder\tests\test_event_models.py::test_session_manifest_defaults PASSED [ 25%]
..\recorder\tests\test_live_win32_integration.py::test_hook_manager_callback_integration PASSED [ 29%]
..\recorder\tests\test_queue_writer.py::test_queue_writer_normal_lifecycle PASSED [ 32%]
..\recorder\tests\test_queue_writer.py::test_queue_overflow_drop_policy PASSED [ 35%]
..\recorder\tests\test_replay.py::test_session_selector_valid_and_invalid PASSED [ 38%]
..\recorder\tests\test_replay.py::test_input_injector_tracking_and_release_all PASSED [ 41%]
..\recorder\tests\test_replay.py::test_replay_engine_execution_and_cancellation PASSED [ 45%]
..\recorder\tests\test_replay.py::test_replay_engine_unsupported_events PASSED [ 48%]
..\recorder\tests\test_session_manager_mutual_exclusivity PASSED [ 51%]
..\recorder\tests\test_replay.py::test_hook_manager_hotkey_k_and_injected_filtering PASSED [ 54%]
..\recorder\tests\test_session_lifecycle.py::test_session_state_machine PASSED [ 58%]
..\recorder\tests\test_state_tracker.py::test_key_down_and_auto_repeat PASSED [ 61%]
..\recorder\tests\test_state_tracker.py::test_modifier_disambiguation PASSED [ 64%]
..\recorder\tests\test_state_tracker.py::test_mouse_button_tracking PASSED [ 67%]
..\recorder\tests\test_state_tracker.py::test_focus_loss_reconciliation PASSED [ 70%]
..\recorder\tests\test_ui_telemetry.py::test_single_key_tap_counting PASSED [ 74%]
..\recorder\tests\test_ui_telemetry.py::test_single_mouse_click_counting PASSED [ 77%]
..\recorder\tests\test_ui_telemetry.py::test_auto_repeat_separation_if_present PASSED [ 80%]
..\recorder\tests\test_ui_telemetry.py::test_mixed_session_total_and_breakdown PASSED [ 83%]
..\recorder\tests\test_win32_defs.py::test_keyboard_hook_constants PASSED [ 87%]
..\recorder\tests\test_win32_defs.py::test_mouse_hook_constants PASSED   [ 90%]
..\recorder\tests\test_win32_defs.py::test_raw_mouse_constants PASSED    [ 93%]
..\recorder\tests\test_win32_defs.py::test_hotkey_virtual_keys PASSED    [ 96%]
..\recorder\tests\test_win32_defs.py::test_struct_sizes PASSED           [100%]

============================= 31 passed in 1.11s ==============================
```

---

## 5. Standalone Binary Build

The standalone bundle was compiled via PyInstaller:
- **Executable Output**: `C:\DEV\tools\recorder\dist\recorder\recorder.exe` (1,913,570 bytes).
- **Configuration Output**: `C:\DEV\tools\recorder\dist\recorder\config.json`.
- **CLI Verification**: Running `recorder.exe --help` and `recorder.exe --replay` confirmed CLI flags, proper parsing, and graceful error reporting when no session directory is present.
