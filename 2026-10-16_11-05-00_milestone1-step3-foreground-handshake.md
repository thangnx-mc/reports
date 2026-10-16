# Task Report: Fix Foreground & Input Activation Handshake for Live Roblox Macro

**Task Identifier:** `TASK-M1-STEP3-FOREGROUND-INPUT-HANDSHAKE`  
**Timezone:** Asia/Ho_Chi_Minh (UTC+7)  
**Received Timestamp:** `2026-10-16T10:35:00+07:00`  
**Completed Timestamp:** `2026-10-16T11:05:00+07:00`  
**Task Status:** `IMPLEMENTATION COMPLETE / AUTOMATED VERIFICATION PASS / EXTERNAL VERIFICATION PENDING`  

---

## 1. Executive Summary & Defect Diagnosis

During the live test of `TASK-M1-STEP3`, the standalone executable launched and moved the cursor to `PLAY_GREEN_LOBBY`, but Roblox initially did not respond to the injected clicks until the user manually clicked 1–2 times inside the Roblox window. Immediately after those manual clicks, Roblox activated its input handling, displayed the hand cursor over `PLAY`, and the macro automatically completed the subsequent 4 clicks (`Create Dungeon` -> `Winter Outpost` -> `Nightmare` -> `Create Lobby`) and safely stopped at `PARTY_STAGING_LOBBY`.

### Telemetry & Log Evidence (`dist/DungeonQuestMacro/macro_audit.log`)
- `[FACT]`: At `10:41:42`, the macro moved the cursor to screen $(1776, 963)$ and dispatched clicks with `driver=Win32DirectInputDriver, method=SendInput`.
- `[FACT]`: From `10:41:42` to `10:44:55`, the macro retried clicking `PLAY_GREEN_LOBBY` every ~7 seconds because the game remained in `LOBBY_IDLE` (Roblox did not open the mode selection modal).
- `[FACT]`: At `10:44:55`, manual clicks were performed inside Roblox.
- `[FACT]`: At `10:44:57` – `10:45:00`, the macro successfully executed the entire remaining sequence without errors:
  - `Clicked CREATE_DUNGEON_ORANGE at (512, 316)`
  - `Selected dungeon tile 'Winter Outpost' at (348, 268)`
  - `Selected difficulty 'Nightmare' at (669, 375)`
  - `Clicked CREATE_LOBBY_RED at (512, 379)`
  - `[SUCCESS] Verified PARTY_STAGING_LOBBY confirmed with START_YELLOW_PARTY_LOBBY visible. Milestone 1 Complete: Winter Outpost Nightmare room created! Stopping automation safely.`

### Root Cause Analysis
1. **Thread Input Queue & Activation Consumption (`WM_MOUSEACTIVATE` swallow):**
   - `[FACT]`: Even after calling `SetForegroundWindow`, Windows threads maintain separate input queues unless attached via `AttachThreadInput`. The target window may appear foregrounded visually, but its input thread does not have keyboard/mouse focus (`GetFocus() == NULL`).
   - `[FACT]`: When an inactive window receives an initial mouse click, Windows dispatches `WM_MOUSEACTIVATE`. For top-level game windows, this commonly returns `MA_ACTIVATEANDEAT`, consuming the first click solely to focus the window without passing button-down/up messages to the in-game UI framework.
2. **RawInput Motion Vector Suppression by `SetCursorPos`:**
   - `[FACT]`: `SetCursorPos` updates the desktop cursor coordinates directly at the display driver level, but it does NOT emit `WM_INPUT` / RawInput movement packets into the game engine's hardware input stream.
   - `[INFERENCE]`: When `SetCursorPos` was called before `SendInput(MOUSEEVENTF_MOVE)`, the cursor was already at $(x, y)$, causing `SendInput` to generate a motion vector $\Delta x = 0, \Delta y = 0$. Roblox's DirectInput/RawInput subsystem detected zero relative movement, preventing it from firing `MouseEnter` and updating the UI hover state to the hand cursor.

---

## 2. Implementation Details

### A. Thread-Synchronized Window Activation (`src/input_controller.py`)
- Upgraded `Win32DirectInputDriver.ensure_foreground`:
  - Retrieves foreground thread ID (`GetWindowThreadProcessId(GetForegroundWindow(), ...)`) and target thread ID (`GetWindowThreadProcessId(target_hwnd, ...)`).
  - Uses `AttachThreadInput` to synchronize input queues between the macro process and Roblox.
  - Calls `BringWindowToTop(target_hwnd)`, `SetForegroundWindow(target_hwnd)`, and `SetFocus(target_hwnd)` to establish true thread-level input focus.
  - Detaches thread input cleanly after focus acquisition.

### B. Hardware RawInput Motion Injection (`src/input_controller.py`)
- Implemented `Win32DirectInputDriver.move_mouse(x, y)`:
  - Computes an approach waypoint $(x - 12, y - 12)$ and dispatches an absolute `SendInput(MOUSEEVENTF_MOVE | MOUSEEVENTF_ABSOLUTE)` event.
  - Emits a second `SendInput(MOUSEEVENTF_MOVE | MOUSEEVENTF_ABSOLUTE)` directly to $(x, y)$, generating non-zero RawInput motion deltas.
  - Confirms coordinates with `SetCursorPos(x, y)` and enforces an 80ms hover settling delay so Roblox fires `MouseEnter` and activates the hand cursor.

