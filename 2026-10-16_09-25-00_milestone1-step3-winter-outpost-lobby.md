# Task Report: Milestone 1 — Step 3: Automate Winter Outpost Nightmare Lobby Creation

## 1. Task Provenance & Identifiers
- **Task ID**: `TASK-M1-STEP3`
- **Task Name**: Milestone 1, Step 3 — Automate Winter Outpost Nightmare Lobby Creation, Coordinate Translation, Live Opt-In GUI & EXE Packaging
- **Timezone**: `Asia/Ho_Chi_Minh` (UTC+7)
- **Task Received Timestamp**: `2026-10-16T08:50:00+07:00`
- **Task Completion Timestamp**: `2026-10-16T09:27:00+07:00`
- **Task Status**: `COMPLETED` (Automated implementation, test suite pass, distribution build verified; live external verification pending user execution)
- **Target Application Repository**: `C:\DEV\tools\rb`
- **Reports Repository**: `https://github.com/thangnx-mc/reports` (branch `main`)

---

## 2. Scope & Objectives

### 2.1. Core Automation Sequence
When starting the macro with Live Mode enabled, automatically execute the state-confirmed 5-click menu flow required to set up a **Winter Outpost — Nightmare** party lobby:
1. **Initial Lobby**: Click `PLAY_GREEN_LOBBY` (~947, 516) only when `LOBBY_IDLE` or `LOBBY` is confirmed.
2. **Mode Selection Modal**: Click `CREATE_DUNGEON_ORANGE` (~512, 316) only when `LOBBY_MODE_SELECTION` is confirmed.
3. **Dungeon Creation Menu**: When `DUNGEON_CREATE_MENU` is confirmed:
   - Click dungeon tile `Winter Outpost` (~348, 268)
   - Click difficulty tab `Nightmare` (~669, 375)
   - Click button `CREATE_LOBBY_RED` (~512, 379)
4. **Party Staging Lobby Confirmation & Safe Stop**:
   - Verify `PARTY_STAGING_LOBBY` is confirmed and `START_YELLOW_PARTY_LOBBY` (~625, 150) is visible.
   - Transition to `MacroState.STOPPED` and **HALT IMMEDIATELY**.

### 2.2. Strict Safety Boundaries & Prohibitions
- **NEVER click `START_YELLOW_PARTY_LOBBY`**: The macro must NOT start the party game or enter the dungeon.
- **NEVER click `START_GREEN_IN_MAP`**: In-dungeon waiting cave Start button must remain unclicked.
- **Zero Combat / Zero Movement Automation**: No WASD routes or attack loops.
- **Safe Observe-Only Default**: Application defaults to `observe_only=True` and `dry_run=True`. Real input generation requires an explicit GUI opt-in checkbox with confirmation dialog.
- **Native Input Driver**: Uses `ctypes` (`user32.SetCursorPos` and `user32.mouse_event`) so live mouse clicks operate reliably on Windows without external `pydirectinput`.
- **Coordinate Translation**: Calibrated $1024 \times 540$ frame coordinates are dynamically mapped to actual client screen coordinates using `WindowViewportGeometry.calibrated_to_screen`.

---

## 3. Implementation Details

### 3.1. Vision Engine (`src/vision_engine.py`)
- Standardized UI element query aliases inside `OpenCVVisionProvider.find_ui_element`:
  - `play_green_lobby` -> `lobby_play_button` (~947, 516)
  - `create_dungeon_orange` -> `create_dungeon_button` (~512, 316)
  - `create_lobby_red` -> `create_lobby_button` (~512, 379)
  - `start_yellow_party_lobby` -> `party_start_button` (~625, 150)
  - `start_green_in_map` -> `in_game_start_button` (~512, 280)

### 3.2. Native Win32 Input Driver & Coordinate Translation (`src/input_controller.py`)
- Implemented `Win32DirectInputDriver`:
  - Directly invokes Win32 `SetCursorPos(x, y)` and `mouse_event(MOUSEEVENTF_LEFTDOWN / UP)` via `ctypes.windll.user32`.
  - Configured as fallback driver when third-party input libraries (`pydirectinput`) are absent.
- Integrated `coordinate_translator: Optional[Callable[[int, int], Tuple[int, int]]]` callback:
  - Invoked automatically in `click_at(x, y)` before dispatching mouse clicks.
  - Translates $1024 \times 540$ calibration coordinates into absolute desktop screen coordinates based on active Roblox viewport geometry.

