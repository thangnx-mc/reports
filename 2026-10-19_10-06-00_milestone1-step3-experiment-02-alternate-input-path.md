# Task Report: Milestone 1 Step 3 — Experiment 02: Alternate Input Path (mouse_event vs SendInput)

**Task Identifier:** `TASK-M1-STEP3-EXPERIMENT-02-ALTERNATE-INPUT-PATH`  
**Timezone:** Asia/Ho_Chi_Minh (UTC+7)  
**Received Timestamp:** `2026-10-19T09:57:32+07:00`  
**Completed Timestamp:** `2026-10-19T10:07:00+07:00`  
**Task Status:** `IMPLEMENTATION COMPLETE / AUTOMATED VERIFICATION PASS / EXTERNAL VERIFICATION PENDING`  

---

## 1. Experiment 02 Context & Technical Rationale

### Live Problem Statement
In live testing of Experiment 01 (`2026-10-16_11-30-00_milestone1-step3-experiment-01-focus-hover-handshake.md`), the standalone macro executable successfully foregrounded Roblox, navigated the cursor to `PLAY_GREEN_LOBBY` at (947, 516), waited the 0.25s hover duration, and emitted the `SendInput` click packet (`down_ret=1, up_ret=1`). However, Roblox did not register the click, and after 3 bounded retries, safely halted. As before, once the user manually clicked 1–2 times inside the Roblox window, Roblox resumed normal mouse interaction and the macro automatically completed the remaining 4 room-creation steps.

### Inspection Findings
1. `[FACT]`: In `src/input_controller.py`, the baseline implementation mixed `SetCursorPos(int(x), int(y))` with `SendInput(MOUSEEVENTF_MOVE | MOUSEEVENTF_ABSOLUTE)`.
2. `[FACT]`: `SendInput` flags synthetic input packets with Windows injection metadata and timestamps, which some game engines and RawInput hooks filter or drop if window activation state is perceived as inactive.
3. `[HYPOTHESIS]`: `user32.mouse_event` provides an alternate Windows mouse input dispatch path that posts directly into the system hardware input queue without modern injection metadata. By pairing `SetCursorPos` with `user32.mouse_event(MOUSEEVENTF_MOVE | MOUSEEVENTF_ABSOLUTE)` and explicit mouse-down, 80ms hold, and mouse-up sequencing, Roblox's input handler may accept the synthetic hardware input without requiring manual user activation.

### Experiment 02 Scope & Design
- Implement a selectable input dispatch strategy:
  - `"mouse_event"`: Alternate hardware input path via `user32.mouse_event` and `SetCursorPos`.
  - `"sendinput"`: Preserved baseline path via `SendInput` with fallback.
- Make the default strategy `"mouse_event"`, while keeping `"sendinput"` selectable in the GUI dropdown and config.
- Tag every log entry with `[EXP-02]` and explicit strategy attribution `(strategy={strat})`.
- Attribute every audit event to the active strategy.
- Preserve all existing safety invariants: `dry_run`, `observe_only`, emergency stop (`F8`/`Esc`), manual debounce, and bounded retries.

---

## 2. Exact Code Changes

### A. Configuration Models (`src/config.py`)
- Extended `DungeonConfig` with:
  - `input_strategy: str = "mouse_event"`: Selectable between `"mouse_event"` (default) and `"sendinput"` (baseline).

### B. Application Wiring (`src/macro_app.py`)
- Initialized `InputController` passing `input_strategy=getattr(self.config.dungeon, "input_strategy", "mouse_event")`.

### C. Hardware Input Driver & Controller (`src/input_controller.py`)
- **`Win32DirectInputDriver.move_mouse(x, y, strategy="sendinput")`:**
  - Added support for `strategy == "mouse_event"`: sets position with `SetCursorPos`, dispatches `user32.mouse_event(MOUSEEVENTF_MOVE | MOUSEEVENTF_ABSOLUTE, norm_x, norm_y, 0, 0)`, and synchronizes with `SetCursorPos`.
  - Preserved baseline `SendInput` motion for `strategy == "sendinput"`.
