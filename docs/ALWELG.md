# Tempest: ALWELG.MAC - Wave Initialization and Pre-Game Logic

**File:** `ALWELG.MAC`
**Purpose:** Despite its name, this file does not contain the main attract mode logic. Instead, it contains the critical routines for **initializing** new waves and new lives, and the complete state machine for the **"Rate Yourself"** screen where players select their starting level.

---

## 1. Game Initialization Subroutines

This file contains the master routines for setting up the game world at the start of a level or a new life.

### 1.1. `INEWAV` (Initialize New Wave)

This major subroutine is called at the beginning of every level. It is responsible for creating a new, clean playing field.
-   **`CONTOUR`:** It first calls a helper to set the "contour" or difficulty parameters for the new level (e.g., enemy speed, number of enemies).
-   **`INIENE` (Initialize Enemies):** It creates the enemies for the new wave.
-   **`INIOBJ` (Initialize Objects):** It deactivates any leftover game objects (like shots or explosions) from the previous level.
-   **`INISUZ` (Initialize Superzapper):** It recharges the player's Superzapper.
-   **Camera Reset:** It resets the 3D camera's position to the top of the well.

### 1.2. `INEWLI` (Initialize New Life)

This is a smaller initialization routine called when a player starts a new life (but not a new level). Its main purpose is to reset the player's state by calling `INICUR` and `INIOBJ` to ensure the player's ship is ready and the playfield is clear of old objects.

### 1.3. `NEWAV2` (New Wave, Part 2 - "Tube Fly-In")

This routine creates the iconic animation where the camera appears to fly down the tube into the next level.
-   It is a state that is executed over many frames.
-   On each frame, it interpolates the 3D camera's position (`EYL`, `EYH`, `ZADJL`) from the top of the old well to the bottom of the new one.
-   This smooth change in the "eye" position creates the swooping animation.
-   Once the camera reaches its destination, the routine transitions the game state to `CPLAY` to begin active gameplay.

---

## 2. "Rate Yourself" Screen Logic

This module contains the complete state machine for the pre-game screen that allows players to choose their starting level.

### 2.1. `INIRA0` (Initialize Rate Request)

This routine prepares the "Rate Yourself" screen.
-   It checks the `HIWAVE` variable (the highest level the player reached in their last game) to determine the maximum level the player is allowed to select. This prevents new players from starting on very high levels.
-   It initializes the selection cursor to the leftmost position.
-   It sets the game state to `CREQRAT` (Request Rate) and the display state to `CDREQRA` (Display Request Rate).
-   It then falls through to the `PRORAT` routine.

### 2.2. `PRORAT` (Process Rating)

This is the active state machine for the "Rate Yourself" screen.
-   **Timer:** A timer (`TIMHIS`) gives the player a limited time to choose. If the timer expires, the fire button is simulated automatically, selecting the current level.
-   **Input:** It calls `GETCUR` to read the rotary dial, which moves the selection cursor left and right across the available levels.
-   **Selection:** When the player presses the fire button:
    -   The currently selected level number is read from the `LEVEL` data table.
    -   This value is stored as the player's starting wave (`WAVEN1`).
    -   The routine then calls the core initialization helpers (`INICOL`, `CONTOUR`, `INIENE`) to set up the game for this chosen level.
    -   Finally, it transitions the game state to `NEWAV2` to begin the "fly-in" animation.

---

This concludes the documentation for `ALWELG.MAC`. 