### 3.3. Configuration Settings (`src/config.py`)
- Added `stop_at_party_staging: bool = True` to `DungeonConfig` to enforce Milestone 1 boundary.
- Added `enable_live_lobby_creation: bool = False` to `SafetyConfig` to keep live input strictly opt-in.

### 3.4. State Machine Sequencing & Anti-Bounce Locks (`src/state_machine.py`)
- Added anti-bounce click tracking flags:
  - `_mode_selection_clicked`, `_last_mode_selection_click_time`
  - `_dungeon_tile_clicked`, `_difficulty_tab_clicked`, `_create_lobby_clicked`
  - Automatically reset on state transition into `SELECTING_DUNGEON`.
- State-confirmed sequencing:
  - `IN_LOBBY`: Clicks `PLAY_GREEN_LOBBY` only when `LOBBY_IDLE` or `LOBBY` is confirmed.
  - `SELECTING_DUNGEON`:
    - On `LOBBY_MODE_SELECTION`: Clicks `CREATE_DUNGEON_ORANGE` once, waits for modal transition.
    - On `DUNGEON_CREATE_MENU`: Emits clicks for `Winter Outpost` tile, `Nightmare` tab, and `CREATE_LOBBY_RED`, then transitions to `CREATING_ROOM`.
  - `CREATING_ROOM`:
    - Checks for `PARTY_STAGING_LOBBY` and validates visibility of `START_YELLOW_PARTY_LOBBY`.
    - Logs Milestone 1 completion and transitions to `MacroState.STOPPED` without clicking Start when `stop_at_party_staging=True`.
- Evidence collection: `_collect_detector_evidence` audits all standardized button visibility states during continuous observation and session recording.

### 3.5. Desktop GUI Integration (`src/gui_app.py`)
- Added `chk_live_creation` ("Enable Live Lobby Creation (Auto 5 Clicks — Requires Roblox in Foreground)").
- Interactive confirmation modal: Warns user before enabling live mouse clicks; cancels cleanly if rejected.
- Header & Footer reactive styling:
  - Header badge switches between `MODE: OBSERVE-ONLY` and `MODE: LIVE AUTOMATION (5-CLICK LOBBY CREATION)`.
  - Footer switches to `LIVE MODE: 5-Click Lobby Creation ACTIVE`.
  - Start button text switches to `Start Lobby Creation`.
- Coordinate translator lifecycle:
  - When window is selected or live mode starts, automatically binds `selected_window.calibrated_to_screen`.
- Safe completion and halt handling:
  - On `MACRO_COMPLETED` or `MACRO_HALTED`, resets GUI to idle and reverts `var_live_creation` to `False` for safety.
  - In `emergency_stop`, immediately disengages live mode and resets controls to observe-only.
  - Single capture (`capture_and_analyze_once`) forces observe-only dispatch (`_dispatch_observe_only`), guaranteeing zero clicks during manual inspections.

---

## 4. Verification & Evidence Classification

### 4.1. Evidence Tags
- `[FACT]`: Test suite ran 174 automated tests across all modules (`test_gui_app`, `test_state_machine`, `test_vision_state_machine_integration`, `test_behavior_recorder`, etc.) with 0 failures and 0 errors in 23.621 seconds.
- `[FACT]`: `tests/test_vision_state_machine_integration.py` verified the full 5-click sequence on real fixtures (`media_1791518308958.jpg`, `media_1791518316304.jpg`, `media_1791518341534.png`, `media_1791518404135.png`), terminating at `PARTY_STAGING_LOBBY` with exactly 0 clicks on `START_YELLOW_PARTY_LOBBY`.
- `[FACT]`: Packaging script `scripts/build_exe.ps1` rebuilt `dist/DungeonQuestMacro/DungeonQuestMacro.exe` (175.01 MB bundle, 3.82 MB executable) in 23.5 seconds.
- `[FACT]`: Smoke check on `dist/DungeonQuestMacro/DungeonQuestMacro.exe` confirmed clean startup (`Responding: True`, 52.8 MB working set) and clean shutdown.
- `[REQUIRES EXTERNAL VERIFICATION]`: Live Roblox game execution with client window foreground focus to confirm in-game animation timing across real Roblox servers.

### 4.2. Test Execution Results

1. **Full Automated Test Suite**:
   ```powershell
   & "C:\Users\nguye\AppData\Roaming\uv\python\cpython-3.12.13-windows-x86_64-none\python.exe" -m unittest discover -s tests
   ```
   **Result**:
   ```
   Ran 174 tests in 23.621s
   OK
   ```