- **`Win32DirectInputDriver.click(x, y, button="left", target_hwnd=None, strategy="sendinput")`:**
  - Added support for `strategy == "mouse_event"`:
    - Ensures window foreground focus.
    - Moves mouse using `strategy="mouse_event"`.
    - Sleeps 80ms hover settling delay.
    - Dispatches `user32.mouse_event(down_flag | MOUSEEVENTF_ABSOLUTE, norm_x, norm_y, 0, 0)`.
    - Holds button for 80ms for reliable frame sampling.
    - Dispatches `user32.mouse_event(up_flag | MOUSEEVENTF_ABSOLUTE, norm_x, norm_y, 0, 0)`.
    - Returns `ClickDispatchResult(method="mouse_event", down_ret=1, up_ret=1, success=True, ...)`.
  - Preserved baseline `SendInput` dispatch with fallback to `mouse_event_fallback` when `strategy == "sendinput"`.
- **`InputController`:**
  - Added `input_strategy: str = "mouse_event"` to `__init__`.
  - Added `set_input_strategy(self, strategy: str) -> None` validating `strategy in {"mouse_event", "sendinput"}`.
  - Updated `move_to(self, x, y, strategy=None)` to record strategy in audit events and forward to driver.
  - Updated `click_at(self, x, y, button="left", strategy=None)` to record strategy in audit events and forward to driver.
  - Updated `_hardware_click(self, x, y, button, strategy=None)` to pass strategy to `driver.click` with fallback.

### D. GUI Controls & Synchronization (`src/gui_app.py`)
- In `_build_controls_and_settings`:
  - Added Strategy label and `ttk.Combobox` (`values=["mouse_event", "sendinput"]`) to `live_row`.
- Added `_on_strategy_changed(self, event=None)`:
  - Synchronizes selection to `app.config.dungeon.input_strategy` and `app.input_ctrl.set_input_strategy(strat)`.
  - Logs `[INFO] Input dispatch strategy set to: '{strat}'`.
- In `_on_live_creation_toggled` and `start_observation`:
  - Synchronizes strategy from GUI combobox to controller.
  - Disables combobox while observation is running.
- In `_reset_ui_to_idle`:
  - Re-enables combobox (`state="readonly"`).

### E. State Machine Logging & Dispatch Attribution (`src/state_machine.py`)
- Updated `IN_LOBBY` and `SELECTING_DUNGEON` to retrieve `strat = getattr(self.input_ctrl, "input_strategy", "mouse_event")`.
- Passed `strategy=strat` to `move_to` and `click_at`.
- Updated all stage log tags from `[EXP-01]` to `[EXP-02]` with explicit strategy attribution:
  - `[EXP-02][FOCUS] ... (strategy={strat})`
  - `[EXP-02][CURSOR] ... (strategy={strat})`
  - `[EXP-02][WAIT] ... (strategy={strat})`
  - `[EXP-02][VISION] ... (strategy={strat})`
  - `[EXP-02][CLICK] ... (strategy={strat})`
  - `[EXP-02][OUTCOME] ... (strategy={strat})`
  - `[EXP-02][POST-CLICK] ... (strategy={strat})`
  - `[EXP-02][RETRY] ... (strategy={strat})`

---

## 3. Automated Test Verification

### Focused Unit Tests
- `tests/test_input_controller.py`:
  - `test_input_strategy_default_and_setter`: Validates default, case-insensitivity, and rejection of invalid values.
  - `test_input_strategy_tagging_in_audit_events`: Validates audit event details contain `Strategy: {strat}`.
  - `test_driver_receives_strategy_parameter`: Validates mock driver receives strategy argument in `move_mouse` and `click`.
  - `test_win32_driver_strategy_mouse_event`: Validates Win32 driver execution under `mouse_event` strategy.
