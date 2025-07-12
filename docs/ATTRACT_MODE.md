# Tempest: Attract Mode

This document provides a reconstruction-grade, exhaustive breakdown of the "Attract Mode" in Tempest. The Attract Mode is the sequence of screens and gameplay demonstrations shown when the machine is idle, designed to attract players and showcase the game.

## 1. High-Level Control Flow & State Machine

The entire game, including the attract mode, is controlled by a state machine in `ALEXEC.MAC`. The core of this is the `ROUTAD` dispatch table, which executes a specific subroutine based on the value in the `QSTATE` variable.

The system is considered to be in "Attract Mode" when the `MATRACT` bit in the `QSTATUS` flag is set. This typically occurs on boot or after a game ends and no new credits are inserted.

### 1.1. Entry Conditions

Entry into the attract mode is triggered by one of the following conditions:

*   **Power-On / Boot-Up:** On a cold start, the game initializes sounds (`INISOU`) and immediately sets `QSTATE` to [`CNEWGA`](GAME_STATE_FLOW.md#41-state-cnewga-new-game) (New Game). However, with no players (`NUMPLA = 0`), the logic immediately flows into the attract sequence.
*   **Game Over:** After a game ends (state [`ENDGAM`](GAME_STATE_FLOW.md#46-state-cendga-chischk--cgetini-game-over--high-scores)), the logic proceeds to the high-score check (`HISCHK`). After the high-score sequence is complete (or skipped), the system transitions to the [`DLADR`](#2-phase-detail-high-score-display-ladder) (Display Ladder/High Scores) state, which is the primary entry point into the main attract loop.
*   **Demo End:** After a gameplay demonstration sequence concludes, it flows back into the main attract loop (typically the high score display).

### 1.2. Exit Conditions

The attract mode is interrupted and exits under the following conditions:

*   **Coin Insert:** Inserting a coin increments the credit counter (`$$CRDT`). The `PROCRE` routine detects this and, if the `START` button is pressed, transitions the game to the [`CNEWGA`](GAME_STATE_FLOW.md#41-state-cnewga-new-game) (New Game) state.
*   **Start Button Press (with credits):** If credits are present, pressing the 1-Player or 2-Player start buttons will exit the attract mode and begin a new game. The `MSTRT1` and `MSTRT2` flags are checked.
*   **Test Mode:** Toggling the physical `TEST` switch on the cabinet hardware forces the system into a diagnostic state ([`CSYSTM`](GAME_STATE_FLOW.md#49-state-csystm-system-test-mode)), immediately halting the attract mode. This is checked by `NONSTA` routine.

### 1.3. Global State Flags

| Flag | Location | Description |
|---|---|---|
| `QSTATE` | `ALCOMN.MAC` | The primary variable controlling the current game state. It's an index into the `ROUTAD` jump table. |
| `QSTATUS` | `ALCOMN.MAC` | A bitfield of global status flags. The `MATRACT` bit indicates if the game is in the attract loop. The `MGTMOD` bit indicates if a game is in progress. |
| `$$CRDT` | `ALCOMN.MAC` | Holds the current number of credits. A value of `0` is a precondition for the attract mode to run. |
| `NUMPLA` | `ALCOMN.MAC` | Number of players in the current game. A value of `0` indicates no game is active. |

## 2. Sequence Overview

The attract mode is a continuous, repeating loop of several distinct phases. The order is hard-coded via state transitions in `ALEXEC.MAC`.

The primary sequence is as follows:

1.  **High Score Display:** The "High Score Ladder" is shown.
    *   **State:** [`DLADR`](#21-high-score-display-cdladr) (`Display Ladder`)
    *   **Logic:** `GETINI.MAC`, `ALSCO2.MAC`
2.  **Logo Presentation:** The Atari logo is drawn.
    *   **State:** [`LOGOINI`](#22-logo-presentation-clogo) (`Logo Init`)
    *   **Logic:** `ALEXEC.MAC`
3.  **Gameplay Demonstration:** A short, pre-programmed gameplay demo is shown.
    *   **State:** [`INIRAT`](#23-gameplay-demonstration) (`Monster Delay/Display` - This name is likely a working title; it initializes the gameplay demo).
    *   **Logic:** `INIRAT` routine in `ALEXEC.MAC` runs the demo.
4.  **Return to High Scores:** After the demo, the loop repeats, returning to the High Score Display.

The transition between these phases is managed by setting the `QSTATE` variable to the value of the next state in the sequence. For example, at the end of the [`LOGOINI`](#22-logo-presentation-clogo) state, `QSTATE` is set to [`CINIRAT`](GAME_STATE_FLOW.md#42-state-cinirat--creqrat-skill-selection) to begin the gameplay demo.

## 3. Phase Detail: High Score Display ("Ladder")

This phase displays the high score table, referred to in the source as the "Ladder."

*   **Game State:** [`CDLADR`](GAME_STATE_FLOW.md#2-complete-list-of-game-states) (`Display Ladder`)
*   **Display State:** `CDLADR` (`Display Ladder`)
*   **Core Logic:** `LDRDSP` in `ALDIS2.MAC`, `INFO` and `NWDIGS` in `ALSCO2.MAC`

### 3.1. Timing and Transitions

*   **Duration:** The High Score Display screen is shown for **60 frames** (approximately 1 second, given the 60Hz refresh rate). This is controlled by the `ITIMHI` constant in `ALSCO2.MAC`.
*   **Entry:** The game transitions to this state from the "Game Over" sequence or from the end of the gameplay demo.
*   **Exit:** After the 60-frame timer expires, the `QSTATE` is set to `CLOGOIN` to transition to the ["Logo Presentation"](#22-logo-presentation-clogo) phase.

### 3.2. Visual Layout and Content

The screen is composed of several text elements drawn by the [Vector Graphics Engine](./SYSTEMS.md#1-vector-graphics-engine). The positions are hard-coded as Y-offsets in `ALLANG.MAC`. Colors and scaling are also defined there.

| Text | Label | Content | Y-Position | Color | Scale |
|---|---|---|---|---|---|
| Title | [`MHIGHS`](DATA_ASSETS.md#text-and-language-data) | "HIGH SCORES" | `+38` | Red | 0 (Large) |
| Copyright | [`MATARI`](DATA_ASSETS.md#text-and-language-data) | "© MCMLXXX ATARI" | `+92` | Blue-Violet | 1 (Normal) |
| Credits | [`MCREDI`](DATA_ASSETS.md#text-and-language-data) | "CREDITS" | `+80` | Green | 1 (Normal) |

### 3.3. High Score Table Generation

The table itself is dynamically generated by the `LDRDSP` routine.

1.  **Header:** The "HIGH SCORES" title is drawn using the `MSGS` subroutine, referencing the `MHIGHS` message data.
2.  **Score and Initial Data:** The high score data is stored in a table starting at `HSCORL`. This table holds `NHISCO` (10) entries. Each entry consists of the score (3 bytes of BCD) and the player's initials (3 bytes).
3.  **Drawing Loop:** The code iterates `NHISCO` times. In each iteration:
    *   It retrieves the rank (1-10), the 3 initials, and the 6-digit score.
    *   The `NWDIGS` and `NWHEXZ` routines in `ALSCO2.MAC` are called to convert the BCD score digits into a sequence of vector drawing commands for the display list.
    *   The initials are likewise converted from their character codes into vector commands.
4.  **Formatting:** The table is formatted into columns for Rank, Score, and Initials. The exact X-positions are hardcoded in the drawing routines. The Y-position is decremented for each row.

### 3.4. Vector Graphics Data

The final output sent to the [Vector Graphics Engine](./SYSTEMS.md#1-vector-graphics-engine) is a display list. This list does not contain high-level text but is a stream of low-level vector commands.

*   **Text Rendering:** Text like "HIGH SCORES" is not a single vector object. The `ASCVG` macro in `ALLANG.MAC` converts each character of a string into a `JSRL` (Jump to Subroutine, Long) command in the vector display list. Each `JSRL` points to the vector data for a specific character shape stored in the Vector ROMs (`ALVROM.MAC`). See the [Alphanumeric Font Data](DATA_ASSETS.md#alphanumeric-font-data) for details.
*   **Score Rendering:** The numeric scores are rendered digit by digit using the same `JSRL` mechanism, with the `NWDIGS` function selecting the correct vector shape for each digit `0` through `9`.

An illustrative snippet of the vector list for a single high score row would look conceptually like this (actual data is binary):

```
; Conceptional Vector Display List for one row: "1. 123450 ABC"
JSRL RANK_1_SHAPE      ; Draw the rank "1."
VCTR (X, Y, BRIGHT)    ; Move beam to score position
JSRL DIGIT_1_SHAPE     ; Draw score digit "1"
JSRL DIGIT_2_SHAPE     ; Draw score digit "2"
... (etc. for all 6 score digits) ...
VCTR (X, Y, BRIGHT)    ; Move beam to initials position
JSRL CHAR_A_SHAPE      ; Draw initial "A"
JSRL CHAR_B_SHAPE      ; Draw initial "B"
JSRL CHAR_C_SHAPE      ; Draw initial "C"
```

### 3.5. Audio

During the High Score Display phase of the attract mode, **no background music or sound effects are played.** The [Sound Engine](./SYSTEMS.md#8-sound-engine-alsounmac) is silent.

## 4. Phase Detail: Logo Presentation

This phase consists of a two-part animated sequence where a 3D box spins into view, stops, and then the Atari logo appears inside it.

*   **Game State:** `CLOGOIN` (`Logo Init`)
*   **Display States:** `CDBOX` (`Display Box`), `CDLOGO` (`Display Logo`)
*   **Core Logic:** `LOGINI` in `ALSCO2.MAC`, `BOXPRO` and `LOGPRO` in `ALDIS2.MAC`

### 4.1. Timing and Transitions

The logo sequence is controlled by a series of timed pauses defined in `LOGINI`.

1.  **Entry:** Enters from the High Score screen. `QSTATE` is set to `CLOGOIN`.
2.  **Box Animation:**
    *   The `LOGINI` routine sets the display state `QDSTATE` to `CDBOX`.
    *   This triggers the `BOXPRO` display routine, which draws a spinning 3D wireframe box. The box spins for **128 frames** (`TIMBOX` = 128).
    *   The animation is a simple, procedural rotation. The `ROLOGO` variable holds the current rotation angle, and it is incremented by `ROLDEL` each frame.
3.  **Logo Display:**
    *   After the box animation timer expires, `LOGINI` sets `QDSTATE` to `CDLOGO`.
    *   This triggers the `LOGPRO` display routine, which draws the static Atari logo inside the now-stationary box.
    *   The logo is displayed for **192 frames** (`TIMLOG` = 192).
4.  **Exit:** After the logo display timer expires, `LOGINI` sets the game state `QSTATE` to [`CINIRAT`](GAME_STATE_FLOW.md#42-state-cinirat--creqrat-skill-selection) to begin the Gameplay Demonstration phase.

| Parameter | Label | Value | Description |
|---|---|---|---|
| Box Anim Time | `TIMBOX` | 128 | Duration of the spinning box animation, in frames. |
| Logo Hold Time| `TIMLOG` | 192 | Duration the static logo is shown, in frames. |
| Rotation Angle| `ROLOGO` | - | Current angle of the spinning box. |
| Rotation Delta| `ROLDEL` | - | Amount the angle is incremented each frame. |

### 4.2. Vector Graphics Data and Animation

The animation is generated by two distinct display list routines in `ALDIS2.MAC`.

#### `BOXPRO`: The Spinning Box

*   This routine is responsible for the 3D animation.
*   It takes a simple 8-point cube definition (`LOGBOX`) and applies a 3D rotation matrix to it based on the current value of `ROLOGO`.
*   The `WRMATR` and `WORSCR` routines are used to perform the world-to-screen transformation, calculating the 2D screen coordinates for each of the cube's 12 edges for the current frame.
*   The output is a vector display list that draws the 12 lines of the transformed cube. The color is white (`WHITE`).

```
; Conceptual Logic of BOXPRO
FOR frame = 1 TO 128:
  ROLOGO = ROLOGO + ROLDEL
  Matrix = CreateRotationMatrix(ROLOGO)
  ScreenCoords = Transform(LOGBOX, Matrix)
  DisplayList = DrawLines(ScreenCoords)
  Draw(DisplayList)
```

#### `LOGPRO`: The Atari Logo

*   This routine draws the final, static image.
*   It first calls `BOXPRO` one last time to draw the stationary box (with `ROLOGO` at its final value).
*   It then calls `MSGS` with the `MATARI` message identifier. This is a special case. Instead of text, the `MATARI` literal in `ALLANG.MAC` contains a `JSRL` to a hand-coded vector subroutine named `ATARI` within `ALVROM.MAC`.
*   This `ATARI` subroutine contains the exact, hard-coded vector commands to draw the famous Atari "Fuji" logo.

The data for these shapes is defined as follows:

| Shape | Label | File | Description |
|---|---|---|---|
| 3D Box | [`LOGBOX`](DATA_ASSETS.md#game-object-and-ui-shapes) | `ALVROM.MAC` | An array of 8 3D vertices defining a unit cube. |
| Atari Logo| [`ATARI`](DATA_ASSETS.md#game-object-and-ui-shapes) | `ALVROM.MAC` | A hard-coded vector subroutine containing `VCTR` commands to draw the logo. |

### 4.3. Audio

Like the High Score screen, the Logo Presentation phase is **completely silent.** No music or sound effects are played.

## 5. Phase Detail: Gameplay Demonstration

This is the final and most complex phase of the attract mode, where the system runs a short, semi-random demonstration of actual gameplay. This is not a video; it is the live game engine running with a procedural AI controlling the player's ship.

*   **Game State:** [`CINIRAT`](GAME_STATE_FLOW.md#42-state-cinirat--creqrat-skill-selection) (`Initialize Rate`) -> [`CREQRAT`](GAME_STATE_FLOW.md#42-state-cinirat--creqrat-skill-selection) (`Request Rate`) -> [`CPLAY`](GAME_STATE_FLOW.md#43-state-cplay-gameplay) (`Play`)
*   **Display State:** `CDREQRA` (`Request Rate Display`) -> `CDPLAY` (`Play`)
*   **Core Logic:** `INIRAT` and `PRORAT` in `ALWELG.MAC`, `PLAY` routine.

### 5.1. Initialization and Entry

1.  **Entry:** The game transitions from the Logo Presentation phase by setting `QSTATE` to [`CINIRAT`](GAME_STATE_FLOW.md#42-state-cinirat--creqrat-skill-selection).
2.  **`INIRAT` Routine:** This routine in `ALWELG.MAC` is the main entry point. For the attract mode (`IFMI` branch), it performs the following setup:
    *   Sets the number of lives (`LIVES1`) to 1 for the demo.
    *   Uses the `RANDOM` number generator to select a starting level between 1 and 8 (`AND I,7`). This ensures the demo is slightly different each time.
    *   Initializes all game systems for the chosen level by calling `INICOL` (colors), `CONTOUR` (well geometry), and `INIENE` (enemies).
    *   Initializes the Superzapper weapon via `INISUZ`.
    *   Sets a timer (`TIMHIS`) to `SECOND` (20 frames) before the demo starts.
    *   Transitions `QSTATE` to [`CREQRAT`](GAME_STATE_FLOW.md#42-state-cinirat--creqrat-skill-selection).

### 5.2. Procedural Player Control (AI)

The player's ["Blaster" ship](ENTITIES.md#entity-player-ship-blaster--cursor) is not controlled by a pre-recorded script of inputs. Instead, its behavior is driven by a procedural AI within the main `PLAY` routine. The logic is designed to simulate a player's actions.

*   **Targeting:** The AI identifies the nearest enemy threat.
*   **Movement:** It calculates the shortest path along the rim of the "well" to align with the target enemy and moves the `CURSPO` (Cursor Position) variable accordingly.
*   **Firing:** Once aligned with a target, the AI triggers a shot by setting the `MFIRE` bit in the `SWFINA` (Software Finished Action) flag. The firing logic is not continuous; it has cooldowns and logic to avoid wasting shots, mimicking human play.
*   **Superzapper:** The AI contains logic to use the [Superzapper](ENTITIES.md#entity-superzapper-weapon). If the number of enemies on the web exceeds a certain threshold, the AI will trigger the `MSUZA` bit in `SWFINA`.

There is no input data table; the AI makes all decisions in real-time based on the game state.

### 5.3. Gameplay Differences

The gameplay demo is almost identical to actual gameplay, with a few key distinctions:

*   **Invincibility:** The player ship is **not** invincible. If an enemy reaches the rim and collides with the ship, the demo will end.
*   **Forced Paths:** The player movement is entirely determined by the targeting AI.
*   **Random Level:** The starting level is randomized, leading to different enemy types and well shapes in each attract cycle.

### 5.4. Timing and Exit Conditions

The gameplay demo ends when one of the following occurs:

*   **Player Death:** The most common exit condition. When the player's ship is destroyed, the `LIVES1` counter decrements to 0. This triggers the [`ENDLIF`](GAME_STATE_FLOW.md#45-state-cendli-cnewli--cnwlf2-player-death--respawn) (End Life) and subsequently the [`ENDGAM`](GAME_STATE_FLOW.md#46-state-cendga-chischk--cgetini-game-over--high-scores) (End Game) state.
*   **Level Clear:** If the AI successfully clears the level, the [`ENDWAV`](GAME_STATE_FLOW.md#44-state-cendwav--cnewv2-end-of-wave-and-warp) (End Wave) state is triggered.

In either case, the `ENDGAM` state is eventually reached. This state sets the `QSTATE` to `CDLADR` ("Display Ladder"), causing the entire attract mode sequence to loop back to the High Score Display.

### 5.5. Audio

Unlike the previous phases, the Gameplay Demonstration features the **full, dynamic audio of normal gameplay.**

*   **Background Music:** The appropriate "thump-thump" heartbeat music for the chosen level plays.
*   **Sound Effects:** All sound effects are active and triggered by in-game events exactly as they would be for a human player. This includes:
    *   Player firing sounds (`SLAUNC`).
    *   Enemy explosions (`SBOING`).
    *   Superzapper activation (`SOUTS3`).
    *   Player death explosion.
    *   All other enemy-specific sounds.
The sound events are triggered directly from the game logic routines (e.g., `PLAY`, `PRBOOM`). 