2. **Standalone Executable Packaging**:
   ```powershell
   powershell -ExecutionPolicy Bypass -File scripts\build_exe.ps1 -PythonPath "C:\Users\nguye\AppData\Roaming\uv\python\cpython-3.12.13-windows-x86_64-none\python.exe"
   ```
   **Result**:
   ```
   Build Succeeded in 23.5 seconds!
   Output Directory: C:\DEV\tools\rb\dist\DungeonQuestMacro
   Main Executable:  C:\DEV\tools\rb\dist\DungeonQuestMacro\DungeonQuestMacro.exe
   Executable Size:  3.82 MB
   Total Bundle Size: 175.01 MB (2022 files)
   ```

3. **Smoke Test of Packaged Executable**:
   ```powershell
   Start-Process -FilePath 'C:\DEV\tools\rb\dist\DungeonQuestMacro\DungeonQuestMacro.exe'
   Get-Process -Name DungeonQuestMacro | Select-Object Id, ProcessName, Responding, WorkingSet64
   Stop-Process -Name DungeonQuestMacro -Force
   ```
   **Result**:
   ```
      Id ProcessName       Responding WorkingSet64
      -- -----------       ---------- ------------
   37392 DungeonQuestMacro       True     52891648
   ```

---

## 5. Artifacts and Files Changed

| File Path | Description of Changes |
| :--- | :--- |
| `src/vision_engine.py` | Added standardized semantic button aliases to `find_ui_element`. |
| `src/input_controller.py` | Added native `Win32DirectInputDriver` and `coordinate_translator` callback. |
| `src/config.py` | Added `stop_at_party_staging` and `enable_live_lobby_creation` config fields. |
| `src/state_machine.py` | Implemented 5-click sequence, anti-bounce flags, and `PARTY_STAGING_LOBBY` safe stop. |
| `src/gui_app.py` | Added Live Lobby Creation checkbox, risk confirmation, badge/footer updates, translator wiring, and completion handlers. |
| `tests/test_state_machine.py` | Added unit tests for 5-click sequence and staging lobby safe stop. |
| `tests/test_vision_state_machine_integration.py` | Added integration tests verifying full fixture-based room creation flow. |
| `tests/test_gui_app.py` | Added unit tests for live toggle lifecycle, coordinate translator integration, and safety reverts. |
| `dist/DungeonQuestMacro/*` | Rebuilt standalone executable distribution incorporating all new modules and dependencies. |

---

## 6. Definition of Done & Status

- **Implementation Status**: `COMPLETE`
- **Automated Verification**: `PASS` (174/174 unit and integration tests passing)
- **Packaging Verification**: `PASS` (Standalone EXE built and smoke tested)
- **External / Runtime Verification**: `PENDING USER TEST`
- **Final Milestone 1 Step 3 Status**: `IMPLEMENTATION COMPLETE / AUTOMATED VERIFICATION PASS / EXTERNAL VERIFICATION PENDING`

---

## 7. User Handoff for External Live Verification

To perform real-game external verification of Milestone 1:

1. **Test Procedure**:
   - Launch Roblox and enter the Dungeon Quest town square lobby. Ensure the window is unminimized and visible.
   - Launch `dist\DungeonQuestMacro\DungeonQuestMacro.exe` (or run `python -m src.gui_app`).
   - Select the Roblox client window from the "Target Window" dropdown.
   - Check the **"Enable Live Lobby Creation"** box and confirm the modal warning.
   - Bring Roblox to the foreground and click **"Start Lobby Creation"** (or press `[F6]`).

2. **Expected Behavior**:
   - The macro sends 5 real clicks:
     1. Green `PLAY` button (~947, 516)
     2. Orange `Create Dungeon` button (~512, 316)
     3. `Winter Outpost` dungeon tile (~348, 268)
     4. `Nightmare` difficulty tab (~669, 375)
     5. Red `Create Lobby` button (~512, 379)
   - When the party staging lobby opens, the macro detects `PARTY_STAGING_LOBBY` and yellow `Start` button, logs `[SUCCESS]`, and stops immediately.
   - **Crucial verification**: The macro does **NOT** click the yellow `Start` button and does **NOT** enter the dungeon.

3. **Failure Criteria**:
   - Misaligned mouse click outside target buttons.
   - Clicking the yellow Start button inside the party lobby.
   - Failing to detect the party staging lobby within timeout.

4. **Emergency Kill-Switch**:
   - Press **`[F8]`** or **`[ESC]`** at any point to instantly engage Emergency Stop and release all inputs.
