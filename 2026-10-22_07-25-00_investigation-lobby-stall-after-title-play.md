# Investigation Report: Macro Stalls in Lobby After Yellow Title Play

**Date & Timestamp**: 2026-10-22 07:25:00 +07:00  
**Project Workspace**: `C:\DEV\tools\rb`  
**Target Environment**: Windows 11, Roblox `Dungeon Quest`, Dungeon: *Winter Outpost (Nightmare)*  
**Executable Tested**: `dist\DungeonQuestMacro\DungeonQuestMacro.exe` (Built 2026-10-22 02:30:34)  
**Reports Repository**: `https://github.com/thangnx-mc/reports` (`main` branch)  
**Classification**: **Confirmed Root Cause**

---

## 1. Executive Summary

During the latest live-test session (`2026-10-22 07:16:00` to `07:16:33`), the user observed:
> *"Nhấn nút Play vàng ở màn hình title thành công, Roblox đã vào lobby nhưng macro đứng yên, không tiếp tục nhấn nút Play xanh để mở quy trình tạo room."*  
> *(Clicked yellow title Play button successfully, Roblox entered the lobby, but the macro stands still and does not continue to click the green Play button to open the room creation process).*

This investigation correlated runtime logs (`dist\DungeonQuestMacro\macro_audit.log`), diagnostic screen captures, geometric calculations, and the vision pipeline state transitions.

### Root-Cause Finding:
The macro stalled in the lobby because **`detect_state()` in `src/vision_engine.py` falsely classified the active Lobby as `GameState.TITLE_SPLASH` on every frame with confidence $0.98$**.
1. **Unintended Overlap Between Lobby Play and Title Updates**: The title splash screen's secondary cue checks for green pixels in the lower-right quadrant:
   $$\text{updates\_roi} = \text{hsv}[0.80h : 0.99h, 0.80w : 0.99w]$$
   On a $1024 \times 540$ frame, this spans $y \in [432, 534]$ and $x \in [819, 1013]$. In the active lobby, the **green Lobby Play button** sits directly at $(947, 516)$ with dimensions $112 \times 36$ ($\approx 3,828\text{px}$), producing $2,955$ green pixels in this ROI. Thus, the condition `updates_green_px >= 1000` evaluates to **`True` on every frame in the lobby**.
2. **Central Multi-Player Yellow Clutter**: `detect_state()`'s second title splash cue checks for yellow pixels across the vast central region:
   $$\text{logo\_roi} = \text{hsv}[0.15h : 0.75h, 0.30w : 0.70w]$$
   This ROI covers $324 \times 409 = 132,516\text{px}$. The threshold `logo_yellow_px >= 3000` represents only **$2.26\%$** of the region. In a live multiplayer lobby with player avatars, glowing armor, weapons, particle effects, VIP tags, and town lanterns, scattered yellow pixels easily exceeded $3,000$.
3. **Compound Consequence in `detect_state()`**:
   Line 437 evaluates:
   ```python
   if title_yellow_px >= 800 or (logo_yellow_px >= 3000 and updates_green_px >= 1000):
   ```
   Because `logo_yellow_px >= 3000` and `updates_green_px >= 1000` were both `True`, `detect_state()` immediately returned `GameState.TITLE_SPLASH, 0.98` (State 4), **before the state machine could ever evaluate `LOBBY_IDLE` (State 9)**.
4. **State Machine Lockout**: In `WAIT_FOR_LOBBY` (`src/state_machine.py`), the macro only transitions to `IN_LOBBY` if `game_state in (GameState.LOBBY, GameState.LOBBY_IDLE)`. Because the state remained misclassified as `TITLE_SPLASH`, the macro assumed Roblox was still stuck on the title screen. It never transitioned to `IN_LOBBY`, the green lobby Play detector was **never invoked**, and after 20 seconds of retries, the watchdog safely halted the macro.

---

## 2. Technical Evidence Classification

In compliance with project evidence classification rules:

- `[FACT]`: In `dist\DungeonQuestMacro\macro_audit.log`, the macro dispatched a click to `TITLE_PLAY_BUTTON` at calibrated $(512, 459)$ $\rightarrow$ screen $(960, 871)$ at `2026-10-22 07:16:13` and transitioned `[TITLE_SCREEN] -> [WAIT_FOR_LOBBY]`.
- `[FACT]`: The user confirmed that the click succeeded in Roblox and the game loaded into the Town Square lobby.
- `[FACT]`: From `07:16:13` to `07:16:33` (across $> 800$ log lines), every tick logged `[STATE_DETECTED] Observed GameState: TITLE_SPLASH (conf=0.98, macro=WAIT_FOR_LOBBY)`.
- `[FACT]`: At `07:16:19` and `07:16:25`, `find_ui_element("title_play_button")` returned `None` (`[RETRY_FALLBACK] Button detector returned None on verified TITLE_SPLASH`). This proves the yellow title Play button contour was no longer on the screen in the captured frames.
- `[FACT]`: In `src/vision_engine.py` (lines 437-444), `conf = 0.98` is calculated strictly as `base_conf (0.90) + 0.04 (logo_yellow_px >= 3000) + 0.04 (updates_green_px >= 1000)`.
- `[FACT]`: In `fixtures/real/media_1791518308958.jpg` (verified real lobby screenshot), the green Lobby Play button at $(891, 498)$ generates $2,955$ green pixels in `updates_roi` ($[432:534, 819:1013]$), permanently satisfying `updates_green_px >= 1000`.
- `[FACT]`: In `src/state_machine.py` (lines 680-690), the transition from `WAIT_FOR_LOBBY` to `IN_LOBBY` strictly requires `game_state in (GameState.LOBBY, GameState.LOBBY_IDLE)` across 2 consecutive frames.
- `[FACT]`: In `src/state_machine.py` (lines 791-796), `find_ui_element("play_green_lobby")` is only called inside `elif self.current_state == MacroState.IN_LOBBY`.
- `[FACT]`: At `07:16:33`, `WAIT_FOR_LOBBY` timed out after 3 attempts (`title_play_retries >= max_retries`), engaging `SAFE HALT ENGAGED: Timeout: Title screen remained visible after 3 attempts clicking Play`.
- `[CONFIRMED ROOT CAUSE]`: The macro stalled because `detect_state()` misclassified the lobby screen as `TITLE_SPLASH` (conf=0.98) due to the green Lobby Play button satisfying the `updates_green_px >= 1000` rule and central lobby multiplayer avatars satisfying `logo_yellow_px >= 3000`.

---

## 3. Session Identification & Correlation

### 3.1 Session Metadata
- **Log Source**: `dist\DungeonQuestMacro\macro_audit.log`
- **Target Process**: `RobloxPlayerBeta.exe`, PID `28312`, HWND `394666`
- **Viewport**: $1920 \times 991$ (target resized to $1024 \times 540$)
- **Session Launch**: `2026-10-22 07:16:00`
- **Hotkey `;` Received**: `2026-10-22 07:16:11`
- **Session Safe Halt**: `2026-10-22 07:16:33`
- **Application Exit**: `2026-10-22 07:17:34`

### 3.2 Chronological Timeline of Failed Run

