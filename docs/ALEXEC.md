# Tempest: ALEXEC.MAC - Main Game Executive

**File:** `ALEXEC.MAC`
**Purpose:** This is the central "executive" module of the Tempest program. It contains the main game loop, the master state machine, and the top-level logic that orchestrates all other game modules. It is the entry point of the game and controls the overall flow from attract mode to gameplay to game over.

---

## 1. Main Control Flow (`MAINLN`)

The program execution begins at the `MAINLN` label. This section initializes the sound hardware and then enters an infinite loop that constitutes the main game loop.

The loop is simple and consistent, running one iteration per video frame.
1.  **Wait for Frame:** It waits for a vertical blank interrupt by checking `FRTIMR`.
2.  **Execute State Logic:** It calls `EXSTAT`, which acts as a dispatcher to run the code for the current game state (e.g., `PLAY`, `ENDWAV`).
3.  **Execute Non-State Logic:** It calls `NONSTA` to process global events that must be checked every frame, such as coin inputs or the test switch.
4.  **Execute Display Logic:** It calls `DISPLA` (located in `ALDIS2.MAC`) to construct and render the vector display list for the current frame.
5.  **Repeat:** The loop repeats indefinitely.

```assembly
MAINLN: JSR INISOU      ; Initialize Sounds
        LDA I,CNEWGA
        STA QSTATE      ; Set initial state to New Game
        BEGIN           ; MAINLOOP
            ; ... wait for frame timer ...
            JSR EXSTAT
            JSR NONSTA
            JSR DISPLA
            CLC
        CSEND           ; Loop Always
```

## 2. State Machine (`EXSTAT` and `ROUTAD`)

The core of the game's logic is a state machine. The current state is stored as a numeric code in the Zero Page variable `QSTATE`. The `EXSTAT` subroutine is responsible for executing the logic associated with the current state.

### 2.1. `EXSTAT` Subroutine

This subroutine reads the value of `QSTATE`, uses it as an index into the `ROUTAD` address table, and then jumps to the corresponding routine. This is effectively a "computed GOTO" statement.

The jump is performed by pushing the high and low bytes of the target address from `ROUTAD` onto the stack and then executing an `RTS` (Return from Subroutine). The `RTS` instruction pops the address off the stack and continues execution there.

### 2.2. `ROUTAD` - Routine Address Table

This table is the map for the entire game flow. Each entry is a 16-bit address pointing to the subroutine that handles a specific game state. The index into this table is the state code divided by two (since addresses are two bytes).

| Index (Hex) | State Constant | Points To | Description |
|---|---|---|---|
| `00` | `CNEWGA` | `NEWGAM` | Initializes a new game. |
| `01` | `CNEWLI` | `NEWLIF` | Initializes a new player life. |
| `02` | `CPLAY` | `PLAY` | The main gameplay state (in `ALSCO2.MAC`). |
| `03` | `CENDLI` | `ENDLIF` | Handles the immediate aftermath of losing a life. |
| `04` | `CENDGA` | `ENDGAM` | Handles the "Game Over" sequence. |
| `05` | `CPAUSE` | `PAUSE` | A generic pause state, used for timed delays. |
| `06` | `CNEWAV` | `0` (null) | (Entry appears unused or handled elsewhere) |
| `07` | `CENDWAV` | `ENDWAV` | Logic after a wave is cleared. |
| `08` | `CHISCHK` | `HISCHK` | Checks for a new high score (in `ALSCO2.MAC`). |
| `09` | `CGETINI` | `GETINI` | Handles the "Get Initials" screen (in `ALSCO2.MAC`). |
| `0A` | `CDLADR` | `DLADR` | Attract mode: Displays the high score table. |
| `0B` | `CREQRAT` | `PRORAT` | Attract mode: Prompts for player rating (in `ALLANG.MAC`).|
| `0C` | `CNEWV2` | `NEWAV2` | Part 2 of the new wave initialization. |
| `0D` | `CLOGO` | `LOGINI` | Attract mode: Initializes the Atari logo display. |
| `0E` | `CINIRAT`| `INIRAT` | Attract mode: Displays the rating table (in `ALLANG.MAC`).|
| `0F` | `CNWLF2` | `NEWLF2` | Part 2 of the new life initialization. |
| `10` | `CDROP` | `PLDROP` | A special "drop" mode (in `ALDIS2.MAC`). |
| `11` | `CSYSTM` | `SYSTEM` | Displays the system configuration/DIP switches. |
| `12` | `CBOOM` | `PRBOOM` | Handles the Superzapper "boom" effect (in `ALDIS2.MAC`). |

---

## 3. Core Subroutines

### 3.1. `PAUSE`

