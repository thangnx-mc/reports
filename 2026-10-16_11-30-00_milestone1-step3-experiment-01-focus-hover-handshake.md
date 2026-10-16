# Task Report: Milestone 1 Step 3 — Experiment 01: Focus + Hover Handshake

**Task Identifier:** `TASK-M1-STEP3-EXPERIMENT-01-FOCUS-HOVER-HANDSHAKE`  
**Timezone:** Asia/Ho_Chi_Minh (UTC+7)  
**Received Timestamp:** `2026-10-16T11:14:00+07:00`  
**Completed Timestamp:** `2026-10-16T11:30:00+07:00`  
**Task Status:** `IMPLEMENTATION COMPLETE / AUTOMATED VERIFICATION PASS / EXTERNAL VERIFICATION PENDING`  

---

## 1. Experiment 01 Hypothesis

### Problem Statement
In live execution, the macro navigates the mouse to `PLAY_GREEN_LOBBY`, but Roblox does not respond to the injected clicks until the user manually clicks 1–2 times inside the Roblox window. Once manually clicked, the in-game hand cursor appears over `PLAY`, and the macro automatically completes all remaining steps (`Create Dungeon` -> `Winter Outpost` -> `Nightmare` -> `Create Lobby`) and safely stops at `PARTY_STAGING_LOBBY`.

### Experiment 01 Hypothesis
Rather than attempting internal double-click workarounds inside the driver, the defect can be systematically tested and addressed by establishing an explicit, staged **Focus + Hover Handshake** protocol:
1. **Window Focus Verification:** When Live Lobby Creation starts (and prior to interaction), explicitly restore and foreground the target Roblox window, verifying whether `target_hwnd == foreground_hwnd`.
2. **Physical Cursor Translation & Approach Motion:** The cursor must be moved to calibrated coordinates using non-zero RawInput approach vectors, entering the button area from outside to trigger `MouseEnter`.
3. **Hover Settling Duration:** A perceptible hover settling delay (`play_button_hover_settle_seconds = 0.25s`) must elapse while the cursor remains positioned over `PLAY_GREEN_LOBBY` before mouse-down is emitted, giving Roblox's UI subsystem time to process the pointer position and update its hover/activation state.
4. **Single Clean SendInput Emission:** Send exactly ONE standard `SendInput` click packet (mouse down, 80ms hold duration, mouse up) with no multi-click noise.
5. **UI-State Confirmed Determination of Success:** Success must NOT be inferred merely from a successful Win32 API return (`down_ret > 0`), but exclusively from visual confirmation of the expected subsequent game state (`GameState.LOBBY_MODE_SELECTION` or presence of `create_dungeon_orange`).
6. **Bounded Retries with State Re-detection:** If the mode-selection modal does not appear within a bounded response window (`play_button_retry_delay_seconds = 1.5s`), allow a strictly bounded retry count (`play_button_max_retries = 3`) with state re-detection between attempts. If retries are exhausted without UI response, safely halt rather than spamming clicks.

---

## 2. Exact Code Changes

### A. Configuration Models (`src/config.py`)
- Extended `DungeonConfig` with configurable parameters for Experiment 01:
  - `play_button_hover_settle_seconds: float = 0.25`: Documented hover settling duration before emitting the PLAY click.
  - `play_button_max_retries: int = 3`: Maximum bounded retries for the PLAY button before safe halt.
  - `play_button_retry_delay_seconds: float = 1.5`: Wait window before determining that a click was dropped and re-attempting.

### B. Hardware Input Driver & Controller (`src/input_controller.py`)
- **`Win32DirectInputDriver.ensure_foreground`:**
  - Added structured stage logging: `[EXP-01][FOCUS] Window focus verification: target_hwnd=..., foreground_hwnd=..., match=...`.
- **`Win32DirectInputDriver.click`:**
  - Cleaned up internal activation multi-clicks; ensures exactly ONE click packet is emitted per `click()` invocation via `SendInput`.
- **`InputController`:**
  - Added `ensure_window_focus(target_hwnd)`: delegates to driver foreground verification.
  - Added `move_to(x, y)`: handles calibrated-to-screen coordinate translation and calls `driver.move_mouse` without clicking, recording `mouse_move` audit events in dry-run/mock modes.

### C. GUI Window Activation Logging (`src/gui_app.py`)
- In `start_observation` and `_on_live_creation_toggled`:
  - When Live Lobby Creation starts, calls `input_ctrl.ensure_window_focus(selected_window.hwnd)` and logs:
    `[EXP-01][FOCUS] Target HWND: {hwnd}, Foreground HWND: {cur_fg}, Status: {'MATCH' if focus_ok else 'MISMATCH'}` directly into the GUI activity log.

### D. Vision Engine Element Forwarding (`src/vision_engine.py`)
- Added `find_ui_element(frame, element_name)` on `VisionEngine` forwarding to the active vision provider.

### E. State Machine Staged Experiment 01 Protocol (`src/state_machine.py`)
- Added Experiment 01 progression state and retry tracking in `MacroStateMachine`:
  - `self.play_button_hover_settle_seconds`, `self.play_button_max_retries`, `self.play_button_retry_delay_seconds`, and `self.play_click_retries`.
- **`MacroState.IN_LOBBY` Protocol:**
  - Checks if mode selection modal is already open; if so, immediately advances to `SELECTING_DUNGEON`.
  - When in `LOBBY` / `LOBBY_IDLE` with `PLAY_GREEN_LOBBY` visible:
    - **Stage 1 (Focus):** Verifies foreground focus: `[EXP-01][FOCUS] ...`
    - **Stage 2 (Cursor):** Dispatches calibrated cursor movement: `[EXP-01][CURSOR] ...`
    - **Stage 3 (Wait):** Enforces hover settling delay: `[EXP-01][WAIT] ...`
    - **Stage 4 (Vision):** Audits supporting vision element presence: `[EXP-01][VISION] ...`
    - **Stage 5 (Click):** Emits ONE click via `SendInput`: `[EXP-01][CLICK] ...`
    - Transitions to `SELECTING_DUNGEON` to determine success.
