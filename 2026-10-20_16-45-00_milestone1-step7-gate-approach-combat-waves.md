# Milestone 1 Step 7: Gate Approach Audit, Combat Automation & Wave Tracking

## Executive Summary

- **Task**: `TASK-M1-STEP7 — Verify Gate Approach, Start Dungeon Combat Automation & Track Enemy Waves`
- **Project**: Roblox Dungeon Quest Winter Outpost Nightmare Macro
- **Workspace**: `C:\DEV\tools\rb`
- **Reports Repo**: `https://github.com/thangnx-mc/reports` (branch `main`)
- **Status Classification**:
  - **Implementation Status**: `Complete`
  - **Automated Verification**: `Passed (253/253 tests pass, 100%)`
  - **Visual Fixture Verification**: `Empirically Verified on Ground-Truth Live Captures`
  - **External / Runtime Verification**: `Pending User Live Verification in Roblox`
  - **Final Task Status**: `IMPLEMENTATION COMPLETE / AUTOMATED VERIFICATION PASS / EXTERNAL VERIFICATION PENDING`

Following the confirmed live testing of Milestone 1 Step 6 (character moves toward the gate, enters the map, and stops at verified exit gate state), Step 7 resolves critical audits from Step 6 and delivers bounded combat automation for the first enemy wave:

1. **Step 6 Implementation Audit**:
   - `[FACT]` *Gate Approach Transitions*: Bounded pulse-based approach (up to 5 discrete 0.35s pulses) with horizontal deviation check ($cx$ within 35% of center) and safety boundary limit ($\text{area} \ge 85,000$). Reaching 5 pulses transitions to `INTERACTING_DUNGEON_GATE` with full key release, where a 10s watchdog protects against a missing Start button.
   - `[FACT]` *Input Release*: Discrete pulses explicitly issue `key_down('w')`, wait `0.35s`, and execute `key_up('w')` with settle delay `0.25s`.
   - `[FACT]` *Countdown & Barrier Drop Protection*: Enforced that barrier disappearance alone cannot prematurely trigger `DUNGEON_STARTED` before $\ge 4.5\text{s}$ countdown elapsed and verified across consecutive confirmation frames ($N \ge 2$), eliminating false-positive premature transitions during the 5-second countdown.
   - `[FACT]` *Start Button Coordinate Scaling & Deduplication*: Normalized green Start button ROI from fixed coordinates to relative viewport fractions (`x in [0.65, 0.85]`, `y in [0.84, 0.99]`), scaling accurately across arbitrary resolutions. Added explicit duplicate click suppression: once clicked, if the Start button visually disappears, the state machine transitions immediately to `GATE_START_COUNTDOWN` with zero retry clicks.
2. **Gate Numbering & Nomenclature**:
   - `[FACT]` **Gate #1**: The initial spawn room entrance gate containing the interactive green `START` button.
   - `[FACT]` **Gate #2**: The first combat barrier blocking advancement into the dungeon corridor (associated with Wave #1).
   - Gate counter and wave counter track progression sequentially (`current_gate_number = 1 -> 2`, `current_wave_number = 1`).
3. **Bounded Combat Automation (`CombatScheduler`)**:
   - Implemented non-blocking, tick-driven `CombatScheduler` running synchronously inside the state machine loop without background thread leaks.
   - Dispatches continuous left-click basic attacks toward the upper-center combat field (`cx=512, cy=300`) at calibrated 0.20s intervals (5 attacks/sec).
   - Dispatches repeated `Q` and `E` skill keypresses at calibrated 3.8s intervals while respecting vision cooldown overlays.
   - Enforces immediate input release (`release_all()`) and scheduler reset upon `STOP`, emergency stop (`F8`), or state transition out of combat.
4. **Conservative Enemy Detection & Snowflake Confetti Rejection**:
   - `[FACT]` Empirically calibrated against user-provided live game screenshots:
     - `media_1792488116520_87cbf96e.png` (active combat): 3 living enemy mobs identified with red health bars at $y \approx 141..148, x \in [326, 425]$ (width $26..29\text{px}$, height $3\text{px}$, aspect ratio $8.67..9.67$, confidence $0.98$).
     - `media_1792488122014_33b717dc.png` (no mobs): Player snowflake cosmetic confetti particles (width $2..6\text{px}$, aspect ratio $0.5..2.0$) are 100% rejected (0 false positives).
   - Health bar detector excludes HUD areas (top-left player card, top boss bar, bottom ability hotbar), applies morphological horizontal close, and requires width $\ge 12\text{px}$, height $\ge 2\text{px}$, area $\ge 20\text{px}$, and aspect ratio $\ge 2.5$.
