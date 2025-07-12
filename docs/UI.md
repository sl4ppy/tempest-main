# Tempest: UI and Screen Documentation

This document provides a reconstruction-grade specification for every screen, menu, overlay, and user interface state in Atari's *Tempest*. The details herein are reverse-engineered from the original 6502 assembly code and are intended to be exhaustive, allowing for a perfect recreation of the game's visual and interactive elements.

---

## 1. Title Screen & Attract Mode

This is not a single screen, but a continuous, looping sequence of states designed to attract players, display copyright and pricing information, show the high score table, and run a demonstration of the gameplay. For a detailed breakdown of the sequence flow, see the [Attract Mode documentation](./ATTRACT_MODE.md).

### 1.1. Screen Identification and Purpose

-   **Name:** Title Screen / Attract Mode
-   **Purpose:**
    1.  Attract potential players in an arcade setting.
    2.  Display legal and branding information (Atari Logo, Copyright Year).
    3.  Inform the player of the cost to play and current credits.
    4.  Display the high score table to encourage competition.
    5.  Demonstrate gameplay through a non-interactive "demo" mode.
-   **Entry/Exit Conditions:**
    -   **Entry:** The [Attract Mode](./ATTRACT_MODE.md) is the default state of the game engine after the initial power-on and self-check routines are complete. It begins immediately.
    -   **Exit:** The loop is exited when a player inserts a coin and presses either the 1-Player or 2-Player Start button. This transitions the game state to [`CNEWGA` (New Game)](./GAME_STATE_FLOW.md#cnewga--new-game-initialization).

### 1.2. Layout and Structure

-   **Dimensions and Coordinate System:** The game uses the Atari [Vector Generator's](./SYSTEMS.md#vector-graphics-engine) coordinate system. All UI elements are vector-based drawings, positioned freely on the screen. There is no grid layout; positions are hardcoded in drawing routines.
-   **Layer Order:** All attract mode elements are drawn on a single layer. The vector hardware does not support Z-indexing; the order of drawing commands in the display list determines which lines are drawn over others, but in practice, the elements are positioned so they do not overlap.

### 1.3. UI Elements and Assets

-   **Atari "Fuji" Logo & Copyright:**
    -   A vector-drawn representation of the Atari logo.
    -   The text "© MCMLXXX ATARI" is displayed beneath it.
    -   **Asset Identifier:** The raw [vector shape data](./DATA_ASSETS.md#vector-shape-data) is located in [`ALVROM.MAC`](./ALVROM.md). The [text string](./DATA_ASSETS.md#text-and-language-data) is defined at label `EATARI` in [`ALLANG.MAC`](./ALLANG.md).
-   **Coin & Credit Information:**
    -   Text displaying the current pricing mode (e.g., "1 COIN 1 PLAY", "FREE PLAY").
    -   Text displaying "CREDITS" followed by the current number of credits.
    -   **Asset Identifier:** [Pricing mode strings](./DATA_ASSETS.md#text-and-language-data) are at labels `ECMOD1-ECMOD3` and `ECMODE` in [`ALLANG.MAC`](./ALLANG.md). The "CREDITS" string is at `ECREDI`.
-   **"Insert Coin" / "Press Start" Prompts:**
    -   Flashing text that alternates between "INSERT COIN" (or a variation based on pricing) and a blank state.
    -   If credits are present, this changes to a static "PRESS START" prompt.
    -   **Asset Identifier:** Strings are at `EINSER` and `MPRESS` (in `ALSCOR.MAC`'s data section).
-   **High Score Table:** The full high score list, showing ranks, scores, and initials.
-   **Gameplay Demo:** The standard [in-game HUD](./UI.md#4-in-game-hud-heads-up-display) and [playfield](./PLAYFIELD.md) are displayed during the gameplay demonstration portion of the attract mode.

### 1.4. Text and Label Data

All on-screen text is stored as [vector shape data](./DATA_ASSETS.md#alphanumeric-font-data). The primary source for these strings is [`ALLANG.MAC`](./ALLANG.md).

| Text Displayed | Label in `ALLANG.MAC` |
| --- | --- |
| © MCMLXXX ATARI | `EATARI` |
| INSERT COINS | `EINSER` |
| FREE PLAY | `SCMODE` |
| 1 COIN 2 PLAYS | `ECMOD1` |
| 1 COIN 1 PLAY | `ECMOD2` |
| 2 COINS 1 PLAY | `ECMOD3` |
| CREDITS | `ECREDI` |
| 2 GAME MINIMUM | `E2GAME` |

The "GAME OVER" and "PRESS START" messages are defined within the data sections of [`ALSCOR.MAC`](./ALSCO2.md).

### 1.5. Colors and Visual Styling

-   **Color Palette:** Standard 16-color Atari vector palette. Colors are defined as constants in [`ALCOMN.MAC`](./ALCOMN.md) (e.g., `ZWHITE`, `ZRED`, `ZBLUE`).
-   **Styling:** All elements are simple vectors with no gradients, transparency, or special effects beyond color and blinking.
-   **Color Usage:**
    -   The Atari Logo and informational text are typically `ZWHITE`.
    -   Colors during the gameplay demo are handled by the main game rendering engine, with the [playfield](./PLAYFIELD.md) and [enemies](./ENTITIES.md#enemies) using their standard palettes.

### 1.6. Animation and Transitions

The Attract Mode is a [state machine-driven](./SYSTEMS.md#5-game-state-machine) sequence. The main game loop in [`ALEXEC.MAC`](./ALEXEC.md) checks the `QSTATE` variable each frame and jumps to the appropriate routine. The flow is as follows:

1.  **[`CLOGO`](./GAME_STATE_FLOW.md#clogo--attract-mode-logo-display) (Logo Init):**
    -   **Action:** The game state is set to `CLOGO`. The `LOGINI` routine in `ALEXEC.MAC` is called.
    -   This sets the display state `QDSTATE` to show the Atari logo (`CDLOGP`) and its surrounding box (`CDBOXP`).
    -   It sets a pause timer, `QTMPAUS`, and sets the next state, `QNXTSTA`, to `CDLADR`.
2.  **[`CPAUSE`](./GAME_STATE_FLOW.md#cpause--generic-pause) (Pause):**
    -   **Action:** The game waits for the `QTMPAUS` timer to expire.
    -   Once expired, it transitions to the state stored in `QNXTSTA`.
3.  **[`CDLADR`](./GAME_STATE_FLOW.md#cdladr--attract-mode-high-score-display) (Display High Score Table):**
    -   **Action:** The `DLADR` routine is called. It sets `QDSTATE` to `CDHITB` (Display High Score Table).
    -   Another pause is initiated, with the next state set to `CREQRAT`.
4.  **[`CREQRAT`](./GAME_STATE_FLOW.md#cinirat--creqrat--skill-level-select) (Request Rating - Demo Setup):**
    -   **Action:** This is the setup phase for the demo game. The `PRORAT` routine in [`ALWELG.MAC`](./ALWELG.md) is executed.
    -   Because the game is in attract mode (checked via `BIT QSTATUS`), it does not wait for player input.
    -   It automatically selects a random starting level from the first 8 levels (`LDA RANDOM; AND I,7`).
    -   It initializes the chosen level (`CONTOUR`), [enemies](./ENTITIES.md#enemies) (`INIENE`), and [player](./ENTITIES.md#entity-player-ship-blaster--cursor) (`INICUR`).
    -   It sets the game state `QSTATE` to `CNEWV2`.
5.  **[`CNEWV2`](./GAME_STATE_FLOW.md#44-state-cendwav--cnewv2-end-of-wave-and-warp) (New Wave Fly-In):**
    -   **Action:** The iconic "fly down the tube" animation plays.
    -   Upon completion, the state transitions to `CPLAY`.
6.  **[`CPLAY`](./GAME_STATE_FLOW.md#cplay--gameplay) (Gameplay Demo):**
    -   **Action:** The main `PLAY` routine in [`ALWELG.MAC`](./ALWELG.md) runs. The game's AI now controls the player ship.
    -   The game plays itself until the player ship is destroyed.
7.  **[`CENDLI`](./GAME_STATE_FLOW.md#cendli--end-of-life) -> [`CENDGA`](./GAME_STATE_FLOW.md#cendga--game-over) (End Life / End Game):**
    -   **Action:** After the demo player dies, the game transitions to the "Game Over" sequence.
    -   The `ENDGAM` routine displays the "GAME OVER" message.
    -   A final pause is initiated, after which the next state is set back to `CLOGO`, restarting the entire attract mode loop.

**Blinking Text Animation:**
-   The "INSERT COIN" text flashes. This is controlled by the `D2GAME` routine in [`ALSCOR.MAC`](./ALSCO2.md).
-   **Logic:** `LDA QFRAME; AND I,1F; CMP I,10; IFCC ...`
-   This logic uses the lower 5 bits of the global frame counter (`QFRAME`), meaning the cycle repeats every 32 frames. The text is visible for the first 16 frames (`$00-$0F`) and invisible for the next 16 frames (`$10-$1F`).

### 1.7. Input Handling and Navigation

-   **Input:** The system listens for two primary inputs during the attract mode:
    1.  **Coin Insertion:** Monitored by the [`COIN65.MAC` library](./SYSTEMS.md#coin-and-credit-logic-coin65-library). Inserting a coin increments the `$$CRDT` counter.
    2.  **Start Buttons:** The `MSTRT1` (1-Player) and `MSTRT2` (2-Player) inputs are checked every frame.
-   **Navigation Logic:** The `PROCRE` (Process Credits) routine in [`ALEXEC.MAC`](./ALEXEC.md) handles the logic. If `$$CRDT` is greater than zero and a start button is pressed, it will:
    1.  Decrement the credit count.
    2.  Set the number of players (`NUMPLA`).
    3.  Clear the `MATRACT` flag in `QSTATUS`, putting the game into active play mode.
    4.  Change the `QSTATE` to [`CNEWGA`](./GAME_STATE_FLOW.md#cnewga--new-game-initialization) to start a new game.

### 1.8. Audio and Sound Integration

-   **Coin Insertion:** A sound effect is played when a coin is registered.
-   **Gameplay Demo:** All standard gameplay sound effects (firing, explosions, enemy sounds) and the background "rumble" are active during the demo.
-   **Music:** There is no persistent background music during the logo or high score screens.

### 1.9. Dynamic/Procedural Elements

-   **Credit Display:** The number of credits is dynamically updated and displayed.
-   **Gameplay Demo:** The entire gameplay sequence is procedural. The AI's performance and the events of the demo game will be different each time the attract mode cycles.
-   **High Score Table:** The list is dynamic and reflects the actual high scores stored in memory.

### 1.10. State Management and Logic

-   **`QSTATE` (`$00`):** A 1-byte variable holding the current main state code (e.g., `CLOGO`, `CPLAY`). This acts as an index into the `ROUTAD` function pointer table in [`ALEXEC.MAC`](./ALEXEC.md).
-   **`QDSTATE` (`$01`):** A 1-byte variable holding the current display state code (e.g., `CDLOGP`, `CDHITB`). This tells the `INFO` routine in [`ALSCO2.MAC`](./ALSCO2.md) what informational text to draw.
-   **`QSTATUS` (`$05`):** A 1-byte variable holding various status flags.
    -   **`MATRACT` (bit 7, mask `$80`):** The most critical flag. If set, the game is in active play. If clear, the game is in Attract Mode. This flag alters the behavior of dozens of routines throughout the codebase.
-   **`QTMPAUS` (`$04`):** A 1-byte timer, decremented once per second. Used for timed pauses between states.
-   **`QNXTSTA` (`$02`):** A 1-byte variable that stores the `QSTATE` code to transition to after `QTMPAUS` reaches zero.

### 1.11. Example Data and Code

**State Transition Example (from [`ALEXEC.MAC`](./ALEXEC.md)):**
```assembly
LOGINI:
    LDA I,CDLOGP    ; Set display state to show the logo picture
    STA QDSTATE
    LDA I,CDLADR    ; Set the *next* state to be "Display Ladder/Scores"
    STA QNXTSTA
    LDA I,6         ; Set the timer for 6 seconds
    STA QTMPAUS
    LDA I,CPAUSE    ; Set the *current* state to "Pause"
    STA QSTATE
    LDA I,CDBOXP    ; Also request the box to be drawn around the logo
    STA QNXTDS2
    RTS
```
This snippet from the `LOGINI` routine perfectly illustrates the state management. It configures what to draw (`QDSTATE`), how long to wait (`QTMPAUS`), where to go next (`QNXTSTA`), and then enters the `CPAUSE` state to wait.

### 1.12. Integration with Game Flow

The [Attract Mode](./ATTRACT_MODE.md) is the central hub of the game's non-interactive flow. It is a self-contained loop that demonstrates the game's features and waits for player interaction. Its only exit point is a transition to the ["New Game"](./GAME_STATE_FLOW.md#cnewga--new-game-initialization) state, which hands control over to the player. Upon ["Game Over,"](./GAME_STATE_FLOW.md#cendga--game-over) the game flow returns to this loop.

---

## 2. High Score & Name Entry Screens

This section details two related but distinct UI states: the display of the high score table and the screen where a qualifying player enters their initials.

### 2.1. High Score Table Display

This screen shows the ranked list of the top 8 scores.

-   **Screen Identification and Purpose:**
    -   **Name:** High Score Table (or "Ladder")
    -   **Purpose:** To display the top scores and initials, fostering competition.
    -   **When/How Shown:**
        1.  During the [Attract Mode](./ATTRACT_MODE.md) sequence, between the Atari logo and the gameplay demo.
        2.  After a game ends and the player *does not* achieve a new high score.
    -   **Entry/Exit Conditions:**
        -   **Entry:** `QSTATE` is set to [`CDLADR`](./GAME_STATE_FLOW.md#cdladr--attract-mode-high-score-display) (Display Ladder) or `QDSTATE` is set to `CDHITB` (Display High Score Table).
        -   **Exit (Attract Mode):** A timer (`QTMPAUS`) expires, transitioning the game to the next state in the attract loop ([`CREQRAT`](./GAME_STATE_FLOW.md#cinirat--creqrat--skill-level-select)).
        -   **Exit (Post-Game):** A timer expires, transitioning back to the beginning of the attract loop ([`CLOGO`](./GAME_STATE_FLOW.md#clogo--attract-mode-logo-display)).

-   **Layout and Structure:**
    -   A single, centered column of text.
    -   The title "HIGH SCORES" is at the top.
    -   Below are 8 rows, each containing a rank number (1-8), a 6-digit score, and 3 initials.
    -   **Layout Logic:** The `LDRDSP` subroutine in [`ALSCO2.MAC`](./ALSCO2.md) is responsible for drawing this screen. It loops 8 times (defined by the `NHISCO` constant), calculating the vertical position for each new line of text.

-   **UI Elements and Data:**
    -   **Title Text:** "HIGH SCORES". [String data](./DATA_ASSETS.md#text-and-language-data) is referenced by the `MHIGHS` label.
    -   **Score Table Data:** The scores and initials are read directly from the high score data tables in RAM, starting at `HSCORL` and `INITAL`.
    -   **Fonts:** Standard game font, rendered by the `JSRL CHAR.` vector graphics routine. See [Font Data](./DATA_ASSETS.md#alphanumeric-font-data).
    -   **Colors:** All text is typically drawn in a single color, usually `ZWHITE` or `ZGREEN`.

### 2.2. "Enter Your Initials" Screen

This screen appears when a player's score is high enough to be added to the table.

-   **Screen Identification and Purpose:**
    -   **Name:** Get Initials
    -   **Purpose:** To allow a player to enter three initials for their high score entry.
    -   **When/How Shown:** Shown immediately after a game ends if the [`HISCHK`](./GAME_STATE_FLOW.md#chischk--high-score-check) routine determines the player's score qualifies.
    -   **Entry/Exit Conditions:**
        -   **Entry:** The `HISCHK` routine successfully finds a place for the new score and sets `QSTATE` to [`CGETINI`](./GAME_STATE_FLOW.md#cgetini--get-player-initials) (Get Initials).
        -   **Exit:**
            1.  **Success:** The player presses the Fire button three times, locking in their initials. The game then transitions to display the updated high score table ([`CDLADR`](./GAME_STATE_FLOW.md#cdladr--attract-mode-high-score-display)).
            2.  **Timeout:** The player fails to enter their initials within the time limit (`ITIMHI` constant, 60 seconds). The entry is aborted, the new score is saved with default initials, and the game proceeds to the high score display.

-   **Layout and Structure:**
    -   The layout is managed by the `GETDSP` routine in [`ALSCO2.MAC`](./ALSCO2.md).
    -   **Top Text:** "PLAYER X" (where X is 1 or 2).
    -   **Main Title:** "ENTER YOUR INITIALS"
    -   **Instructional Text:** "SPIN KNOB TO CHANGE", "PRESS FIRE TO SELECT"
    -   **High Score Table:** The full table is displayed below the text, with the new score entry highlighted or flashing.
    -   **Initials:** The three initials for the current entry are displayed. The currently-being-edited initial is animated (see below).

-   **Animation and Interaction:**
    -   **Initial Selection:** The player uses the rotary controller to change the currently selected letter. The `GINICO` subroutine in `ALSCO2.MAC` processes the dial input (`TBHD`) and cycles through the alphabet (A-Z and a blank space).
    -   **Flashing Initial:** The character currently being edited flashes or cycles colors to indicate it is active.
    -   **Confirmation:** Pressing the Fire button (`MFIRE` or `MSUZA`) confirms the current letter and moves the cursor to the next initial.
    -   **Timer:** A countdown timer (`TIMHIS`) runs in the background. It is not displayed on screen. If it reaches zero, the process is aborted.

-   **Input Handling:**
    -   **Rotary Knob:** Used to select a letter.
    -   **Fire/Superzapper Buttons:** Used to confirm a letter.

-   **State Management and Logic:**
    -   The entire process is managed by the [`GETINI` state](./GAME_STATE_FLOW.md#cgetini--get-player-initials).
    -   `TBLIND`: A pointer to the current initial (0, 1, or 2) being edited in the `INITAL` table.
    -   `TIMHIS`: The countdown timer for the screen.
    -   When all three initials are entered, the `WRHIIN` routine is called to permanently write the new score and initials to the EAROM (non-volatile memory).

-   **Integration with Game Flow:**
    -   This screen is the bridge between the end of a successful game and the updated high score table. It is a temporary, interactive state that feeds its result (the player's initials) back into the permanent game data before returning to the standard, non-interactive display states.

---

## 3. Skill Level Select Screen

This screen allows the player to choose their starting level, a key feature of *Tempest*.

### 3.1. Screen Identification and Purpose

-   **Name:** Skill Level Select / "Rate Yourself"
-   **Purpose:** To allow players to select a starting level appropriate to their skill, bypassing the initial, easier waves.
-   **When/How Shown:** This screen is shown at the start of a new game for the first player, and at the start of the second player's turn in a 2-player game.
-   **Entry/Exit Conditions:**
    -   **Entry:** After a player presses Start and a new game begins, the state is set to [`CINIRAT`](./GAME_STATE_FLOW.md#cinirat--creqrat--skill-level-select) (Initialize Rating).
    -   **Exit:**
        1.  **Player Selection:** The player highlights a level with the knob and presses the Fire button. The game state changes to `CNEWLIF` to start the selected level.
        2.  **Timeout:** If the player does not make a selection within the 10-second timer, the game automatically selects the first available level (Level 1) and proceeds.

### 3.2. Layout and Structure

-   The screen is drawn by the `RQRDSP` (Request Rate Display) routine in [`ALSCO2.MAC`](./ALSCO2.md).
-   **Title:** "RATE YOURSELF"
-   **Instructional Text:** "SPIN KNOB TO CHANGE", "PRESS FIRE TO SELECT"
-   **Level Display:** The screen shows two columns of text representing the selectable levels.
    -   The left column lists levels from 1 to 48.
    -   The right column lists levels from 49 to 96.
    -   Not all levels are selectable initially.
-   **Selector:** A flashing or highlighted cursor indicates the currently selected level.
-   **Timer:** A numeric countdown timer from 10 is displayed.

### 3.3. UI Elements and Assets

-   **Text Labels:** See [Text Data](./DATA_ASSETS.md#text-and-language-data).
    -   "RATE YOURSELF": `MRATE`
    -   "SPIN KNOB TO CHANGE": `MPRMOV`
    -   "PRESS FIRE TO SELECT": `MPRFIR`
    -   "NOVICE": `MNOVIC`
    -   "EXPERT": `MEXPER`
-   **Level Numbers:** The numbers are drawn procedurally using the standard character-drawing routines. The available levels are stored in the `LEVEL` table in [`ALWELG.MAC`](./ALWELG.md).
-   **Selector Cursor:** The flashing effect of the cursor is achieved by rapidly changing its color.

### 3.4. Animation and Transitions

-   **Cursor Movement:** As the player turns the rotary knob, the highlighted selection moves up or down the list of levels. The `GETCUR` routine updates the cursor's position (`CURSL1`).
-   **Timer Countdown:** The displayed timer decrements every second. This is driven by the `TIMHIS` counter in the `PRORAT` state.
-   **3-Second Warning:** When the timer reaches 3, the `S3SWAR` sound routine is called to play an audible warning.

### 3.5. Input Handling and Navigation

-   **Rotary Knob:** The primary input, read via `TBHD`. It moves the `CURSL1` cursor variable to select a level from the `LEVEL` table.
-   **Fire/Superzapper Buttons:** A press of `MFIRE` or `MSUZA` confirms the selection.

### 3.6. Dynamic/Procedural Elements

-   **Selectable Level Range:** The range of levels a player can choose from is not fixed. It is determined by the `INIRA0` routine based on two factors:
    1.  **Highest Wave Achieved:** The maximum level selectable is tied to the highest wave the player reached in their *previous* game (`HIWAVE` variable). A player cannot start on a level they have not previously reached.
    2.  **Operator Settings:** A DIP switch setting can tie the maximum selectable level to the player's all-time high score, allowing very skilled players to start on higher levels without having to "unlock" them in the current session.

### 3.7. State Management and Logic

-   **State:** The main state is [`CREQRAT`](./GAME_STATE_FLOW.md#cinirat--creqrat--skill-level-select) (Request Rate), which executes the `PRORAT` subroutine in [`ALWELG.MAC`](./ALWELG.md). The display state is `CDREQRA`.
-   **`LEVEL` table:** An array of bytes in `ALWELG.MAC` that maps a selection index to an actual wave number. This allows the level list to have gaps (e.g., jumping from level 14 to 17).
-   **`LEFSID`/`RITSID`:** Variables that hold the index of the level currently selected.
-   **`HIRATE`:** The maximum index in the `LEVEL` table that the player is allowed to select.
-   **`QTMPAUS`:** Used as the 10-second countdown timer for this screen.
-   **Timeout Logic:** `IFMI; LDA I,MFIRE; STA SWFINA; ENDIF` in `PRORAT`. If the timer (`QTMPAUS`) reaches zero, the code simulates a fire button press, automatically selecting the currently highlighted level (which defaults to the first one).

### 3.8. Integration with Game Flow

This screen is the final step before gameplay begins. It acts as a gateway, taking the `NUMPLA` (number of players) and `PLAYUP` (current player) variables as input and producing the starting `CURWAV` (current wave) as output. Once a level is selected, it loads all the necessary game assets and transitions directly into the main gameplay state (`CPLAY`) via `CNEWLIF`.

---

## 4. In-Game HUD (Heads-Up Display)

The in-game HUD provides the player with critical status information during gameplay. It is not a separate screen but an overlay rendered on top of the playfield.

### 4.1. Screen Identification and Purpose

-   **Name:** In-Game HUD / Status Display
-   **Purpose:** To display the current score, remaining lives, and high score for one or two players.
-   **When/How Shown:** It is continuously displayed during active gameplay (`QSTATE` = [`CPLAY`](./GAME_STATE_FLOW.md#cplay--gameplay)). It is hidden during the inter-level "warp" animation.
-   **Entry/Exit Conditions:** It appears when a level begins and disappears when the game ends or transitions to a different screen (like ["Game Over"](./UI.md#51-game-over-screen)).

### 4.2. Layout and Structure

-   The HUD is laid out in a fixed, two-player arrangement, even during a one-player game.
-   **Player 1 (Left Side):**
    -   Score is displayed in the upper-left quadrant.
    -   Life icons are displayed directly below the score.
-   **Player 2 (Right Side):**
    -   Score is in the upper-right quadrant.
    -   Life icons are below the score.
-   **High Score (Center):**
    -   The current high score is displayed at the top-center of the screen, between the two player scores.
-   **Layout Logic:** The layout is defined by a master template, `SCORES`, in [`ALVROM.MAC`](./ALVROM.md). At the start of a game, `INITEM` copies this template to a RAM buffer (`SCOBUF`). The `UPSCLI` routine then populates this buffer with the current game data. The X/Y positions are hardcoded into the template using `VCTR` commands.

### 4.3. UI Elements and Assets

-   **Player Score:** A 6-digit BCD number. Drawn using the standard character set.
-   **Life Icons:** A vector drawing of the player's ship. Up to 6 icons are displayed per player.
    -   **Asset Identifier:** The life icon shape is `LIFE0` in [`ALVROM.MAC`](./ALVROM.md). The routine displays one icon for each life remaining in the `LIVES1`/`LIVES2` variables.
-   **High Score:** The overall high score is displayed, fetched from the `HSCORL`... tables.
-   **Superzapper Status:** There is no persistent on-screen icon for the [Superzapper](./ENTITIES.md#entity-superzapper-weapon). Its status is communicated through on-screen messages:
    -   "SUPERZAPPER RECHARGED": Displayed at the start of a level.
    -   The "BOOM" that clears the screen when it's used.

### 4.4. Text and Label Data

-   No text labels are part of the core HUD, only numeric scores and icons.
-   Post-wave messages like "SUPERZAPPER RECHARGED" (`MSUPZA`) or "AVOID SPIKES" (`MSPIKE`) are handled by the general message display system (`MSGS`).

### 4.5. Colors and Visual Styling

-   **Player 1 Score/Lives:** Typically `ZGREEN`.
-   **Player 2 Score/Lives:** Also typically `ZGREEN`.
-   **Life Icons:** Typically `ZYELLO`.
-   **Bonus Life Flash:** When a bonus life is awarded, the `BOFLASH` timer is set, causing the lives display to flash for a short duration.

### 4.6. State Management and Logic

-   **Rendering Engine:** The `INFO` subroutine in [`ALSCO2.MAC`](./ALSCO2.md) is the master rendering routine for the HUD. It is called every frame when the game is in the `CDPLAY` display state.
-   **`UPSCLI` (Update Score & Lives):** This is the key helper function called by `INFO`.
    -   **Input:** Player ID (0 or 1) in the Y-register.
    -   **Action:**
        1.  Reads the score for the specified player from `LSCORL`/`RSCORL` tables.
        2.  Reads the number of lives from `LIVES1`/`LIVES2`.
        3.  Converts the BCD score to vector digit drawing commands using `NWDIGS`.
        4.  Draws the appropriate number of life icons. It draws one fewer icon than the player has lives, as the ship currently in play counts as one life.
        5.  Writes all these commands directly into the correct locations in the `SCOBUF` display buffer, overwriting the template data for that frame.
-   **Data Sources:**
    -   **Scores:** `LSCORL`/`RSCORL` through `LSCORH`/`RSCORH`.
    -   **Lives:** `LIVES1` and `LIVES2`.
    -   **High Score:** `HSCORL`, `HSCORM`, `HSCORH`.
-   In a one-player game, the Player 2 side of the HUD is simply not updated and remains at zero.

---

## 5. Game State Transitions and Effects

This section details the visual screens and animations that occur between primary game states, such as the end of a game or the transition between levels.

### 5.1. "Game Over" Screen

This is a simple, non-interactive screen displayed at the conclusion of a game when all players have exhausted their lives.

-   **Screen Identification and Purpose:**
    -   **Name:** Game Over
    -   **Purpose:** To formally signal the end of the current game session.
    -   **When/How Shown:** Triggered after the last player loses their last life. The `ENDLIF` routine in `ALEXEC.MAC` checks if `LIVES1` and `LIVES2` are both zero, and if so, calls the `ENDGAM` state routine.
    -   **Entry/Exit Conditions:**
        -   **Entry:** `QSTATE` is set to [`CENDGA`](./GAME_STATE_FLOW.md#cendga--game-over). The display state `QDSTATE` is set to `CDGOVR`.
        -   **Exit:** After a brief, timed pause, the game flow automatically proceeds to the high score check ([`CHISCHK`](./GAME_STATE_FLOW.md#chischk--high-score-check)). There is no player interaction.

-   **Layout and UI Elements:**
    -   **Text:** A large "GAME OVER" message is displayed in the center of the screen.
    -   **Asset Identifier:** The string is referenced by the `MGAMOV` label. The drawing logic is in the `DGOVER` subroutine in [`ALSCO2.MAC`](./ALSCO2.md).
    -   The rest of the screen is typically black, with the [playfield](./PLAYFIELD.md) and HUD no longer being drawn.

### 5.2. Inter-Level Warp ("Tube Fly-In")

This is the iconic 3D animation that transitions the player from a completed wave to the next one.

-   **Screen Identification and Purpose:**
    -   **Name:** New Wave Warp / Tube Fly-in
    -   **Purpose:** To provide a visually exciting transition between levels and to load the next wave's geometry and enemies.
    -   **When/How Shown:** Occurs immediately after a wave is cleared. The `ENDWAV` routine sets `QSTATE` to [`CNEWV2`](./GAME_STATE_FLOW.md#cendwav--cnewv2--end-of-wave--warp-transition).
    -   **Entry/Exit Conditions:**
        -   **Entry:** Game state is `CNEWV2`, which executes the `NEWAV2` routine in [`ALWELG.MAC`](./ALWELG.md).
        -   **Exit:** The animation is programmatic and time-based. Once the animation's target values are reached, `NEWAV2` automatically changes the `QSTATE` to [`CPLAY`](./GAME_STATE_FLOW.md#cplay--gameplay) to begin the next level.

-   **Animation and Procedural Logic:**
    -   The effect is entirely procedural, driven by the `NEWAV2` routine. It is not a pre-recorded animation.
    -   **Camera Movement:** The routine rapidly interpolates the 3D camera's eyepoint (`EYL`, `EYH`) from the front of the completed well to the back of the next well.
    -   **Well Morphing:** Simultaneously, it interpolates the Z-axis vanishing point (`ZADJL`) of the well itself.
    -   **Combined Effect:** The combination of the camera moving forward and the well's perspective point changing creates a smooth, seamless "swooping" animation down a twisting tube into the next playfield.
    -   **Starfield:** During the warp, the background starfield is disabled by setting `PLAGRO` to 1 to enhance the sense of speed and focus on the tube.
    -   **HUD:** The In-Game HUD is not drawn during the warp. Instead, messages like "SUPERZAPPER RECHARGED" and the end-of-wave bonus points are displayed.

-   **State Management:**
    -   The `NEWAV2` routine is a state that persists across many frames. On each frame, it slightly adjusts the camera and well-vertex values before yielding control. It continues until the `EYL` and `EYH` registers reach their destination values (`EYLDES`).

---

## 6. Operator Test & Diagnostics Screens

This section covers the operator-facing screens for hardware testing, bookkeeping, and settings configuration. These screens are not accessible to the player during normal gameplay.

### 6.1. Screen Identification and Purpose

-   **Name:** Test Mode / System Diagnostics / Bookkeeping
-   **Purpose:** To provide a suite of tools for the arcade operator to test the hardware, view usage statistics, and configure the game's difficulty and pricing.
-   **When/How Shown:** The mode is entered by activating a physical "Test" switch inside the arcade cabinet. This sets the `MTEST` flag, which is detected by the `NONSTA` routine in `ALEXEC.MAC`, forcing the game state to `CSYSTM`.
-   **Entry/Exit Conditions:**
    -   **Entry:** `MTEST` switch is active.
    -   **Exit:** `MTEST` switch is deactivated. The game returns to the standard [Attract Mode](./ATTRACT_MODE.md) loop by setting the state to `CNEWGA`.

### 6.2. Layout and Structure

-   The diagnostic screen is a dense, multi-section display of text and data, all rendered by the `DSPSYS` routine in [`ALTES2.MAC`](./ALTES2.md). There is no single, consistent layout; it is a collection of different data readouts positioned in fixed locations.
-   **Sections Include:**
    -   DIP Switch Settings (Binary and Text)
    -   Bookkeeping Statistics (Total Plays, Coins, etc.)
    -   Game Configuration (Lives, Bonus, Difficulty)
    -   Interactive Test Menu

### 6.3. UI Elements and Assets

-   **Text:** All information is displayed as vector text. This includes labels like "ROM TEST", "COINAGE", "LIVES PER GAME", and the values associated with them.
-   **Interactive Cursor:** A simple flashing or color-cycling vector shape is used to indicate the currently selected option in the interactive menu portion of the screen.

### 6.4. Text and Label Data

The screen is composed almost entirely of textual data. Key labels include:
-   `BOOKEEPING TOTALS`
-   `TOTAL PLAYS`
-   `TOTAL COINS`
-   `MINUTES PLAYED`
-   `LIVES PER GAME`
-   `BONUS LIFE AT`
-   `GAME DIFFICULTY` (EASY, MEDIUM, HARD)
-   `HIGH SCORE TABLE ERASED` (Confirmation message)

### 6.5. Operator-Interactive Tests

The test mode provides an interactive menu that the operator navigates with the game's rotary knob and selects with the Fire button. This allows access to several hardware sub-tests.

-   **ROM Test:** Performs a checksum on all program ROMs and displays "OK" or "BAD" for each.
-   **RAM Test:** Writes and reads patterns to all RAM addresses and reports any failures.
-   **Crosshatch Pattern:** Fills the screen with a stable crosshatch pattern for monitor alignment and geometry checks.
-   **Color Bars:** Cycles the screen through solid fields of the 16 available colors to test the monitor's color output.
-   **Sound Test:** Plays various sound effects.
-   **Erase High Scores:** A selectable option that, when confirmed, calls the `EAZERO` routine to wipe all high scores and bookkeeping data from the EAROM.

### 6.6. State Management and Logic

-   **Main State:** The test mode runs under the [`CSYSTM` game state](./GAME_STATE_FLOW.md#csystm--system-test-mode), executing the `SYSTEM` subroutine in [`ALTES2.MAC`](./ALTES2.md).
-   **Display State:** The screen is rendered when the `QDSTATE` is set to `CDSYST`.
-   **Input Handling:** The `SYSTEM` routine polls the rotary knob (`TBHD`) and fire button (`MFIRE`) to navigate its internal menu state.
-   **DIP Switches:** The state of the physical DIP switches is read directly from the hardware I/O ports (`INOP0`, `INOP1`, `INOP2`) and the corresponding text (e.g., "3 LIVES", "HARD") is displayed.
-   **Bookkeeping Data:** Statistics like total coins and playtime are read from the EAROM via the `EAUPD` driver and displayed as BCD numbers.

### 6.7. Integration with Game Flow

The Test Mode is a completely separate branch of the game's logic. It bypasses the entire player-facing game flow (Attract Mode, Gameplay, etc.). It has a single entry point (the physical test switch) and a single exit point (deactivating the switch), which cleanly resets the game and returns it to the [Attract Mode](./ATTRACT_MODE.md).