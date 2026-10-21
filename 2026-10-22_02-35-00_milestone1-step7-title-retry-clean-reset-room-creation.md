# Milestone 1 Step 7 / Step 8 Fix: Title Play Retry Lifecycle, Clean Session Reset, and Verified Room-Creation Workflow

**Date & Timestamp**: 2026-10-22 02:35:00 +07:00  
**Project Workspace**: `C:\DEV\tools\rb`  
**Target Environment**: Windows 11, Roblox `Dungeon Quest`, Dungeon: *Winter Outpost (Nightmare)*  
**Executable Artifact**: `dist\DungeonQuestMacro\DungeonQuestMacro.exe`  
**Reports Repository**: `https://github.com/thangnx-mc/reports` (`main` branch)

---

## 1. Executive Summary

This report delivers comprehensive fixes for the root causes identified in investigation report [`2026-10-21_23-55-00_investigation-green-play-room-creation.md`](https://github.com/thangnx-mc/reports/blob/main/2026-10-21_23-55-00_investigation-green-play-room-creation.md).

### Key Deficiencies Resolved:
1. **Title Play Detection ROI Geometry**:
   On non-standard window client viewports (such as the failed session's $1920 \times 991$ letterboxed frame and $1024 \times 768$ viewports), the yellow `title_play_button` center sat at $y = 399$ (relative $y \approx 0.739$). The previous rigid ROI $y \in [0.74, 0.94]$ clipped the button boundary by 1 pixel, causing contour detection to fail and return `None`.
   - **Fix**: Expanded the title Play ROI to $y \in [0.68, 0.95]$ and $x \in [0.38, 0.62]$ with adaptive minimum contour area scaling $(w \times h \times 0.001)$. Verified 100% detection rate across $1920 \times 1080$, $1920 \times 991$, $1024 \times 768$, and $1024 \times 540$.

2. **Title Play Retry Lifecycle & Bounded Backoff**:
   In `WAIT_FOR_LOBBY`, when the title Play detector returned `None` on `TITLE_SPLASH`, the state machine previously did not enforce a minimum backoff delay or verify window focus before falling back to blind coordinates, leading to rapid retry exhaustion or ungrounded clicks.
   - **Fix**: Added bounded retry delays (`title_play_retry_delay_seconds = 2.0s`), detection failure tracking (`_title_play_detection_failures`), and strict fallback preconditions. Calibrated fallback to `(512, 440)` is permitted **only** when `GameState.TITLE_SPLASH` is verified and target window foreground focus is independently confirmed via Win32 API. If the detector continues to fail 3 consecutive times, it engages `Safe Halt` rather than clicking blindly.

3. **Clean Session State Reset across Hotkey Runs**:
   When the user pressed `;` (`START LIVE TEST`) after an earlier run stopped or halted, per-run state variables (including `title_play_clicked`, `title_play_retries`, `title_play_click_time`, `_title_play_detection_failures`, and navigation step counters) were retained. Session 2 immediately inherited Session 1's exhausted retry state.
   - **Fix**: Updated `src/gui_app.py` (`start_live_test`) to call `self.app.state_machine._reset_run_state()` unconditionally on each `;` hotkey trigger, fully clearing all run-scoped counters, click flags, gate memories, and halt reasons.

4. **Context Guards on Lobby Play Detection**:
   The Title Splash screen features a green `Updates` button in the lower-right quadrant, which risked false-positive matching by `play_green_lobby` / `lobby_play_button`.
   - **Fix**: Added 5 strict context guards to `find_ui_element("play_green_lobby")` and `find_ui_element("lobby_play_button")` in `src/vision_engine.py`. Detections are rejected if Title Splash cues (yellow logo $> 3000\text{px}$ or title Play yellow $> 800\text{px}$), Dungeon Create Menu cues (red create button $> 1200\text{px}$), Mode Selection cues, Start Room cues, or Party Staging cues are present.

5. **Disambiguation of the Room-Creation Workflow**:
   Clarified and verified the actual sequence of UI elements from title screen to dungeon start:
   - **Screen 1 (Title Screen)**: Yellow `title_play_button` at calibrated $(512, 465)$ -> transitions to `WAIT_FOR_LOBBY`.
   - **Screen 2 (Lobby Town Square)**: Green `play_green_lobby` at calibrated $(947, 516)$ -> transitions to `SELECTING_DUNGEON`.
   - **Screen 3 (Mode Selection Dialog)**: Orange `create_dungeon_button` at calibrated $(512, 316)$ -> opens Dungeon Create Menu.
   - **Screen 4 (Dungeon Create Menu)**: Red `create_lobby_button` (`create_lobby_red`) at calibrated $(512, 379)$ -> creates Winter Outpost Nightmare room. *(Confirmed: There is NO green Play button on this screen; the button is red).*
   - **Screen 5 (Party Staging Room)**: Yellow `party_start_button` at calibrated $(625, 150)$ -> loads into dungeon map.
   - **Screen 6 (Start Room in Map)**: Green `start_green_in_map` at calibrated $(768, 500)$ -> starts countdown.

All **323** automated unit and integration tests passed (100% pass rate). Standalone executable has been compiled to `dist\DungeonQuestMacro\DungeonQuestMacro.exe`.

---

## 2. Technical Evidence Classification

In compliance with project evidence classification rules:

- `[FACT]`: In `src/vision_engine.py`, `title_play_button` detection ROI is $y \in [0.68, 0.95]$ and $x \in [0.38, 0.62]$. Unit tests in `tests/test_vision_opencv_real.py` prove detection with confidence $\ge 0.90$ across native $(1024 \times 540)$, $1920 \times 1080$, $1920 \times 991$, and $1024 \times 768$.
- `[FACT]`: In `src/vision_engine.py`, `play_green_lobby` / `lobby_play_button` evaluates context guards against `logo_yellow`, `tp_yellow`, `red_create_lobby`, and `orange_mode_selection`. `tests/test_vision_opencv_real.py` confirms that on Title Splash fixture `media_1791518293115.png` and Dungeon Create fixture `media_1791518341534.png`, `play_green_lobby` returns `None`.
- `[FACT]`: In `src/vision_engine.py`, `find_ui_element(img, "create_lobby_red")` accurately localizes the red Create button at $(512, 379)$ with confidence $0.95$ on real fixture `media_1791518341534.png`.
- `[FACT]`: In `src/state_machine.py`, `_reset_run_state()` resets `title_play_clicked`, `title_play_retries`, `title_play_click_time`, `_title_play_last_attempt_time`, `_title_play_detection_failures`, `play_click_retries`, `create_dungeon_click_retries`, `create_lobby_click_retries`, `party_start_click_retries`, and clears all gate memories.
- `[FACT]`: In `src/gui_app.py`, `start_live_test()` calls `self.app.state_machine._reset_run_state()` unconditionally on each `;` hotkey trigger.
- `[FACT]`: Automated test discovery executes 323 passing tests (0 failures, 0 errors, duration 33.3s).
- `[FACT]`: Standalone compilation to `dist\DungeonQuestMacro\DungeonQuestMacro.exe` completed successfully with PyInstaller 6.22.3 (duration 43.6s, executable size 3.92 MB).
- `[REQUIRES EXTERNAL VERIFICATION]`: Live Roblox execution of the full sequence from title screen to lobby, room creation, dungeon load, Gate #1 countdown, and Gate #2 reconnaissance.

---

## 3. UI Navigation Hierarchy Flowchart

```mermaid
flowchart TD
    START[Macro Started via ';' Hotkey] --> CLEAN_RESET[_reset_run_state()<br/>Zero All Run Counters & Flags]
    CLEAN_RESET --> INIT[INITIALIZING State]
    
    INIT --> TITLE[TITLE_SCREEN<br/>Detect GameState.TITLE_SPLASH]
    TITLE --> DETECT_TP{Detect Yellow<br/>title_play_button<br/>ROI y: 0.68-0.95?}
    DETECT_TP -->|Detected| CLICK_TP[Click TITLE_PLAY_BUTTON at Center<br/>Set title_play_clicked = True]
    DETECT_TP -->|None & Focused| FALLBACK_TP[Calibrated Fallback (512, 440)<br/>Requires Verified Foreground Focus]
    DETECT_TP -->|None & Unfocused| BACKOFF_TP[Backoff 2.0s & Retry<br/>Halt after 3 Failures]
    
    CLICK_TP --> WAIT_LOBBY[WAIT_FOR_LOBBY<br/>Awaiting GameState.LOBBY / LOBBY_IDLE]
    FALLBACK_TP --> WAIT_LOBBY
    
    WAIT_LOBBY --> LOBBY[IN_LOBBY<br/>Context Guards Inhibit Green Updates Button]
    LOBBY --> CLICK_LOBBY[Click PLAY_GREEN_LOBBY at (947, 516)]
    
    CLICK_LOBBY --> MODE[SELECTING_DUNGEON<br/>Click Orange CREATE_DUNGEON at (512, 316)]
    MODE --> CREATE_MODAL[CREATING_ROOM<br/>Dungeon Create Modal Opened]
    CREATE_MODAL --> CLICK_RED[Click Red CREATE_LOBBY_RED at (512, 379)<br/>Selects Winter Outpost Nightmare]
    
    CLICK_RED --> STAGING[WAITING_FOR_PARTY<br/>Party Staging Lobby]
    STAGING --> CLICK_PARTY[Click Yellow PARTY_START at (625, 150)]
    CLICK_PARTY --> MAP_LOAD[WAITING_FOR_LOAD / MAP_LOADING<br/>Dungeon Map Loads]
    
    MAP_LOAD --> GATE_START[LOCATE_GATE_1 -> APPROACH_GATE_1<br/>Click Green START_GREEN_IN_MAP at (768, 500)]
    GATE_START --> COUNTDOWN[WAIT_FOR_COUNTDOWN<br/>5s Countdown & Barrier Drop]
    COUNTDOWN --> RECON[ROUTE_RECONNAISSANCE / PASS_A<br/>Recon to Gate #2]
```

---

## 4. Code Modifications Detailed Audit

### 4.1 `src/vision_engine.py`
- **ROI Expansion**:
  Updated `detect_state()` and `find_ui_element("title_play_button")`:
  ```python
  y_start = int(h * 0.68)
  y_end = int(h * 0.95)
  x_start = int(w * 0.38)
  x_end = int(w * 0.62)
  min_area = int((w * h) * 0.001)
  ```
  Prevents boundary clipping on $1920 \times 991$ letterbox and $1024 \times 768$ viewports.
- **Context Guards for Green Lobby Play**:
  In `find_ui_element()` for `play_green_lobby` and `lobby_play_button`:
  - Added Title Splash rejection check: rejects match if yellow DQ logo pixels $\ge 3000$ or title Play yellow pixels $\ge 800$.
  - Added Dungeon Create Menu rejection check: rejects match if red create button pixels $\ge 1200$.
  - Added Mode Selection, Start Room, and Party Staging rejection checks.
- **Red Create Lobby Button Registration**:
  Added dedicated detection and aliases for `create_lobby_red`, `create_lobby_button`, and `create_lobby` targeting the red button at $(512, 379)$ on `media_1791518341534.png`.

### 4.2 `src/state_machine.py`
- **Comprehensive Run Reset (`_reset_run_state`)**:
  Clears:
  - `self.title_play_clicked = False`
  - `self.title_play_retries = 0`
  - `self.title_play_click_time = 0.0`
  - `self._title_play_last_attempt_time = 0.0`
  - `self._title_play_detection_failures = 0`
  - `self.play_click_retries = 0`
  - `self.create_dungeon_click_retries = 0`
  - `self.create_lobby_click_retries = 0`
  - `self.party_start_click_retries = 0`
  - `self.halt_reason = ""`
  - All navigation counters, gate memories, and camera steering states.
- **Bounded Backoff & Guarded Fallback in `WAIT_FOR_LOBBY`**:
  Enforces `retry_delay = 2.0s`. Prevents rapid retry exhaustion when button detector returns `None`. Calibrated fallback to $(512, 440)$ strictly verifies foreground window focus via `input_ctrl.ensure_window_focus()` and confirmed `TITLE_SPLASH` state. Halts safely after 3 consecutive failures without valid focus.

### 4.3 `src/gui_app.py`
- In `start_live_test()`, invoked `self.app.state_machine._reset_run_state()` unconditionally on each `;` hotkey trigger, guaranteeing a clean initial state for every live session.

### 4.4 `tests/test_vision_opencv_real.py`
- Added `test_title_play_button_detection_across_resolutions`: validates yellow title Play detection across $(1024 \times 540)$, $(1920 \times 1080)$, $(1920 \times 991)$, and $(1024 \times 768)$.
- Added `test_play_green_lobby_context_guards_reject_non_lobby_screens`: validates zero false positives on Title Splash and Dungeon Create Menu.
- Added `test_find_create_lobby_red_button_on_create_modal`: validates red Create button localization at $(512, 379)$ with confidence $\ge 0.90$.

### 4.5 `tests/test_lifecycle_rejoin_progress_step7.py`
- Added `test_clean_session_reset_clears_all_run_variables`: validates full reset of all run variables.
- Added `test_session_restart_after_halt_starts_fresh_and_proceeds`: validates that Session 2 started via `;` hotkey begins clean and succeeds without carrying over Session 1 failure counts.
- Added `test_wait_for_lobby_retry_backoff_and_bounded_attempts`: validates retry delay enforcement in `WAIT_FOR_LOBBY`.
- Added `test_wait_for_lobby_fallback_click_strict_preconditions`: validates that fallback click requires verified foreground focus.

---

## 5. Automated Verification Results

Command executed:
```powershell
& "C:\Users\nguye\AppData\Roaming\uv\python\cpython-3.12.13-windows-x86_64-none\python.exe" -m unittest discover -s tests
```

Output:
```text
Ran 323 tests in 33.336s
OK
```

Breakdown of test coverage:
| Test Suite Module | Tests | Result | Notes |
| :--- | :--- | :--- | :--- |
| `tests/test_vision_opencv_real.py` | 19 | PASS | Multi-resolution title Play, red Create button, context guards |
| `tests/test_lifecycle_rejoin_progress_step7.py` | 21 | PASS | Clean reset, retry backoff, foreground focus fallback |
| `tests/test_gui_app.py` | 27 | PASS | UI lifecycle, hotkey registration, observe-only enforcement |
| `tests/test_state_machine.py` | 42 | PASS | Core state transitions, watchdogs, safety halts |
| `tests/test_camera_recovery_step7.py` | 14 | PASS | Camera steering sign adaptation, off-center recovery |
| `tests/test_combat_step7.py` | 24 | PASS | Combat scheduler gating, input safety |
| `tests/test_route_recon_step8.py` | 9 | PASS | Pass A reconnaissance, Gate #2 discovery, manifest persistence |
| Other modules (input, screen, recording) | 167 | PASS | Full regression suite intact |
| **Total** | **323** | **PASS** | **100% Pass Rate** |

---

## 6. Standalone Executable Verification

The application was packaged into a standalone distribution:
- **Build Script**: `.\scripts\build_exe.ps1`
- **Output Executable**: `dist\DungeonQuestMacro\DungeonQuestMacro.exe`
- **Executable Size**: 4,112,726 bytes (3.92 MB)
- **Total Bundle Size**: 175.13 MB (2,023 files)
- **Last Write Timestamp**: `10/22/2026 2:30:34 AM`
- **UI Contract**:
  - Two primary buttons: `START LIVE TEST (;)` and `STOP (')`.
  - Global hotkeys: `;` to start, `'` to stop, `F8` for emergency stop.
  - Safe initialization in OBSERVE-ONLY mode until explicitly started.

---

## 7. Status & User Handoff for Live Verification

### 7.1 Status Matrix
- **Implementation Status**: Complete
- **Automated Verification**: Passed (323/323 tests)
- **External / Runtime Verification**: Pending User Live Test
- **Final Definition of Done**: Satisfied (Code, Tests, Build & Report published; awaiting live-client test results)
- **Overall Task Status**: `IMPLEMENTATION COMPLETE / AUTOMATED VERIFICATION PASS / EXTERNAL VERIFICATION PENDING`

### 7.2 Step-by-Step Live Testing Instructions
1. Open Roblox and navigate to `Dungeon Quest` (either Title Screen or Lobby).
2. Launch the freshly built executable:
   ```text
   C:\DEV\tools\rb\dist\DungeonQuestMacro\DungeonQuestMacro.exe
   ```
3. Ensure the Roblox window is selected in the window combo box.
4. Press `;` (or click `START LIVE TEST (;)`):
   - **From Title Screen**: Macro detects yellow Play button, clicks it, waits for lobby load, clicks green lobby Play, selects Winter Outpost Nightmare, clicks red Create button, starts party, loads dungeon, approaches Gate #1, clicks green Start, awaits 5s countdown, and begins Gate #2 route reconnaissance.
   - **From Lobby**: Macro directly clicks green lobby Play and proceeds through room creation.
5. If at any time you wish to stop, press `'` (or `F8` for emergency stop). All inputs are released immediately.

### 7.3 What to Observe and Report
- If the macro encounters the Title Screen, observe whether the yellow Play button is clicked cleanly without retry exhaustion.
- Verify whether the macro navigates through the mode selection to the red Create button without stalling.
- Report any halt message displayed in the GUI footer or `macro_audit.log`.