5. **Multi-Frame Wave Clearance & Safe Halt**:
   - `[FACT]` Wave clearance requires BOTH:
     1. Absence of enemy health bars over $\ge 4$ consecutive frames AND stability window $\ge 2.0\text{s}$.
     2. Confirmation of dropped progression barrier (`barrier_detected is False`).
   - A single frame absence never triggers wave clearance.
   - Upon confirming Wave #1 cleared at Gate #2, macro transitions to `ROOM_CLEARED` -> `STOPPED` with zero autonomous pathfinding, zero movement through the cleared barrier, and zero click spam.
6. **Strict UI Invariant**:
   - Exactly and strictly two action buttons: `START LIVE TEST (.)` and `STOP (,)`.

---

## 1. Visual Evidence & Empirical Fixture Calibration

### 1.1 Empirical Analysis of User Screenshots

| Image Fixture | Ground Truth State | Mobs Present | Health Bar Coordinates $(x, y, w, h)$ | Aspect Ratio | Detector Result | Confetti False Positives |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| `media_1792488116520_87cbf96e.png` ($1024 \times 541$) | Wave 1 Active Combat | 3 living mobs | `(326, 147, 29, 3)`, `(364, 142, 28, 3)`, `(398, 142, 26, 3)` | $9.67$, $9.33$, $8.67$ | **3 enemy bars detected** | 0 |
| `media_1792488122014_33b717dc.png` ($1024 \times 542$) | No Mobs / Confetti Only | 0 mobs | None | N/A | **0 enemy bars detected** | **0 (100% filtered)** |

### 1.2 Discrimination Invariants
- `[FACT]` Red health bars above mobs are wide, thin rectangles: HSV $H \in [0, 10] \cup [170, 180]$, $S \ge 110$, $V \ge 90$, with aspect ratio $w/h \ge 2.5$ (typically $8.5..9.7$) and width $\ge 24\text{px}$.
- `[FACT]` Confetti particles generated by player cosmetics or snowflake effects have aspect ratios near square ($0.5..2.0$) and narrow widths ($2..6\text{px}$). Enforcing $w \ge 12\text{px}$ and aspect ratio $\ge 2.5$ provides complete separation with zero false triggers.
- `[FACT]` Boss health bar at top center is filtered by width $> 25\%$ viewport width ($>250\text{px}$) and top-center positioning ($y \le 35\%$).

---

## 2. Architecture & Detailed Implementation

```
[APPROACHING_DUNGEON_GATE] (Gate #1 Entrance)
       │  ◄── Feedback-driven 'w' pulses (0.35s)
       │  • Max safe area (>=85,000) ────────────────┐
       │  • Target area (>=20,000) ──────────────────┤
       │  • Max steps reached (5 pulses) ────────────┤
       │                                             ▼
       └───────────────────────────────────► [INTERACTING_DUNGEON_GATE]
                                                     │
                                                     ▼ (Click START_GREEN_IN_MAP at 768, 496)
                                             [GATE_START_COUNTDOWN]
                                                     │ (ZERO movement, ZERO clicks, >=4.5s elapsed)
                                                     ▼ (Barrier dropped & active dungeon confirmed)
                                             [DUNGEON_STARTED]
                                                     │ (stop_at_dungeon_started: False)
                                                     ▼ (Set Gate #2, Wave #1; Start Combat Scheduler)
                                             [ROOM_COMBAT]
                                                     │  ◄── Continuous left-clicks (0.20s interval)
                                                     │  ◄── Periodic Q / E skills (3.8s interval)
                                                     │  ◄── Live enemy bar detection & tracking
                                                     │
                                                     ▼ (Enemies absent >= 4 frames + barrier dropped)
                                             [ROOM_CLEARED]
                                                     │ (stop_at_wave_cleared: True)
                                                     ▼ (Safe Halt: Zero movement, Zero combat)
                                                 [STOPPED]
```