- `tests/test_state_machine.py`:
  - `test_experiment_02_strategy_logging_and_event_attribution`: Validates state machine dispatches with strategy tag.
  - `test_experiment_01_bounded_retries_halts_safely_when_retries_exhausted`: Updated halt reason check.
- `tests/test_gui_app.py`:
  - `test_gui_input_strategy_combobox_selection_syncs_to_controller`: Validates UI combobox updates config and controller.
  - TearDown cleanup for `var_input_strategy`.

- **Focused Test Command:**  
  `python -m unittest tests/test_input_controller.py tests/test_state_machine.py tests/test_gui_app.py`
- **Result:** **62/62 passed** in 10.2s.

### Full Test Suite Regression
- **Full Test Suite Command:**  
  `python -m unittest discover -s tests`
- **Result:** **192/192 passed** (0 failures, 0 errors, 24.8s).

---

## 4. Standalone Executable Packaging

- **Packaging Script:** `scripts\build_exe.ps1`
- **Command:**  
  `powershell -ExecutionPolicy Bypass -File scripts\build_exe.ps1 -PythonPath "$env:APPDATA\uv\python\cpython-3.12.13-windows-x86_64-none\python.exe"`
- **Result:**
  - Output: `dist\DungeonQuestMacro\DungeonQuestMacro.exe` (3.83 MB)
  - Total Bundle: 175.02 MB (2,022 files)
  - Status: **Build Succeeded in 48.3 seconds**.

---

## 5. Evidence Classification

- `[FACT]`: In live testing of Experiment 01, `SendInput` failed to register clicks on `PLAY_GREEN_LOBBY` until user manually interacted with Roblox.
- `[FACT]`: The baseline implementation mixed `SetCursorPos` with `SendInput`.
- `[FACT]`: `user32.mouse_event` is a distinct Win32 dispatch path that posts directly into the system hardware input queue without modern injection metadata.
- `[HYPOTHESIS]`: Discarding `SendInput` injection flags and using `mouse_event` with explicit `SetCursorPos` coordinate alignment will register the click in Roblox without manual activation.
- `[REQUIRES EXTERNAL VERIFICATION]`: Live test in Roblox Dungeon Quest runtime using `dist/DungeonQuestMacro/DungeonQuestMacro.exe`.

---

## 6. User Handoff for External Runtime Verification

### Instructions for Live Testing
1. Launch Roblox and enter the Dungeon Quest town square lobby. Ensure the window is visible and not minimized.
2. Launch `dist\DungeonQuestMacro\DungeonQuestMacro.exe`.
3. Verify that the **Strategy** dropdown defaults to `mouse_event` (you can also switch back to `sendinput` if you want to compare).
4. Select your Roblox window from the dropdown.
5. Check **"Enable Live Lobby Creation"** and confirm the safety modal.
6. Click **"Start Lobby Creation"** (or press `[F6]`).
7. **Do NOT touch the mouse or keyboard after clicking Start.**

### Expected Behavior
1. Window focus is verified (`[EXP-02][FOCUS] (strategy=mouse_event)`).
2. Cursor moves to `PLAY_GREEN_LOBBY` at (947, 516) (`[EXP-02][CURSOR] (strategy=mouse_event)`).
3. Hover settling delay of 0.250s elapses (`[EXP-02][WAIT] (strategy=mouse_event)`).
4. Click is dispatched via `mouse_event` (`[EXP-02][CLICK] Attempt #1: method=mouse_event ...`).
5. Roblox immediately opens the mode selection modal (`LOBBY_MODE_SELECTION`).
6. Macro executes remaining 4 clicks and safely stops at `PARTY_STAGING_LOBBY`.

### Failure Criteria & Evidence Reporting
- If Roblox still does not register the click after 3 retry attempts, the macro will halt safely.
- Please report:
  1. Did the mode selection dialog open on Attempt #1 or on a retry?
  2. Did the in-game hand cursor appear over PLAY before the click?
  3. Did switching the dropdown between `mouse_event` and `sendinput` change the behavior?
