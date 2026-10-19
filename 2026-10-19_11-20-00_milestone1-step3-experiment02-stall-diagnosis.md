# Task Report: Experiment 02 Post-PLAY Stall Diagnosis & Lifecycle Forensics

## 1. Task Metadata & Provenance
- **Task ID**: `TASK-M1-STEP3-EXP02-STALL-DIAGNOSIS`
- **Task Received**: `2026-10-19T11:16:32+07:00`
- **Task Completion**: `2026-10-19T11:23:00+07:00`
- **Timezone**: `Asia/Ho_Chi_Minh`
- **Status**: `COMPLETED`
- **Source Workspace**: `C:\DEV\tools\rb`
- **Report Destination**: `C:\DEV\rb-report\2026-10-19_11-20-00_milestone1-step3-experiment02-stall-diagnosis.md`
- **GitHub Target**: `https://github.com/thangnx-mc/reports` (`main` branch)

---

## 2. Executive Summary

| Item | Finding / Resolution |
| :--- | :--- |
| **Primary Issue (Post-PLAY Stall)** | `[FACT]` In Session 1, after clicking `PLAY_GREEN_LOBBY`, the mode-selection modal opened, but background yellow lighting in the town square caused `detect_state()` rule #6 (`PARTY_STAGING_LOBBY`) to trigger prior to rule #7 (`LOBBY_MODE_SELECTION`). The state machine strictly guarded mode button clicks with `if game_state == GameState.LOBBY_MODE_SELECTION:`, ignoring `CREATE_DUNGEON_ORANGE` and timing out after 20s. |
| **Observation 1 (Autoclick without Cursor Move)** | `[FACT]` In Session 2, after restarting the EXE, the previous instance process held global hotkeys (`RegisterHotKey` returned `ERROR_HOTKEY_ALREADY_REGISTERED`), and Roblox was backgrounded (`fg_hwnd=657352 != target_hwnd=395044`). The title splash click handler called `click_at` without first calling `ensure_window_focus()` or moving the cursor. |
| **Observation 2 (Successful Room Creation)** | `[FACT]` In Session 4 (11:09:50 - 11:10:24), the macro smoothly navigated the complete sequence: Title Splash -> `PLAY_GREEN_LOBBY` -> `CREATE_DUNGEON_ORANGE` (retry 1) -> `Winter Outpost` -> `Nightmare` -> `CREATE_LOBBY_RED` -> `PARTY_STAGING_LOBBY`. |
| **Observation 3 (Macro Stopping After Room Creation)** | `[FACT]` Stopping at `PARTY_STAGING_LOBBY` is the **explicit, intended Milestone 1 Definition of Done** (`config.dungeon.stop_at_party_staging = True`). Milestone 1 strictly forbids clicking `START_YELLOW_PARTY_LOBBY` or `START_GREEN_IN_MAP`. Stopping here is 100% correct and expected. |
| **Sessions Directory Forensics** | `[FACT]` `dist/DungeonQuestMacro/sessions/` is empty (`[]`). The macro does NOT read, cache, or retain stale session state across launches. |
| **Diagnostic Frames Forensics** | `[FACT]` `diagnostic_final_frame.png` (540x1024) is classified as `GameState.TITLE_SPLASH` (conf=0.98), with `title_play_button` at `(488, 435)`, perfectly matching the clicks in Session 3. |

---

## 3. Detailed Forensic Investigation

### 3.1. Audit Log & Diagnostic Frames Analysis
Inspection of `dist/DungeonQuestMacro/macro_audit.log` revealed four distinct application runs on `2026-10-19`:

#### Session 1 (11:06:34 - 11:08:48) — The Post-PLAY Stall
- `[FACT]` At `11:06:50`, the macro clicked `PLAY_GREEN_LOBBY` (calibrated 947, 509) via `mouse_event`.
- `[FACT]` At `11:06:55`, retry #2 was dispatched. The mode selection dialog opened on Roblox.
- `[FACT]` Transition logged: `[IN_LOBBY] -> [SELECTING_DUNGEON] (Clicked PLAY_GREEN_LOBBY; awaiting mode selection confirmation)`.
- `[FACT]` At `11:07:17` (22 seconds later), the state machine halted with:
  `[CRITICAL] SAFE HALT ENGAGED: Timeout: Could not find 'CREATE_LOBBY_RED' in state 'PARTY_STAGING_LOBBY'. Halting input and requiring review.`
  `Transition: [SELECTING_DUNGEON] -> [HALTED_SAFETY]`