### 2.1 Non-Blocking Combat Scheduler (`src/combat_scheduler.py`)
`CombatScheduler` implements tick-based, non-blocking input dispatching:
- **Left-Click Basic Attacks**: Dispatched to $(512, 300)$ at configured `left_click_interval_seconds` (default $0.20\text{s}$).
- **Q / E Skills**: Dispatched at configured `q_interval_seconds` and `e_interval_seconds` (default $3.8\text{s}$), with cooldown checking via `is_skill_on_cooldown`.
- **Immediate Input Release**: `reset(input_ctrl)` halts scheduling and releases all pressed mouse/keyboard keys instantly upon stop.

### 2.2 Vision Engine Extensions (`src/vision_engine.py`)
- Added `EnemyHealthBarMatch` dataclass: `center_x`, `center_y`, `bounding_box`, `width`, `height`, `aspect_ratio`, `confidence`.
- Implemented `detect_enemy_health_bars(frame)` in `OpenCVVisionProvider`, `SyntheticVisionProvider`, and `VisionEngine`.
- Updated `find_ui_element` for `start_green_in_map` to use normalized relative viewport coordinates:
  - $x \in [0.65 \cdot W, 0.85 \cdot W]$, $y \in [0.84 \cdot H, 0.99 \cdot H]$.
  - Output coordinates map to actual window coordinates at any resolution.

### 2.3 State Machine Progression (`src/state_machine.py`)
- **Gate Approach Audit Fix**: `APPROACHING_DUNGEON_GATE` halts safely on severe horizontal orientation deviation ($>35\%$ frame deviation).
- **Start Click Deduplication**: `INTERACTING_DUNGEON_GATE` immediately advances to `GATE_START_COUNTDOWN` when the Start button visually disappears, suppressing duplicate clicks.
- **Countdown Safety**: `GATE_START_COUNTDOWN` strictly enforces countdown elapsed $\ge 4.5\text{s}$ before barrier drop alone can confirm dungeon start.
- **Transition to Combat**: `DUNGEON_STARTED` sets `current_gate_number = 2`, `current_wave_number = 1`, starts `combat_scheduler`, and transitions to `ROOM_COMBAT`.
- **Wave Clearance & Safe Halt**: `ROOM_COMBAT` evaluates multi-signal clearance via `RoomProgressionDetector`. When $\ge 4$ consecutive empty frames and open barrier are verified, transitions to `ROOM_CLEARED` and stops safely at `STOPPED`.

### 2.4 Strict UI Invariant (`src/gui_app.py`)
- UI contains strictly two action buttons: `START LIVE TEST (.)` and `STOP (,)`.
- `start_live_test` automatically configures Step 7 runtime flags (`stop_at_dungeon_started = False`, `stop_at_wave_cleared = True`).
- `stop_live_test` and `emergency_stop` immediately reset `combat_scheduler` and release all inputs.

---

## 3. Automated Verification & Test Results

The complete test suite of **253 tests** passed with 0 failures and 0 errors:

```text
Ran 253 tests in 28.944s

OK
```

### 3.1 Step 7 Targeted Test Suite (`tests/test_combat_step7.py`)