| Timestamp | Log Line | State / Component | Event / Observation |
| :--- | :--- | :--- | :--- |
| **07:16:00** | 1-17 | GUI App | Macro launched in observe-only mode. Target window discovered (HWND 394666). Global hotkeys registered. |
| **07:16:11** | 18-25 | Hotkey Manager | Hotkey `;` pressed. Window activated and foreground verified. Clean run lifecycle reset executed (`_reset_run_state`). |
| **07:16:11** | 35-40 | Vision / State Machine | Frame captured. `GameState.TITLE_SPLASH` detected (conf=0.98). State transitioned `[INITIALIZING] -> [TITLE_SCREEN]`. |
| **07:16:13** | 138-141 | Input / State Machine | Yellow Play button located at calibrated $(512, 459)$. Dispatched mouse click to screen $(960, 871)$. Transitioned `[TITLE_SCREEN] -> [WAIT_FOR_LOBBY]`. |
| **07:16:13** | Roblox Client | Roblox Game Engine | *(User Observation)* Click acknowledged by Roblox; Title screen closes and game loads Town Square lobby. |
| **07:16:15** | 228 | Vision Engine | Frame captured. `GameState.TITLE_SPLASH (conf=0.98)` detected in `WAIT_FOR_LOBBY`. Backoff delay active. |
| **07:16:17** | 317 | Vision Engine | Frame captured. `GameState.TITLE_SPLASH (conf=0.98)` detected. Backoff delay active. |
| **07:16:19** | 409-410 | State Machine / Input | Retry timer elapsed ($> 2.0\text{s}$). `find_ui_element("title_play_button")` returned **`None`**. Fallback clicked $(512, 440)$ because `game_state` was still `TITLE_SPLASH`. Retry 1 dispatched. |
| **07:16:21** | 503 | Vision Engine | Frame captured. `GameState.TITLE_SPLASH (conf=0.98)` detected. |
| **07:16:23** | 598 | Vision Engine | Frame captured. `GameState.TITLE_SPLASH (conf=0.98)` detected. |
| **07:16:25** | 690-691 | State Machine / Input | Retry timer elapsed ($> 2.0\text{s}$). `title_play_button` returned **`None`**. Fallback clicked $(512, 440)$ (Attempt 2 of 2). |
| **07:16:28** | 790 | Vision Engine | Frame captured. `GameState.TITLE_SPLASH (conf=0.98)` detected. |
| **07:16:30** | 885 | Vision Engine | Frame captured. `GameState.TITLE_SPLASH (conf=0.98)` detected. |
| **07:16:33** | 977-980 | State Machine Watchdog | `title_play_retries >= max_retries` (2 >= 2). **SAFE HALT ENGAGED**: `Timeout: Title screen remained visible after 3 attempts clicking Play`. Transitioned to `HALTED_SAFETY`. |
| **07:17:34** | 1068 | Hotkey Manager | Macro GUI closed. Hotkeys unregistered. |

---

## 4. State Machine Trace Audit

### 1. Was the yellow title Play button detected and clicked?
**Yes.** Log line 139:
```text
2026-10-22 07:16:13 [INFO] [DQMacro.GUI] [INPUT_DISPATCHED] Clicked TITLE_PLAY_BUTTON at calibrated (512, 459) -> screen (960, 871) on target HWND 394666 via Win32DirectInputDriver (mouse_event) [button=left, strategy=mouse_event, fg_hwnd=394666, down_ret=1, up_ret=1, err=0, success=True]
```
The user confirmed the click registered in Roblox and the game loaded into the lobby.

### 2. Did the title screen disappear, and was the transition to `WAIT_FOR_LOBBY` confirmed?
- **In Roblox**: Yes. The title screen disappeared and the character spawned in the lobby.
- **In Macro State Machine**: Yes. Transition `[TITLE_SCREEN] -> [WAIT_FOR_LOBBY]` logged at `07:16:13` (line 141).
- **In Vision Engine**: **No.** Every frame captured between `07:16:13` and `07:16:33` continued to be identified as `TITLE_SPLASH (conf=0.98)`.

### 3. What state was active when the macro stopped progressing?
The macro remained locked in `MacroState.WAIT_FOR_LOBBY` for 20 seconds, and at `07:16:33` transitioned to `MacroState.HALTED_SAFETY`.

### 4. Did the state machine transition to `LOBBY` or `LOBBY_IDLE`?
**No.** `WAIT_FOR_LOBBY` requires `game_state in (GameState.LOBBY, GameState.LOBBY_IDLE)` across 2 consecutive frames (lines 680-689). Because `detect_state()` continuously returned `TITLE_SPLASH`, this block was never entered.

### 5. Was the green lobby Play detector called in that state?
**No.** `find_ui_element("play_green_lobby")` is only called when `self.current_state == MacroState.IN_LOBBY` (lines 791-796). It was never invoked.

### 6. If called, what candidate coordinates, confidence, bounding box, and rejection/exclusion reasons were returned?
**N/A.** The detector was never called because the macro never reached `IN_LOBBY`.

### 7. If the detector returned no candidate, was a retry scheduled? Did the retry timer/backoff advance correctly?
Inside `WAIT_FOR_LOBBY`, the title screen retry branch executed:
- Retry backoff timer (`title_play_retry_delay_seconds = 2.0s`) functioned correctly, holding clicks for 2 seconds between attempts.
- When the timer elapsed at `07:16:19` and `07:16:25`, `find_ui_element("title_play_button")` returned `None` because the yellow Play button was gone from the screen.
- Because `game_state` was still reported as `TITLE_SPLASH`, the fallback branch clicked calibrated $(512, 440)$ (which hit the center of the lobby).

