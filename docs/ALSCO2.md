# Tempest: ALSCO2.MAC - Scoring & Gameplay Module

**File:** `ALSCO2.MAC`
**Purpose:** Despite its name suggesting it's only for scoring, this is one of the most important modules in the game. It handles the rendering of all informational displays (scores, lives, messages), the high score table logic, and, most critically, contains the main `PLAY` subroutine that is executed every frame during active gameplay.

---

## 1. Informational Display Engine (`INFO`)

The `INFO` subroutine is the master routine for drawing all text and status information on the screen. It is called every frame from the `DISPLA` routine. It functions like a "HUD" or "UI" renderer.

### 1.1. Overview

The routine's behavior changes dramatically depending on whether the game is in attract mode or active gameplay (`QSTATUS` bit `MATRACT`).

-   **In Attract Mode:** It is responsible for flashing messages like "INSERT COIN", "PRESS START", or "GAME OVER".
-   **In Gameplay Mode:** It displays the scores and lives for the current player(s).
-   **In Post-Game Mode:** It handles rendering the high score entry screen.
-   **In Post-Wave Mode:** It displays messages like "SUPERZAPPER RECHARGED" and the bonus points awarded.

The entire on-screen text display is built from a master template located at `SCOBUF`. The `INFO` routine and its helpers modify this template in RAM before calling the vector generator to draw it.

### 1.2. Helper Subroutines

The `INFO` routine relies on several key helpers to perform its function.

#### `UPSCLI` (Update Score & Lives)

This subroutine draws the score and the life icons for one player.
- **Input:** Y-register contains the player ID (0 or 1).
- **Functionality:**
    1.  It sets the scale for the text. In attract mode or for the non-active player, the score is drawn smaller. For the active player, it's drawn larger.
    2.  It reads the number of lives for the specified player from `LIVES1` or `LIVES2`.
    3.  It draws one life icon for each life remaining. Crucially, if it's drawing for the currently active player, it draws `N-1` lives, because the player's ship on screen is considered the final life.
    4.  It calls `NWDIGS` to draw the player's score.

#### `NWDIGS` (New Digits) & `NWHEXZ` (New Hex with Zero-Suppression)

These are utility routines for rendering numbers.
- **`NWDIGS`**: Takes a pointer to a 3-byte, 6-digit BCD number (like the score) and calls `NWHEXZ` for each digit.
- **`NWHEXZ`**: Takes a single BCD digit (0-9) in the accumulator. It converts this to a vector graphic by looking up the digit's shape data. It includes logic for **zero-suppression**: it won't draw leading zeros until the first non-zero digit has been rendered. For example, a score of `007250` is rendered as `7250`.

---

*(More subroutines, including the main `PLAY` routine, will be documented as the rest of the file is analyzed.)*

## 2. Message and Text Display Engine

While `INFO` orchestrates the display, a set of specialized subroutines handles the rendering of specific, often centered, text messages.

### 2.1. `MSGS` (Message Subroutine)

This is the generic, core engine for drawing any string of text to the screen.
- **Input:** The X-Register holds a message ID, which is an index into a master table of message pointers (`LITRAL`).
- **Functionality:**
    1.  It looks up the message's data pointer from `LITRAL`. This data contains the string of character codes, along with attributes like color, scale, and X/Y position.
    2.  It positions the vector beam at the specified coordinates.
    3.  It iterates through the character codes in the message string.
    4.  For each character, it looks up a `JSRL` (Jump to Subroutine, Long) instruction from the `VGMSGA` table. This table contains the specific vector commands to draw that character.
    5.  It adds this `JSRL` command to the main display list (`VGLIST`), effectively drawing the character.
    6.  This continues until an end-of-string marker is found.

### 2.2. Specific Message Subroutines

These routines are simple wrappers around the `MSGS` engine, used to display specific, common messages.
- **`DPLPLA` (Display Play Player):** Displays "PLAY PLAYER X".
- **`DGOVER` (Display Game Over):** Displays "GAME OVER".
- **`DPRSTA` (Display Press Start):** Displays "PRESS START".
- **`D2GAME` (Display 2 Game Minimum):** Displays "2 GAME MINIMUM" and flashes "INSERT COIN" if needed.

### 2.3. `DSPCRD` (Display Credits)

This routine is called to render the credit count at the bottom of the screen.
- It looks up the coin mode message (e.g., "FREE PLAY") from the `TCOMOD` table.
- It displays the number of credits stored in `$$CRDT`, using `DSP1HX` to handle the hex-to-vector-digit conversion.
- It also handles displaying a "1/2" graphic if a partial credit has been registered.

### 2.4. `INITEM` (Initialize Item Template)

This routine is called at the start of a game to set up the informational display. It copies a master template from ROM (`SCORES`) into the active display buffer in RAM (`SCOBUF`). This ensures the display starts in a clean, consistent state before routines like `UPSCLI` begin modifying it with live game data.

---

## 3. High Score & Ranking Logic

This module contains all the logic for checking, entering, and displaying the high score table.

### 3.1. `HISCHK` (High Score Check)

*(This routine appears later in the file and will be documented in the next pass.)*

### 3.2. `GETINI` (Get Initials State)