| Test Case | Objective | Result |
| :--- | :--- | :--- |
| `test_inactive_scheduler_does_nothing` | Verify inactive combat scheduler emits 0 inputs | **PASS** |
| `test_scheduler_starts_and_dispatches_actions` | Verify scheduler dispatches attack, Q, and E at scheduled intervals | **PASS** |
| `test_cooldown_overlay_suppresses_skill` | Verify skill cast is suppressed when vision detects numeric cooldown | **PASS** |
| `test_reset_halts_and_releases_all_inputs` | Verify scheduler reset immediately stops active state and releases keys | **PASS** |
| `test_approach_pulse_exhaustion_transitions_to_interaction` | Audit Item 2: Reaching 5 approach steps transitions to interaction with input release | **PASS** |
| `test_approach_off_center_gate_safely_halts` | Audit Item 2: Deviated gate orientation triggers safe halt | **PASS** |
| `test_start_button_disappearing_after_click_advances_to_countdown` | Audit Item 5: Disappearing Start button advances to countdown without duplicate click | **PASS** |
| `test_single_missed_barrier_during_early_countdown_does_not_confirm_start` | Audit Item 3: Barrier drop before 4.5s countdown elapsed prevents premature start | **PASS** |
| `test_countdown_elapsed_and_multi_frame_confirms_dungeon_start` | Audit Item 3: Countdown >= 4.5s + dropped barrier confirms dungeon start | **PASS** |
| `test_initial_gate_numbering` | Verify macro starts at Gate #1 (entrance gate with Start button) | **PASS** |
| `test_dungeon_started_advances_to_gate2_wave1_combat` | Verify DUNGEON_STARTED advances to Gate #2, Wave #1 ROOM_COMBAT | **PASS** |
| `test_single_frame_absence_does_not_clear_wave` | Verify 1 empty frame returns STABILITY_WAIT and does not clear wave | **PASS** |
| `test_multi_frame_absence_and_barrier_drop_clears_wave` | Verify >= 4 empty frames + stability window + barrier drop clears wave | **PASS** |
| `test_room_cleared_safely_halts_when_stop_at_wave_cleared_is_true` | Verify ROOM_CLEARED halts macro at STOPPED and resets scheduler | **PASS** |
| `test_enemy_health_bar_detection_on_fixture_with_living_enemies` | Verify 3 living enemy health bars detected on ground-truth fixture | **PASS** |
| `test_snowflake_confetti_rejection_on_fixture_without_enemies` | Verify snowflake confetti is 100% rejected (0 false positives) | **PASS** |

---

## 4. Standalone Executable Build

Built standalone Windows executable using PyInstaller 6.22.3:
- **Build Script**: `scripts\build_exe.ps1`
- **Output Executable**: `C:\DEV\tools\rb\dist\DungeonQuestMacro\DungeonQuestMacro.exe`
- **Executable Size**: 3.87 MB
- **Total Bundle Size**: 175.06 MB (2,022 files)
- **Configuration**: Bundled `dist\DungeonQuestMacro\config.json` synchronized with Step 7 settings (`stop_at_dungeon_started: false`, `stop_at_wave_cleared: true`, `consecutive_empty_frames_required: 4`).

---

## 5. User Handoff & Live Verification Protocol

`[REQUIRES EXTERNAL VERIFICATION]`

To verify Milestone 1 Step 7 in the live Roblox client:

### 5.1 Test Instructions
1. Open Roblox and enter Dungeon Quest town square lobby.
2. Launch `dist\DungeonQuestMacro\DungeonQuestMacro.exe`.
3. Verify GUI displays strictly two action buttons: `START LIVE TEST (.)` and `STOP (,)`.
4. Press `.` (or click `START LIVE TEST (.)`).
5. Observe automated progression:
   - Macro clicks `PLAY_GREEN_LOBBY`, selects `Winter Outpost` / `Nightmare`, creates party, and clicks yellow `START`.
   - Character arrives in cave lobby (Gate #1), approaches gate barrier, clicks green `START_GREEN_IN_MAP`.
   - Macro waits during 5-second countdown with zero inputs.
   - Upon countdown conclusion and barrier drop, macro transitions to Gate #2 / Wave #1 combat.
   - Continuous left-click attacks and Q/E skill activations engage enemies.
   - Upon defeating all mobs and barrier drop at Gate #2, macro stops safely.

### 5.2 Expected Behavior
- **Gate Approach**: Character moves forward smoothly toward Gate #1 with short `w` pulses.
- **Start Click**: Clicks green Start button once; button disappears; countdown starts without click spam.
- **Combat Engagement**: When Gate #1 disappears and dungeon begins, character starts left-click attacking and casting Q/E.
- **Wave Clearance**: While enemies are alive (red bars visible), combat continues. When all 3 mobs die and Gate #2 barrier drops, combat ceases within 2.0s and macro transitions to `STOPPED`.

### 5.3 What Counts as Failure
- Character continuing to attack after Wave 1 is cleared and Gate #2 drops.
- Character attempting to walk through Gate #2 before user approval.
- Macro clicking during the 5-second countdown.
- Failure to attack or cast skills during active combat.
- Premature wave clearance while mobs are still alive.

### 5.4 Reporting Evidence
Observe and note:
1. Did Gate #1 Start click work on the first attempt?
2. Did left-click attacks and Q/E skills cast continuously during Wave 1?
3. Did the macro stop cleanly after Wave 1 was defeated?
4. Attach `dist\DungeonQuestMacro\macro_audit.log` from the session.
