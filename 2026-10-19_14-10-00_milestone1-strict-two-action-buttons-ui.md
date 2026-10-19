# Milestone 1 Report: Strict UI Requirement — Exactly Two Action Buttons

## 1. Task Metadata
- **Task ID**: `TASK-M1-STRICT-TWO-BUTTONS-UI`
- **Task Description**: Strict consolidation and simplification of the application user interface to contain exactly two action buttons (`START LIVE TEST (.)` and `STOP (,)`), removing `Re-scan Window` and all manual capture, target selection, mode selection, or recovery controls in favor of 100% automated lifecycle management.
- **Task Received Timestamp**: `2026-10-19T13:57:35+07:00`
- **Task Completion Timestamp**: `2026-10-19T14:10:00+07:00`
- **Timezone**: `Asia/Ho_Chi_Minh (+07:00)`
- **Task Status**: `COMPLETED` (Implementation & Automated Verification Complete; External Roblox Runtime Verification Pending User Handoff)

### Status Breakdown
- **Implementation Status**: `Complete`
- **Automated Verification**: `Passed` (213/213 automated unit and integration tests passed in 27.1s)
- **External / Runtime Verification**: `Pending User Verification` (Requires execution in live Roblox client)
- **Remaining Limitations / Unknowns**:
  - `[FACT]` Windows OS DirectInput/RawInput packet consumption inside the Roblox proprietary game engine requires verification in the real runtime environment.
  - `[FACT]` Global hotkey registration depends on host key unicity (`.` and `,`).
- **Final Definition of Done**: `Satisfied` for TASK-M1-STRICT-TWO-BUTTONS-UI scope.
- **Final Task Status**: `IMPLEMENTATION COMPLETE / AUTOMATED TEST VERIFICATION PASS / EXTERNAL VERIFICATION PENDING`

---

## 2. Executive Summary & Core Objective

The user established a strict architectural requirement for the macro application user interface:
> "The application UI must contain exactly these two action buttons:
> 1. `START LIVE TEST (.)`
> 2. `STOP (,)`
> No other action buttons are allowed. Remove `Re-scan Window` and any other manual controls for capture, target selection, mode selection, or recovery.
> Everything else must be handled automatically by the system."

This specification completely eliminates manual operational friction:
- No manual window re-scanning.
- No manual capture or observation mode checkboxes.
- No manual mode/strategy comboboxes.
- No separate recording or recovery buttons on the primary user surface.
- Full automation for target discovery, validation, input initialization, state detection, and recovery.

---

## 3. Strict UI Implementation & Control Consolidation

### Action Surface Consolidation:
In `src/gui_app.py`:
1. **Action Buttons**: The visible UI contains **strictly and exactly two action buttons**:
   - `START LIVE TEST (.)` (`btn_start`): Discovers and validates the target Roblox window, activates and verifies foreground focus, initializes the live input controller, and starts the automation worker loop.
   - `STOP (,)` (`btn_stop`): Gracefully requests worker shutdown, halts input dispatch, resets input handshake, and returns UI to ready state.
2. **Removed Action Buttons from UI**:
   - `Re-scan Window` (`btn_refresh`): Completely removed from the interface.
   - `EMERGENCY STOP (F8)`: Button widget removed from UI surface. Emergency stop remains fully operational system-wide via native Win32 global hotkey `F8` and is documented in the read-only hotkeys status text.
   - `Clear Log` (`btn_clear_log`): Button widget removed from UI surface.
   - `Start Recording` / `Stop Recording` (`btn_rec_start` / `btn_rec_stop`): Removed from UI surface. Instantiated on an unmapped frame to preserve headless/test backward compatibility without exposing any extra controls to the user.
   - `Capture & Analyze Once`: Kept unmapped.

### Read-Only Target Status:
The target window panel displays read-only status:
- `Target: Auto-discovering Roblox...` (when game client is not yet launched).
- `Target: HWND <hwnd> | RobloxPlayerBeta.exe (PID <pid>)` (when game client is discovered).
- `Status`, `Viewport`, `Capture`, and `Error` metrics remain purely informative read-only labels.

---

## 4. Fully Automated System Behaviors

In accordance with the specification, all secondary operations are now handled autonomously by the system:

1. **Automatic Target Discovery & Validation**:
   - Discovery runs automatically on GUI launch.
   - During idle periods, the GUI event loop (`_process_queue`) automatically validates window validity every 2.0s via `ctypes.windll.user32.IsWindow`. If Roblox is launched after the macro, or if Roblox restarts, the system detects the new window automatically without user intervention.
   - When `START LIVE TEST` (or `.`) is pressed, live process identification (`RobloxPlayerBeta.exe`) and window filtering run immediately. If the target is missing or fails foreground activation, the system halts safely without emitting any input.

2. **Safe Input Controller Initialization**:
   - When live test starts, `InputController` is automatically configured with:
     - `observe_only=False` and `dry_run=False`
     - Verified target HWND
     - Calibrated coordinate translator
     - Verified `mouse_event` input dispatch strategy
     - Cleared input handshake and safety flags

3. **Autonomous State Detection & Recovery**:
   - The state machine (`src/state_machine.py`) continuously classifies game states:
     - Detects `INITIALIZING`, `IN_LOBBY`, `SELECTING_DUNGEON`, `CREATING_ROOM`, `PARTY_STAGING_LOBBY`, `IN_MAP`, etc.
     - Automatically recovers from open modals (e.g. if started while `LOBBY_MODE_SELECTION` or `DUNGEON_CREATE_MENU` is already open, it skips redundant play clicks and proceeds directly).
     - Retries `CREATE_DUNGEON_ORANGE` up to 5 times if modal animation delays response.
     - Halts safely without emitting inputs if an unrecognized error prompt or unknown state persists.

4. **Duplicate Worker Prevention & Responsive Stopping**:
   - `start_live_test` is strictly idempotent: duplicate hotkey presses or button clicks while running are ignored with an audit log.
   - Worker loops check `_stop_event` every 20ms for instant stopping when `,` or `STOP` is pressed.
   - Global hotkeys are unregistered cleanly upon window closing (`WM_DELETE_WINDOW`).

---

## 5. Automated Test Verification

A dedicated automated test `test_ui_has_strictly_two_action_buttons` was added to `tests/test_target_discovery_hotkeys.py`. The test inspects the entire Tkinter widget hierarchy and asserts:
1. Exactly 2 button widgets have `winfo_manager() != ""` in the visible UI.
2. The button texts match `["START LIVE TEST (.)", "STOP (,)"]` exactly.
3. All auxiliary buttons (`btn_emergency_stop`, `btn_clear_log`, `btn_capture_once`, `btn_rec_start`, `btn_rec_stop`) have `winfo_manager() == ""` (strictly unmapped).

### Full Test Suite Results:
```text
Ran 213 tests in 27.126s
OK
```
All 213 unit and integration tests across the test suite passed with zero errors or failures.

---

## 6. Standalone Executable Packaging

The standalone binary was rebuilt using `scripts\build_exe.ps1`:
- **Executable**: `C:\DEV\tools\rb\dist\DungeonQuestMacro\DungeonQuestMacro.exe`
- **Output Bundle**: `C:\DEV\tools\rb\dist\DungeonQuestMacro`
- **Build Status**: Clean build, 0 errors.

---

## 7. User Handoff & Testing Protocol

### Live Testing Instructions:
1. Launch Roblox and enter Dungeon Quest lobby.
2. Launch `dist\DungeonQuestMacro\DungeonQuestMacro.exe`.
3. Verify the simplified UI:
   - Target status updates automatically to `HWND <hwnd> | RobloxPlayerBeta.exe (PID <pid>)`.
   - **Exactly two action buttons** are visible:
     - `START LIVE TEST (.)` (Green)
     - `STOP (,)` (Crimson)
4. Focus into the Roblox window.
5. Press the **`.`** (period) key to start live automation.
6. Verify automated sequence:
   - Green `PLAY` button is clicked.
   - Orange `Create Dungeon` button is clicked.
   - `Winter Outpost` and `Nightmare` are selected.
   - Red `Create Lobby` is clicked.
   - Room is created and macro transitions to `PARTY_STAGING_LOBBY`.
7. Press the **`,`** (comma) key or `F8` at any time to halt automation immediately.

### Failure Evidence:
If any issue occurs, report `dist\DungeonQuestMacro\macro_audit.log`.