This subroutine is the state machine for the high score name entry screen. It is executed when `QSTATE` is `CGETINI`.
- **Timer:** The player has a limited time to enter their initials, tracked by the `TIMHIS` counter. If it expires, the state aborts.
- **Input Handling:**
    - It reads the rotary dial input via `TBHD`.
    - It calls `GINICO` to "spin" the current letter up or down the alphabet based on the dial's movement.
    - It checks for the fire button press (`MFIRE` or `MSUZA`).
- **State Flow:**
    1. When the fire button is pressed, the current initial is locked in.
    2. The active initial index (`TBLIND`) is moved to the next slot.
    3. Once all three initials are entered, it calls `WRHIIN` to permanently save the new high score and initials to the EAROM (non-volatile memory).
    4. Finally, it transitions to the next logical state, which is typically to display the updated high score table.

### 3.3. `GETDSP` (Get Initials Display)

This is the corresponding display routine that renders the "Enter Your Initials" screen. It is active when `QDSTATE` is `CDGETI`.
- It calls `INFO` to draw the standard player scores.
- It calls the `MSGS` engine to draw the following text strings:
    - "PLAYER X"
    - "ENTER YOUR INITIALS"
    - "PRESS MOVE TO SELECT"
    - "PRESS FIRE TO ENTER"
- It then calls `LDRDSP` to draw the high score table itself, with logic to make the current entry "glow".

### 3.4. `LDRDSP` (Ladder Display)

This routine renders the entire high score table, often called a "ladder".
- It draws the "HIGH SCORES" title.
- It then enters a loop that runs 8 times (for the `NHISCO` entries).
- In each iteration, it draws one full line of the table:
    1.  The rank number (1 through 8).
    2.  The three initials for that entry.
    3.  The six-digit score for that entry.
- It contains logic to change the color of a specific line to white, used to highlight the spot where the player is entering their new high score.

### 3.5. `BOLOUT` (Bonus Life Out)

A small utility that displays the message "BONUS LIFE EVERY X0000 PTS", which is configured by the machine operator via DIP switches.

---

## 4. Core Gameplay Logic (`PLAY` routine)

The `PLAY` subroutine is the single most important routine during active gameplay. It is called every single frame when the game is in the `CPLAY` state. It acts as a master controller that updates every object and process in the game world.

The routine is a long, sequential block of code that calls out to numerous other subroutines to perform specific tasks.

**Execution Flow of the `PLAY` routine:**
1.  **`MOVCUR` (Move Cursor):** Processes the rotary dial input to move the player's "Blaster" ship around the rim of the web.
2.  **`DOFIRE` (Do Firing):** Checks if the fire button is pressed and, if so, creates a new player shot object.
3.  **`DOENER` (Do Enemies):** The main enemy AI routine. It updates the state and position of every enemy on the screen (Flippers, Tankers, Spikers, etc.).
4.  **`DOSHOT` (Do Shots):** Updates the position of all active player and enemy shots.
5.  **`DOEXPL` (Do Explosions):** Updates the animation of all active explosions.
6.  **`COLDET` (Collision Detection):** A large and critical routine that checks for collisions between:
    -   Player shots and enemies.
    -   Enemy shots and the player.
    -   The player and enemies on the rim.
    -   The Superzapper and enemies.
7.  **Check End Conditions:**
    -   If the player's Blaster has been destroyed, it transitions the game state to `CENDLI` (End of Life).
    -   If all enemies have been destroyed, it transitions the game state to `CENDWAV` (End of Wave).

## 5. Scoring and High Score Logic (Continued)

### 5.1. `HISCHK` (High Score Check)

This routine is called at the end of a game to see if the player's score qualifies for the high score table.
- **Input:** The current player's score.
- **Functionality:**
    1.  It iterates through the high score list stored in RAM (from `HSCORL` downwards).
    2.  It compares the player's score with each entry in the table.
    3.  If the player's score is higher than an entry, it finds the insertion point.
    4.  It then ripples all the lower scores down one slot in the table to make room for the new score.
    5.  It copies the player's score into the newly opened slot.
    6.  Finally, it sets the game state `QSTATE` to `CGETINI` so the player can enter their initials.
- **If no high score is achieved:** It transitions to the next state, typically `DLADR` to show the existing table in attract mode.

### 5.2. `UPSCOR` (Update Score)

This is the central utility for adding points to the player's score.
- **Input:** A 3-byte BCD value stored in `TEMP0`, `TEMP1`, and `TEMP2`.
- **Functionality:** It performs a 3-byte BCD addition of the input value to the current player's score (`LSCORL`/`RSCORL`). It also checks if the new score has crossed the threshold for a bonus life and sets the `BONUS` flag if it has.

### 5.3. `BONSCO` (Bonus Score)

This routine calculates the bonus points awarded at the end of a level and loads the value into the `TEMP` registers for `UPSCOR` to add to the player's score.

---

## 6. Initialization Routines

### 6.1. `INEWAV` (Initialize New Wave)

This routine is called at the start of each new level. Its primary responsibility is to call `INITE` to create the enemies for the upcoming wave based on the current level number (`CURWAV`).

### 6.2. `INEWLI` (Initialize New Life)

This routine is called at the start of a new life. It resets the state of the player's objects, ensuring the Blaster is ready for action.

### 6.3. `CLRSCO` (Clear Score)

This routine is called from `NEWGAM` and simply zeroes out the score, lives, and wave count for a given player, ensuring a clean slate for a new game.

---

This concludes the documentation for `ALSCO2.MAC`. 