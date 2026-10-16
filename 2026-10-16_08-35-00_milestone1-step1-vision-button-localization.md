# Task Report: Milestone 1 — Step 1: Vision Engine Button Localization

## 1. Task Provenance & Identifiers
- **Task ID**: `TASK-M1-STEP1`
- **Task Name**: Milestone 1, Step 1 — Fix `create_dungeon_button` and `join_dungeon_button` visual localization and add regression tests
- **Timezone**: `Asia/Ho_Chi_Minh` (UTC+7)
- **Task Received Timestamp**: `2026-10-16T08:31:19+07:00` (Step 1 approval), reporting workflow configured at `2026-10-16T08:35:33+07:00`
- **Task Completion Timestamp**: `2026-10-16T08:34:00+07:00` (Implementation & test verification ended)
- **Task Status**: `COMPLETED` (for Step 1 scope)
- **Target Repository**: `C:\DEV\tools\rb` (application), reports tracked in `https://github.com/thangnx-mc/reports`

---

## 2. Scope & Objectives
- **Scope**: Step 1 of Milestone 1 ("Automatically create a Dungeon Quest room").
- **Objective**: Fix the visual detection of `create_dungeon_button` (orange) and `join_dungeon_button` (green) on the `LOBBY_MODE_SELECTION` modal screen using OpenCV contour matching. Add unit/regression tests verifying correct coordinates and zero false positives across fixture frames.
- **Explicit Exclusions**:
  - Do NOT implement Steps 2 through 4 yet.
  - Do NOT click the party header Start button or enter dungeon combat.
  - Do NOT rebuild the executable in `dist/`.
  - Preserve `observe_only=True` and `dry_run=True` invariants.
  - Do NOT delete or modify session recordings or fixture files.

---

## 3. Investigation & Root Cause
Prior to this fix, `src/vision_engine.py` (lines 454–486) searched for `create_dungeon_button` and `join_dungeon_button` using side-by-side ROIs:
- `cd_roi = hsv[270:345, 520:740]` (expected button right of center)
- `jd_roi = hsv[270:345, 280:510]` (expected button left of center)
- Size filter required `bw > 80 and bh > 30`

Analysis of real screenshot fixture `fixtures/real/media_1791518316304.jpg` (`LOBBY_MODE_SELECTION`) revealed that the buttons are actually **vertically stacked and centered** at $x \approx 512$:
- **Join Dungeon (green)**: Bounding box $(461, 272, 103, 23)$, center $(512, 283)$.
- **Create Dungeon (orange)**: Bounding box $(461, 306, 103, 20)$, center $(512, 316)$.

Because the search ROI and height thresholds did not match the real UI elements, `find_ui_element("create_dungeon_button")` returned `None`.

---

## 4. Implementation Details
### 4.1. Source Code Modifications
- `src/vision_engine.py`:
  - `create_dungeon_button`:
    - Search ROI updated to `hsv[295:335, 440:580]`.
    - HSV filter bounds calibrated to $[8, 120, 120]$ through $[25, 255, 255]$.
    - Contour bounding box constraint set to `80 <= bw <= 125 and 14 <= bh <= 28`.
    - Returns `UIElementMatch(name="create_dungeon_button", center_x=cx, center_y=cy, confidence=0.95, bounding_box=(440 + bx, 295 + by, bw, bh))`.
    - Accepted identifier aliases: `create_dungeon_button`, `create_dungeon`, `mode_create_dungeon`.
  - `join_dungeon_button`:
    - Search ROI updated to `hsv[260:300, 440:580]`.
    - HSV filter bounds calibrated to $[35, 80, 80]$ through $[85, 255, 255]$.
    - Contour bounding box constraint set to `80 <= bw <= 125 and 15 <= bh <= 30`.
    - Returns `UIElementMatch(name="join_dungeon_button", center_x=cx, center_y=cy, confidence=0.95, bounding_box=(440 + bx, 260 + by, bw, bh))`.
    - Accepted identifier aliases: `join_dungeon_button`, `join_dungeon`, `mode_join_dungeon`.

### 4.2. Test Suite Additions
- `tests/test_vision_opencv_real.py`:
  - Added `test_find_create_dungeon_button_on_mode_selection`: Verifies `media_1791518316304.jpg` matches `create_dungeon_button` with center near $(512, 316)$ ($\pm 15$ px) and confidence $\ge 0.90$.
  - Added `test_find_join_dungeon_button_on_mode_selection`: Verifies `media_1791518316304.jpg` matches `join_dungeon_button` with center near $(512, 283)$ ($\pm 15$ px) and confidence $\ge 0.90$.
  - Extended `test_button_not_found_on_irrelevant_screen`: Verifies both buttons return `None` on combat frames (`media_1791519239845.png`) and lobby frames (`media_1791518308958.jpg`).

---

## 5. Verification Results

### 5.1. Classification of Evidence
- `[FACT]`: `cv2` contour detection on `media_1791518316304.jpg` yields $(512, 316)$ for orange button and $(512, 283)$ for green button.
- `[FACT]`: Scanning all 18 real fixtures in `fixtures/real/` produces 0 false positive detections for both buttons.
- `[FACT]`: `tests.test_vision_opencv_real` ran 16 tests with 0 failures and 0 errors in 0.936s.
- `[FACT]`: The full automated test suite ran 159 tests with 0 failures and 0 errors in 21.983s.
- `[REQUIRES EXTERNAL VERIFICATION]`: Real-time detection against live Roblox window rendering at arbitrary resolutions or client UI scale variations requires in-game execution.

### 5.2. Test Execution Commands & Outputs
1. **Targeted Real Vision Tests**:
   - Command:
     ```powershell
     & "C:\Users\nguye\AppData\Roaming\uv\python\cpython-3.12.13-windows-x86_64-none\python.exe" -m unittest tests.test_vision_opencv_real
     ```
   - Result:
     ```
     ................
     ----------------------------------------------------------------------
     Ran 16 tests in 0.936s

     OK
     ```

2. **Full Automated Test Suite**:
   - Command:
     ```powershell
     & "C:\Users\nguye\AppData\Roaming\uv\python\cpython-3.12.13-windows-x86_64-none\python.exe" -m unittest discover -s tests
     ```
   - Result:
     ```
     ----------------------------------------------------------------------
     Ran 159 tests in 21.983s

     OK
     ```

---

## 6. Safety & Boundary Invariants
- `OBSERVE_ONLY` & `DRY_RUN`: Retained without alteration. No OS-level mouse clicks or keystrokes were dispatched.
- Packaging: `dist/` and PyInstaller builds were untouched.
- Data Integrity: No session records, recordings, or fixture images were deleted or modified.
- Scope Adherence: Only Step 1 was executed. Steps 2, 3, and 4 remain unstarted pending approval.

---

## 7. Remaining Limitations & Next Steps
- **Limitation**: While `OpenCVVisionProvider.find_ui_element` now reliably localizes `create_dungeon_button`, `MacroStateMachine` currently jumps from `LOBBY_IDLE` to expecting `DUNGEON_CREATE_MENU` without handling the intermediate `LOBBY_MODE_SELECTION` modal step.
- **Next Step (Step 2)**:
  - Add state-confirmed handling for `LOBBY_MODE_SELECTION` in `src/state_machine.py`.
  - When `LOBBY_MODE_SELECTION` is detected in state `SELECTING_DUNGEON`, locate `create_dungeon_button` and dispatch a dry-run click.
  - Implement bounded timeout waiting for transition to `DUNGEON_CREATE_MENU`.
