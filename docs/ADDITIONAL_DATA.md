# Tempest: Additional Data and Non-Tabular Content

This document covers the data categories from the original request that are not represented by discrete, tabular data files. It explains how these systems are implemented in Tempest.

## State Transition Data

The game's primary [state machine](./SYSTEMS.md#5-game-state-machine) is implemented directly in the 6502 assembly code, not in data tables. The core logic for game flow is located in `ALEXEC.MAC` and `ALWELG.MAC`.

A central variable, `QSTATE`, holds the current game state code. At the start of each frame, the main loop in `ALEXEC.MAC` uses the value of `QSTATE` as an index into a jump table (`ROUTAD`) to execute the appropriate subroutine for the current state.

**Example State Machine Flow (`ALEXEC.MAC`):**
1.  Game starts in state [`CNEWGA` (New Game)](./GAME_STATE_FLOW.md#41-state-cnewga-new-game). The `ROUTAD` table points this to the `NEWGAM` subroutine.
2.  `NEWGAM` initializes the game and sets the next state to [`CINIRAT` (Initialize Rate Yourself screen)](./GAME_STATE_FLOW.md#42-state-cinirat--creqrat-skill-selection).
3.  The `PRORAT` routine in `ALWELG.MAC` handles the "Rate Yourself" screen. When the player makes a selection, it sets the next state to [`CNEWV2` (Warp Transition)](./GAME_STATE_FLOW.md#44-state-cendwav--cnewv2-end-of-wave-and-warp).
4.  The `NEWAV2` routine sets up the game and transitions to the main [`CPLAY` state](./GAME_STATE_FLOW.md#43-state-cplay-gameplay), handled by the `PLAY` subroutine in `ALWELG.MAC`.
5.  If the player dies, the state changes to [`CENDLI` (End Life)](./GAME_STATE_FLOW.md#45-state-cendli--cnewli--cnwlf2-player-death--respawn), which might transition back to `CNEWLI` or to [`CENDGA` (End Game)](./GAME_STATE_FLOW.md#46-state-cendga--chischk--cgetini-game-over--and-high-scores).

The conditions for state transitions (e.g., timers expiring, player actions, all enemies destroyed) are checked within the code of each state's subroutine. For a complete understanding of the state transitions, please refer to the [Game State Flow](./GAME_STATE_FLOW.md) documentation.

## Particle Systems

Tempest does not have a formal particle system engine in the modern sense. However, it creates several particle-like effects using the [Vector Engine](./SYSTEMS.md#1-vector-graphics-engine) and timed software routines.

-   **Explosions:** Standard enemy explosions are a sequence of four pre-defined vector shapes (`EXPL1` to `EXPL4`) that are displayed in sequence to create an expanding animation. The "shrapnel" explosion (`SHRAP`) is a single, complex vector shape composed of many disconnected lines.
-   **Superzapper:** The [Superzapper's](./ENTITIES.md#3-superzapper) "boom" effect is created by spawning a number of particles (`NPARTI = 10`) at the player's location. Each particle is a single line vector. The game logic then applies a velocity and deceleration (`PARLXA`, `PARLYA`, `PARLZA`) to each particle over time, creating an expanding wave.
-   **Player Shots:** [Player shots](./ENTITIES.md#2-player-bullet) are simply a static vector shape (`DIARA2`) that is moved along a trajectory by the game logic.

The vector shapes for these effects are documented in the [Vector Shapes data](./DATA_ASSETS.md#2-vector-shapes), and the constants controlling their behavior are documented in `docs/GAME_TUNING.md`.

## Sprite Sheets and Texture Atlases

Tempest is a **vector graphics** game. It does not use any raster graphics, sprites, or textures. All graphics—from the player's ship to the text and enemies—are drawn on the screen by a beam of light as a series of connected lines (vectors). The data for these shapes is stored as lists of [vector commands](./DATA_ASSETS.md#2-vector-shapes), not as pixel bitmaps. 