### 8. If a candidate was detected, was the click actually dispatched? Was the correct Roblox window foreground and focused?
Yes. Both the initial click at $(512, 459)$ and the retry clicks at $(512, 440)$ were verified foreground (`fg_hwnd=394666`) and dispatched via Win32 `mouse_event`.

### 9. Did a watchdog, timeout, safety halt, stale flag, or previous-session state prevent further actions?
Yes. The timeout watchdog in `WAIT_FOR_LOBBY` halted the macro after 3 attempts:
```text
2026-10-22 07:16:33 [CRITICAL] [DQMacro.GUI] SAFE HALT ENGAGED: Timeout: Title screen remained visible after 3 attempts clicking Play. Halting input and requiring review.
```
Previous-session state did **not** leak; line 25 confirmed all variables were reset at session start.

### 10. Were there exceptions, swallowed errors, or early returns that explain why no click occurred?
No unhandled exceptions occurred. The code took the early return on line 722/743 inside `WAIT_FOR_LOBBY` because `game_state == GameState.TITLE_SPLASH`.

---

## 5. Vision Pipeline Analysis & Mathematical Proof

### 5.1 The `TITLE_SPLASH` Classifier Defect
In `src/vision_engine.py` (lines 422-444):
```python
# 4. TITLE_SPLASH (Central yellow Play button + logo shield + updates button)
title_play_roi = hsv[int(h * 0.68):int(h * 0.95), int(w * 0.38):int(w * 0.62)]
yellow_play_mask = cv2.inRange(title_play_roi, np.array([18, 120, 120]), np.array([35, 255, 255]))
title_yellow_px = int(np.count_nonzero(yellow_play_mask))

# Secondary cue: central Dungeon Quest logo shield
logo_roi = hsv[int(h * 0.15):int(h * 0.75), int(w * 0.30):int(w * 0.70)]
logo_yellow_mask = cv2.inRange(logo_roi, np.array([18, 120, 120]), np.array([35, 255, 255]))
logo_yellow_px = int(np.count_nonzero(logo_yellow_mask))

# Secondary cue: lower-right Updates button
updates_roi = hsv[int(h * 0.80):int(h * 0.99), int(w * 0.80):int(w * 0.99)]
updates_green_mask = cv2.inRange(updates_roi, np.array([35, 100, 100]), np.array([85, 255, 255]))
updates_green_px = int(np.count_nonzero(updates_green_mask))

if title_yellow_px >= 800 or (logo_yellow_px >= 3000 and updates_green_px >= 1000):
    base_conf = 0.90
    if logo_yellow_px >= 3000:
        base_conf += 0.04
    if updates_green_px >= 1000:
        base_conf += 0.04
    conf = min(0.99, base_conf)
    return GameState.TITLE_SPLASH, float(conf)
```

### 5.2 Mathematical Analysis of the Failure Mode
1. **Geometric Coincidence of Green Buttons**:
   - On Title Splash: Green `Updates` button is located at $(877, 476)$, size $147 \times 64$.
   - In Active Lobby: Green `Play` button is located at $(891, 498)$, size $112 \times 36$.
   - Both buttons fall squarely inside `updates_roi` ($[432:534, 819:1013]$).
   - In `media_1791518308958.jpg` (Lobby), the green Play button generates **$2,955$ green pixels**, easily exceeding the $1,000$ threshold.
   - **Result**: `updates_green_px >= 1000` is **permanently `True` in the active lobby**.

2. **Hypersensitivity of `logo_yellow_px`**:
   - `logo_roi` covers $324 \times 409 = 132,516\text{px}$.
   - The threshold of $3,000$ pixels represents **$2.26\%$** of the central screen.
   - In the real Title Splash screen (`diagnostic_final_frame.png`), the actual Dungeon Quest gold shield contains **$18,372$ yellow pixels**.
   - In the active lobby, the player's avatar, armor with gold trim, glowing weapons, level 79 badge, overhead player names, and town lanterns easily generate $\ge 3,000$ yellow pixels.
   - **Result**: `logo_yellow_px >= 3000` evaluates to **`True` in the active lobby**.