This subroutine implements a timed delay. It decrements the `QTMPAUS` timer every few frames. When the timer reaches zero, it changes the game state (`QSTATE`) to the value stored in `QNXTSTA`. This is used for sequences like showing a message for a few seconds before returning to the attract mode.

### 3.2. `NONSTA` (Non-State Dependent Processing)

This routine is called every frame to handle inputs and events that are not specific to a single game state.
- **Test Mode:** It checks if the `MTEST` input is active. If so, it forces the game into the `CSYSTM` state to show the diagnostic and settings screen.
- **Attract Mode Coin-In:** If the game is in attract mode (`QSTATUS` bit `MATRACT` is 0), it manages the logic for coin insertion and the "2 GAME MINIMUM" option. It transitions to the `PROCRE` subroutine if credits are available.

### 3.3. `PROCRE` (Process Credits)

This routine is called when a player presses a Start button and credits are available.
- It decrements the credit count (`$$CRDT`).
- It determines if it's a 1-player or 2-player game based on which start button was pressed.
- It updates the player count (`NUMPLA`).
- It sets the game status to "in-game" (`MATRACT` flag).
- Most importantly, it changes the game state to `CNEWGA` to begin a new game.

---
## 4. Game State Subroutines

These are the implementations for the various game states, pointed to by the `ROUTAD` table.

### 4.1. `NEWGAM` (New Game)

This state is entered once at the very beginning of a new game.
- It calls `INICHK` and `INIDSP` to initialize various subsystems.
- It clears the scores for both players.
- It sets the initial number of lives (`LVSGAM`) and resets the wave count for each player in the game.
- It sets the first player (`NEWPLA = 0`).
- It transitions to the `INIRA0` routine (in `ALSCO2.MAC`), which handles the "select your starting level" sequence.

### 4.2. `NEWLIF` (New Life) & `NEWLF2` (New Life, Part 2)

This two-part state handles the setup for a new player turn.
1.  **`NEWLIF`**:
    -   If it's a 2-player game and control is switching from P1 to P2 (or vice-versa), it displays a "PLAY PLAYER X" warning message and pauses the game for a few seconds to allow the players to switch places. It also calls `SWAPEN` to handle enemy states for the new player.
    -   It calls `COCFLI` to flip the screen if it's a cocktail cabinet.
    -   It loads the new player's current wave number into `CURWAV`.
    -   It calls `INEWLI` to initialize the player's objects.
    -   It transitions its state to `CNWLF2`.
2.  **`NEWLF2`**:
    -   This state simply introduces a short, one-second pause (`CPAUSE`).
    -   After the pause, it sets the next state to `CPLAY` to begin gameplay.

### 4.3. `ENDLIF` (End of Life)

This state is triggered immediately after the player's ship is destroyed.
- It decrements the current player's life counter (`LIVES1` or `LIVES2`).
- It checks if any players have lives remaining.
    -   **If no lives are left for anyone:** It jumps to the `ENDGAM` (Game Over) routine.
    -   **If lives remain:** It determines the next player up. If the current player still has lives, they play again. If not, it switches to the other player. It then sets the next state to `CNEWLIF` to begin the new turn, with a pause to let the "player death" animation finish.

### 4.4. `ENDWAV` (End of Wave)

This state is triggered when all enemies on a level are cleared.
- It increments the current player's wave counter (`WAVEN1` or `WAVEN2`).
- It checks if the player earned a bonus score for completing the level and calls `BONSCO` if so.
- It transitions the state to `CNEWV2` and immediately falls through to the `INEWAV` routine (in `ALSCO2.MAC`) to initialize the enemies for the next wave.

### 4.5. `ENDGAM` (End of Game)

This is the game over state.
- It determines the highest wave number reached between the two players and stores it in `HIWAVE`.
- If the game was just played (i.e., not in attract mode), it sets the next state to `CHISCHK` to check if any scores made it to the high score table.
- Otherwise, it transitions to the `DLADR` state to show the high score table in attract mode.

### 4.6. `DLADR` (Display Ladder)

This is an attract mode state.
- It puts the game logic back into attract mode by clearing the `MATRACT` and `MGTMOD` flags.
- It sets the display state to `CDHITB` (Display High Score Table).
- It sets a long pause (`QTMPAUS = $A0`).
- After the pause, it sets the next state to `CLOGO` to restart the attract mode sequence.

---
## 5. Utility Subroutines

### 5.1. `COCFLI` (Cocktail Flip)

This utility is called to handle screen orientation for a cocktail table cabinet.
- It checks if the machine is in cocktail mode (`COCTAL` flag).
- If it is, and if `PLAYUP` indicates it's Player 2's turn, it sets the `MFLIP` bit in `TNKOUT` and sets the `TOUT0` register to flip the video hardware.
- For Player 1, or in an upright cabinet, it ensures the flip bits are cleared.

---

This concludes the documentation for `ALEXEC.MAC`. 