### C. Automatic First-Interaction Handshake (`src/input_controller.py`)
- Implemented `_has_performed_initial_handshake` guard:
  - If the target window is newly focused or has not yet performed the handshake, `Win32DirectInputDriver.click` automatically dispatches an activation handshake:
    1. Sends a preliminary left mouse down/up click to satisfy `WM_MOUSEACTIVATE`.
    2. Pauses 120ms for the game engine to complete window activation.
    3. Re-verifies foreground focus.
    4. Re-injects mouse motion with approach deltas to activate the hover state.
    5. Sets `_has_performed_initial_handshake = True`.
  - The method then immediately dispatches the primary button click.
  - Subsequent clicks in the sequence skip the handshake, ensuring rapid execution.
- Added `reset_handshake()` method to reset the handshake state whenever observation restarts or settings change.

### D. GUI Thread Immediate Activation (`src/gui_app.py`)
- In `start_observation`: When the user clicks "Start Lobby Creation", the GUI thread immediately calls `input_ctrl.driver.ensure_foreground(self.selected_window.hwnd)` and `input_ctrl.reset_handshake()` before starting the worker loop.
- In `_on_live_creation_toggled`: Resets handshake state upon checkbox toggles.

### E. Accelerated Lobby Idle Recovery (`src/state_machine.py`)
- In `SELECTING_DUNGEON`: Reduced the `LOBBY_IDLE` retry timeout from 4.0s to 2.5s. If Roblox still fails to register the initial click, the FSM quickly drops back to `IN_LOBBY` to re-attempt `PLAY_GREEN_LOBBY`.

---

## 3. Verification & Build Results

### Automated Regression Tests
- **Focused Tests:**
  - Command: `python -m unittest tests/test_input_controller.py tests/test_state_machine.py`
  - Result: **26/26 tests passed** (0.88s runtime).
- **Full Test Suite:**
  - Command: `python -m unittest discover -s tests`
  - Result: **181/181 tests passed** (0 failures, 0 errors, 16.36s runtime).

### Standalone Executable Packaging
- **Build Script:** `scripts\build_exe.ps1`
- **Command:** `powershell -ExecutionPolicy Bypass -File scripts\build_exe.ps1 -PythonPath "C:\Users\nguye\AppData\Roaming\uv\python\cpython-3.12.13-windows-x86_64-none\python.exe"`
- **Result:**
  - Executable: `dist\DungeonQuestMacro\DungeonQuestMacro.exe` (3.83 MB, updated 2026-10-16 11:02:35)
  - Total Bundle: 175.02 MB (2,022 files)
  - Status: **Build Succeeded in 12.1 seconds**.

---

## 4. Evidence Classification

- `[FACT]`: In `macro_audit.log`, manual user clicks at `10:44:55` allowed the macro to execute all remaining clicks automatically and reach `PARTY_STAGING_LOBBY`.
- `[FACT]`: `AttachThreadInput` and `SetFocus` are required Win32 APIs for cross-thread focus assignment without UI thread preemption.
- `[FACT]`: Automated test suite passed 181/181 tests including the new `move_mouse` and `reset_handshake` contracts.
- `[REQUIRES EXTERNAL VERIFICATION]`: Verifying that Roblox accepts the automated activation click and non-zero RawInput motion delta on live hardware without any user clicks.

---

## 5. User Handoff for External Runtime Verification

### Test Instructions
1. Launch Roblox and enter the Dungeon Quest town square lobby. Ensure the Roblox window is visible and not minimized.
2. Launch `dist\DungeonQuestMacro\DungeonQuestMacro.exe`.
3. Select your Roblox window from the dropdown (or click "Refresh Windows" if necessary).
4. Check the **"Live Lobby Creation"** checkbox and confirm the safety modal.
5. Click **"Start Lobby Creation"** (or press F6).
6. **Do NOT touch mouse or keyboard after clicking Start.** Allow the macro to perform the handshake automatically.

### Expected Behavior
1. The macro immediately brings Roblox to the foreground and synchronizes focus.
2. The cursor moves to the green **PLAY** button with a non-zero motion delta; the in-game hand cursor appears over `PLAY`.
3. The macro performs the activation handshake and clicks `PLAY` **without requiring manual user clicks**.
4. The mode selection modal opens -> macro clicks orange `Create Dungeon`.
5. The dungeon creation menu opens -> macro clicks `Winter Outpost` -> `Nightmare` -> red `Create Lobby`.
6. The macro detects `PARTY_STAGING_LOBBY` with the yellow `START` button visible and stops safely.

### Failure Criteria
- Cursor moves to `PLAY` but Roblox does not show the hand cursor and does not open mode selection without manual user clicking.
- Any click outside the expected 5 buttons or unintended mouse movement.