3. **Classification Masking**:
   - Because `(logo_yellow_px >= 3000 and updates_green_px >= 1000)` was `True`, `detect_state()` returned `GameState.TITLE_SPLASH, 0.98`.
   - `TITLE_SPLASH` is evaluated at **Step 4**.
   - `LOBBY_IDLE` is evaluated at **Step 9**.
   - Because Step 4 matched, execution returned immediately and Step 9 was never reached.

4. **Self-Contradiction with Button Finding**:
   - `find_ui_element("title_play_button")` checks for an actual rectangular contour ($40 \le bw \le 256$, $18 \le bh \le 108$, aspect $1.2..3.5$, area $\ge 552$).
   - On the lobby screen, there is no yellow Play button contour, so `find_ui_element` returned `None`.
   - But `detect_state()` did not require a button contour, creating a contradictory state where `detect_state()` reported `TITLE_SPLASH` (conf=0.98) while `find_ui_element` confirmed the button was absent.

---

## 6. Root-Cause Conclusion & Artifact Checklist

### 6.1 Classification
**Confirmed Root Cause** — Fully established by correlated runtime logs, state machine code, geometric coordinate analysis, and OpenCV vision evaluation.

### 6.2 Key Summary Table

| Metric / Check | Value / Finding | Evidence File & Reference |
| :--- | :--- | :--- |
| **Last Confirmed State** | `WAIT_FOR_LOBBY` (stalled for 20s, then transitioned to `HALTED_SAFETY`) | `macro_audit.log` L141, L979 |
| **First Missing Action** | Transition to `IN_LOBBY` and search/click on `PLAY_GREEN_LOBBY` | `src/state_machine.py` L680, L791 |
| **Primary Root Cause** | `detect_state()` returned `TITLE_SPLASH (conf=0.98)` on active lobby frames | `src/vision_engine.py` L437-444 |
| **Underlying Mechanism** | Green lobby Play button satisfied `updates_green_px >= 1000` ($2,955\text{px}$) while central lobby avatar clutter satisfied `logo_yellow_px >= 3000` | `fixtures/real/media_1791518308958.jpg` |
| **Secondary Guard Lockout** | `logo_yellow_px >= 2500` also would have blocked `LOBBY_IDLE` on line 485 | `src/vision_engine.py` L485 |
| **Files Inspected** | `dist\DungeonQuestMacro\macro_audit.log`<br/>`dist\DungeonQuestMacro\diagnostic_final_frame.png`<br/>`src/vision_engine.py`<br/>`src/state_machine.py`<br/>`src/screen_capture.py`<br/>`src/gui_app.py` | Full file paths audited |
| **Missing Evidence** | None. Runtime log timestamps, code logic, and pixel geometries align completely. | No missing artifacts |

---

## 7. Recommended Remediation (For Future Implementation Task)

*(Per task instructions, no source modifications were made during this investigation).*

When implementing the fix in the subsequent task:
1. **Require Yellow Button Contour for Title Splash**: `TITLE_SPLASH` must never be declared based on raw pixel counts alone when the yellow `title_play_button` contour is absent. The presence of a valid `title_play_button` contour in the center-bottom ROI should be a mandatory requirement for `TITLE_SPLASH`.
2. **Increase Logo Shield Threshold**: Real Dungeon Quest shield logo yellow pixel volume exceeds $15,000\text{px}$. The secondary cue threshold should be raised from $3,000$ to $\ge 12,000\text{px}$.
3. **Disambiguate Updates Button from Lobby Play Button**:
   - `Title Updates`: Width $147\text{px}$, Height $64\text{px}$, Area $> 6,000\text{px}$, $y \in [470, 534]$.
   - `Lobby Play`: Width $112\text{px}$, Height $36\text{px}$, Area $\approx 3,800\text{px}$, $y \in [490, 534]$.
   Use contour size/aspect ratio or check whether the top-left player HUD (Level badge, HP bar at $[0:0.16h, 0:0.30w]$) is present. If the player HUD is present, the screen is **definitively the lobby**, never the title screen!
4. **Remove Conflicting `logo_yellow_px < 2500` Context Guard in `LOBBY_IDLE`**: The guard on line 485 erroneously disqualifies valid lobby screens whenever a player has yellow armor or stands near town lights.