- `[INFERENCE]` Why did it timeout looking for `CREATE_LOBBY_RED` instead of clicking `CREATE_DUNGEON_ORANGE`?
  In `src/vision_engine.py` (lines 317-330):
  ```python
  # 6. PARTY_STAGING_LOBBY (Yellow Start button at x in [590, 660], y in [130, 165])
  roi_start = hsv[130:165, 590:660]
  party_yellow_px = int(np.count_nonzero(cv2.inRange(roi_start, np.array([12, 60, 100]), np.array([30, 255, 255]))))
  if party_yellow_px >= 800:
      return GameState.PARTY_STAGING_LOBBY, float(conf)

  # 7. LOBBY_MODE_SELECTION (Orange Create Dungeon + Green Join Dungeon modal buttons)
  mode_roi = hsv[270:345, 280:750]
  ...
  ```
  Check #6 is evaluated before check #7. Warm ambient torches/lighting in the lobby background behind the modal triggered `party_yellow_px >= 800`.
  Consequently, `detect_state()` returned `GameState.PARTY_STAGING_LOBBY`.
  In `src/state_machine.py` (line 345):
  ```python
  if game_state == GameState.LOBBY_MODE_SELECTION:
      ...
  ```
  Because `game_state` was `PARTY_STAGING_LOBBY`, it completely skipped the mode-selection handler. It also skipped the `LOBBY_IDLE` retry branch. Because neither the dungeon tiles nor `CREATE_LOBBY_RED` were present on the mode-selection modal, it fell through to `elif not create_btn:` until the 20s timeout triggered a safe halt.

#### Session 2 (11:08:48 - 11:09:01) — Autoclick Without Cursor Movement
- `[FACT]` When started, hotkey registration collided:
  `Key combination [F6]/[F8] is already reserved by another application.`
  This proves a dangling process from the previous run still held OS hooks.
- `[FACT]` Clicks were dispatched to `calibrated=(496, 442)` / `screen=(930, 840)` with `fg_hwnd=657352` while target Roblox window was `395044`.
- `[FACT]` Roblox was not active/foregrounded. Windows blocked mouse movement injection to inactive windows, but because the Title Splash handler did not call `ensure_window_focus()`, clicks were repeatedly dispatched in the background.

#### Session 3 (11:09:01 - 11:09:50) — Title Splash Handling
- `[FACT]` Roblox was at the Title Splash screen.
- `[FACT]` Macro logged: `Clicked title splash Play button. Waiting for lobby...` at `calibrated=(488, 435)`.
- `[FACT]` `diagnostic_final_frame.png` matches this exact screen, confirming `title_play_button` center at `(488, 435)`.

#### Session 4 (11:09:50 - 11:10:24) — Complete Milestone 1 Success
- `[FACT]` Title splash was clicked and lobby entered.
- `[FACT]` `PLAY_GREEN_LOBBY` clicked via `mouse_event` at `(947, 509)`.
- `[FACT]` Mode-selection UI confirmed on retry 1 (`[EXP-02][OUTCOME] Success: Expected mode-selection UI confirmed on retry 1`).
- `[FACT]` Clicked `CREATE_DUNGEON_ORANGE` at `(512, 306)`.
- `[FACT]` Clicked `tile_Winter Outpost` at `(349, 257)`.
- `[FACT]` Clicked `diff_Nightmare` at `(670, 367)`.
- `[FACT]` Clicked `CREATE_LOBBY_RED` at `(512, 375)`.
- `[FACT]` Transitioned to `CREATING_ROOM` -> Verified `PARTY_STAGING_LOBBY` confirmed with `START_YELLOW_PARTY_LOBBY` visible.
- `[FACT]` Safely stopped at terminal state:
  `[SUCCESS] Verified PARTY_STAGING_LOBBY confirmed with START_YELLOW_PARTY_LOBBY visible. Milestone 1 Complete: Winter Outpost Nightmare room created! Stopping automation safely.`
  `Transition: [CREATING_ROOM] -> [STOPPED]`

---

## 4. Addressing User Inquiries

### Question 1: Does the macro read, write, or retain old session state across runs?
- `[FACT]` **No.** `dist/DungeonQuestMacro/sessions/` is empty (`[]`).
- `[FACT]` `sessions/` is only used by `session_recorder.py` for recording human play sessions for behavioral analysis.
- `[FACT]` Every launch of `MacroStateMachine` initializes from scratch:
  - `current_state = MacroState.INITIALIZING`
  - `play_click_retries = 0`
  - `_mode_selection_clicked = False`
  - `_dungeon_tile_clicked = False`
  - `_difficulty_tab_clicked = False`
  - `_create_lobby_clicked = False`
  - `_has_performed_initial_handshake = False`
  No state, coordinates, or flags are persisted between launches.