- **`MacroState.SELECTING_DUNGEON` Protocol:**
  - **Stage 6 (UI Confirmation):** If `game_state == GameState.LOBBY_MODE_SELECTION`:
    - Logs `[EXP-01][OUTCOME] Success: Expected mode-selection UI confirmed...`
    - Resets `self.play_click_retries = 0`.
    - Proceeds with existing verified sequence (`create_dungeon_orange` -> `tile_Winter Outpost` -> `diff_Nightmare` -> `create_lobby_red`).
  - **Stage 7 (Bounded Retries):** If `game_state` remains `LOBBY` / `LOBBY_IDLE`:
    - Waits for `play_button_retry_delay_seconds` (1.5s).
    - Logs `[EXP-01][POST-CLICK] Post-click visual state remains 'LOBBY_IDLE'. Expected 'LOBBY_MODE_SELECTION'.`
    - If `play_click_retries >= play_button_max_retries` (3 attempts exhausted):
      - Logs `[EXP-01][OUTCOME] Failed: Expected mode-selection UI did not appear after 3 attempts. Halting safely.`
      - Engages `halt_safely`.
    - Else:
      - Increments `play_click_retries += 1`.
      - Logs `[EXP-01][RETRY] Retrying PLAY_GREEN_LOBBY (attempt N of 4)...`
      - Transitions back to `IN_LOBBY` to execute the handshake cleanly.

---

## 3. Automated Test Verification

### Focused Tests
- Added unit tests in `tests/test_input_controller.py`:
  - `test_input_controller_move_to_records_event`
  - `test_input_controller_ensure_window_focus_forwards_to_driver`
- Added unit tests in `tests/test_state_machine.py`:
  - `test_experiment_01_focus_hover_handshake_emits_move_settle_click`
  - `test_experiment_01_bounded_retries_halts_safely_when_retries_exhausted`
  - `test_experiment_01_mode_selection_confirmed_resets_retry_counter`
- **Command:** `python -m unittest tests/test_input_controller.py tests/test_state_machine.py`
- **Result:** **31/31 passed** (1.14s runtime).

### Complete Test Suite
- **Command:** `python -m unittest discover -s tests`
- **Result:** **186/186 tests passed** (0 failures, 0 errors, 23.88s runtime).

*Note: All automated tests are executed in mock/synthetic and unit test environments; they verify logic, state transitions, event dispatch, and telemetry contracts, but do not prove external Roblox client runtime acceptance.*

---

## 4. Standalone Executable Packaging

- **Build Script:** `scripts\build_exe.ps1`
- **Command:** `powershell -ExecutionPolicy Bypass -File scripts\build_exe.ps1 -PythonPath "C:\Users\nguye\AppData\Roaming\uv\python\cpython-3.12.13-windows-x86_64-none\python.exe"`
- **Result:**
  - Output: `dist\DungeonQuestMacro\DungeonQuestMacro.exe` (3.83 MB, updated 2026-10-16 11:25:29)
  - Bundle: 175.02 MB across 2,022 files.
  - Status: **Build Succeeded in 22.1 seconds**.

---

## 5. Unresolved Risks & Evidence Classification

- `[FACT]`: In `macro_audit.log`, manual user clicks in Roblox allowed the subsequent 4 clicks to succeed automatically.
- `[FACT]`: Automated tests prove that the state machine enforces focus verification, cursor movement, hover delay, single click emission, UI confirmation, and safe halting upon retry exhaustion.
- `[FACT]`: Existing vision infrastructure can verify element bounding boxes and color masks, but cannot detect OS-level pointer hover cursors (e.g. hand cursor) rendered by Windows outside client bitmaps.
- `[HYPOTHESIS]`: A dedicated cursor approach movement and a 0.25s hover settling delay before sending a single `SendInput` click will allow Roblox's UI hover engine to register `MouseEnter` and process the click without manual user intervention.
- `[REQUIRES EXTERNAL VERIFICATION]`: Live validation in the actual Roblox game window.

---

## 6. User Handoff for External Runtime Verification

### Test Instructions
1. Launch Roblox and enter Dungeon Quest town square lobby. Ensure the window is visible and not minimized.
2. Launch `dist\DungeonQuestMacro\DungeonQuestMacro.exe`.
3. Select your Roblox window from the dropdown.
4. Check **"Live Lobby Creation"** and confirm the safety dialog.
5. Click **"Start Lobby Creation"** (or press F6).
6. **Do NOT touch the mouse or keyboard after clicking Start.**

### Expected Telemetry in Log & Behavior
1. `[EXP-01][FOCUS]`: Window focus verified (`MATCH`).
2. `[EXP-01][CURSOR]`: Cursor moves to `PLAY_GREEN_LOBBY` at (947, 516).
3. `[EXP-01][WAIT]`: Hover delay of 0.250s elapses.
4. `[EXP-01][CLICK]`: Exactly ONE click emitted via `SendInput`.
5. `[EXP-01][OUTCOME]`: Mode selection modal opens (`LOBBY_MODE_SELECTION` confirmed).
6. Macro completes the 4 remaining clicks and safely stops at `PARTY_STAGING_LOBBY`.

### Failure Criteria
- Cursor hovers over `PLAY` but Roblox does not open the mode selection modal after 3 attempts, engaging safe halt.