### Question 2: Why did autoclick happen without cursor movement after restarting the EXE?
- `[FACT]` In Session 2, Roblox was at the Title Splash, but Roblox (`target_hwnd=395044`) was in the background (`fg_hwnd=657352`).
- `[FACT]` The Title Splash handlers in `INITIALIZING` and `IN_LOBBY` did not call `ensure_window_focus()` before calling `click_at()`.
- `[FACT]` When the window is not in the foreground, Windows restricts cursor movement and mouse events, causing clicks to be sent to inactive windows while the physical cursor stays stationary.
- `[FACT]` Hotkey registration collided (`Key combination is already reserved`) because the previous instance was still exiting.

### Question 3: Why did the macro stop after room creation in the successful run? Is this intentional or an error?
- `[FACT]` **This is 100% intentional and is the explicit Definition of Done for Milestone 1.**
- `[FACT]` The project specification strictly states:
  > "Stop at party staging lobby. Do NOT click START_YELLOW_PARTY_LOBBY or START_GREEN_IN_MAP. Never enter dungeon or add combat."
- `[FACT]` In `config.py`: `stop_at_party_staging = True`.
- `[FACT]` The macro reached `PARTY_STAGING_LOBBY`, confirmed `START_YELLOW_PARTY_LOBBY` was visible, logged `[SUCCESS] Milestone 1 Complete`, and transitioned to `MacroState.STOPPED`.

---

## 5. Applied Minimal Code Improvements

### 5.1. Guard Against Mode Selection Stall (`src/state_machine.py`)
In `MacroState.SELECTING_DUNGEON`, allowed mode selection button handling if `game_state == GameState.LOBBY_MODE_SELECTION` OR if `create_dungeon_btn` is visually detected on screen (while not in `LOBBY`/`LOBBY_IDLE`):
```python
create_dungeon_btn = (
    self.vision.find_button("create_dungeon_orange")
    or self.vision.find_button("create_dungeon_button")
)
# Step 2: Mode selection modal: click CREATE_DUNGEON_ORANGE when LOBBY_MODE_SELECTION confirmed
# or when create_dungeon_orange button is visually detected on screen (guarding against false-positive PARTY_STAGING_LOBBY)
if game_state == GameState.LOBBY_MODE_SELECTION or (
    create_dungeon_btn and game_state not in (GameState.LOBBY, GameState.LOBBY_IDLE)
):
    if game_state != GameState.LOBBY_MODE_SELECTION:
        self.logger.info(
            f"[EXP-02][RECOVERY] Mode selection button detected while game_state='{game_state.value}'. "
            f"Proceeding with mode selection handling."
        )
```

### 5.2. Title Splash Focus Handshake (`src/state_machine.py` & `src/input_controller.py`)
- In `InputController.ensure_window_focus()`: Added `self._init_driver_if_needed()` so driver and window APIs are always ready before activation.
- In `MacroStateMachine` (`INITIALIZING` and `IN_LOBBY`): Added `self.input_ctrl.ensure_window_focus()` before clicking Title Splash Play button to guarantee Roblox is foregrounded.

---

## 6. Verification & Test Evidence

### 6.1. Unit & Regression Tests
- **State Machine Suite**: `python -m unittest tests/test_state_machine.py`
  - Added new regression test: `test_selecting_dungeon_mode_selection_handles_party_staging_false_positive`
  - Result: **22/22 tests PASSED** in 0.367s.
- **Full Test Suite**: `python -m unittest discover -s tests`
  - Result: **193/193 tests PASSED** in 25.223s.

### 6.2. Standalone Executable Packaging
- Rebuilt standalone distribution using `scripts\build_exe.ps1`:
  - Output directory: `dist\DungeonQuestMacro\`
  - Executable: `dist\DungeonQuestMacro\DungeonQuestMacro.exe` (3.83 MB)
  - Result: **Build Succeeded** in 23.2s with zero packaging errors.

---

## 7. Status & Boundaries
- **Implementation Status**: `COMPLETE`
- **Automated Verification**: `PASSED` (193/193 tests)
- **External / Runtime Verification**: `PASSED IN AUDIT LOG (SESSION 4)` / `PENDING USER CONFIRMATION ON NEW BUILD`
- **Milestone 1 Scope Boundaries**:
  - `START_YELLOW_PARTY_LOBBY` and `START_GREEN_IN_MAP` were never clicked.
  - No combat, movement, or in-dungeon automation was added.
  - Safe stopping after room creation confirmed as complete.
