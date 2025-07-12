# Entity: Enemy Bullet ("Invader Charge")

This document provides a complete technical specification for the enemy projectile, referred to in the source code as an "Invader Charge."

## 1. Identification and Purpose

-   **Entity Name:** Enemy Bullet / Invader Charge.
-   **Role and Purpose:** The primary ranged attack for several enemy types (Flippers, Tankers, some Pulsars). It travels from the enemy's position at the bottom of the well towards the player at the top.
-   **Asset References:**
    -   **Vector Shape Data:** `ALVROM.MAC`, label `PTESHO`.
    -   **Primary Logic:** `ALWELG.MAC` (routines: `FIREIC`, `MOVCHA`, `CHATOP`).
    -   **State Variables:** `ALCOMN.MAC` (within the `CHARGE` object array).
    -   **Drawing Logic:** `ALDIS2.MAC` (routine: `DSPCHG`).
    -   **Sound Effect (Launch):** `ESLSON`.

## 2. Physical Attributes

-   **Size and Shape:** A small shape defined by `PTESHO`. It has a 4-frame animation sequence.
-   **Hitbox:** An enemy bullet can be destroyed by a player bullet. This collision is checked in the `COLCHK` routine. The range is defined by `CHACHA`.
-   **Color:** White (`ICHCOL`).
-   **Rendering Layer:** Drawn by `DSPCHG`. Bullets are rendered in front of the player ship but behind other enemies and explosions.

## 3. Movement and Locomotion

-   **Movement Type:** Linear, constant velocity.
-   **Logic:** The `MOVCHA` routine in `ALWELG.MAC` updates the bullet's position each frame.
-   **Speed Values:**
    -   The bullet travels from the enemy towards the player at a speed controlled by the per-wave variables `WCHARL` (fractional) and `WCHARIN` (integer). Its `CHARY` value decreases each frame.

## 4. Animation Data

-   The `DSPCHG` drawing routine gives the enemy bullet a 4-frame animation.
-   It uses the global frame counter (`QFRAME`) to cycle through four shapes based at the `PTESHO` address.

## 4.1. Procedural and Programmatic Animation

The Enemy Bullet's animation is composed of two independent programmatic systems: a frame-cycled animation for its body and a procedural transform for its movement.

### A. Body Animation: Programmatic Frame-Cycle

The bullet's shape is in constant motion, giving it a shimmering or tumbling appearance. This is a simple, time-based programmatic animation.

-   **Animation Type:** Programmatic, Frame-Sequence.
-   **Trigger:** The animation runs continuously and is updated every time the bullet is drawn via the `DSPCHG` routine.
-   **Core Logic:** `DSPCHG`.

#### Algorithm

1.  **Get Frame Counter:** The `DSPCHG` routine loads the global 8-bit frame counter, `QFRAME`.
2.  **Calculate Frame Index:** It performs a bitwise `AND` with `3` (`AND I,3`). This isolates the two least significant bits of the frame counter, resulting in a value that cycles `0, 1, 2, 3` at the game's full frame rate.
3.  **Retrieve Shape Pointer:** This index is used to select one of four vector shapes, with `PTESHO` being the base address for the enemy bullet shapes.
4.  **Render:** The `ONELIN` routine is called to draw the selected vector shape at the bullet's current location.

#### Parameters and Timing
-   **`QFRAME`:** The global frame counter, used as the animation's time source.
-   **Animation Rate:** The animation cycles at the full frame rate of the game (60Hz). The full animation loop takes 4 frames, or approximately 67 milliseconds.
-   **Vector Shapes:** The four shapes are located sequentially in memory starting at the `PTESHO` label.

### B. Movement: Procedural Constant Velocity Transform

The bullet's primary animation is its continuous movement from the enemy towards the player at the top of the well.

-   **Animation Type:** Procedural, Constant Velocity Transform.
-   **Core Logic:** The `MOVCHA` routine updates the position of all active bullets each frame.
-   **Algorithm:**
    1.  For each active enemy bullet, the routine subtracts a fixed 16-bit velocity value from its 16-bit `CHARY`/`CHARYL` position.
    2.  The new 3D position is then projected to the screen by the `DSPCHG` and `ONELIN` routines.

-   **Mathematical Formula:**
    -   `Position(t) = Position(t-1) - Velocity`

-   **Parameters:**
    -   **`WCHARIN` / `WCHARL`:** A 16-bit fixed-point velocity value that is defined per wave. This value is subtracted from the bullet's `CHARY`/`CHARYL` position each frame, moving it toward the player at a constant speed.

## 5. State Machine / Behavior Logic

An Enemy Bullet is either `Active` or `Inactive`. It uses the `CHARGE` object array slots from `NPCHARG` to `NPCHARG + NICHARG - 1`.

-   **State: Inactive**
    -   **Condition:** The `CHARY` for its slot is `0`.
-   **State: Active**
    -   **Instantiation (`FIREIC`):**
        -   An enemy can fire if it is low enough on the screen, not in the middle of a flip, and its personal fire timer has expired.
        -   A random check (`CHANCE` table) is performed, making it less likely for enemies to fire if many enemy shots are already on screen.
        -   If all conditions pass, a vacant slot in the `CHARGE` table is found. The bullet is created at the enemy's position (`INVAY`, `INVAL1`, `INVAL2`).
        -   The `ESHCOU` (Enemy Shot Count) is incremented.
    -   **Behavior:**
        1.  Its `CHARY` position is decreased by `WCHARIN`/`WCHARL` each frame.
        2.  It is checked for collisions with the player and player bullets.
    -   **Deactivation Conditions:**
        -   It reaches the top of the well (`CHARY` <= `CURSY`).
        -   It is destroyed by a player bullet (`COLCHK`).
        -   The bullet is deactivated by setting its `CHARY` back to `0` and decrementing `ESHCOU`.

## 6. Interaction and Input Handling

-   Not applicable.

## 7. Combat / Ability Data

-   **Player Collision (`CHATOP`):**
    -   When an enemy bullet's `CHARY` position reaches the player's `CURSY` depth, this routine is called.
    -   It checks if the bullet is on the same line segment as the player (`CHARL1` matches `CURSL1`).
    -   If it is a match and the player is alive, the `INCPSQ` routine is called to kill the player.

## 8. Sound and Audio Integration

-   The `ESLSON` (Enemy Shot Launch Sound) is played by `FIREIC` when a shot is successfully fired.

## 11. Parameter Tables

| Parameter | Variable/Constant | Description |
|---|---|---|
| **Shape Data** | `PTESHO` | Pointer to the vector data for the Enemy Bullet shape. |
| **Max On-Screen** | `NICHARG` | 4 | Number of available slots for enemy bullets. |
| **Velocity** | `WCHARIN`/`WCHARL` | The speed at which the bullet travels towards the player. Set per wave. |
| **Fire Chance Table**| `CHANCE`| A table of probabilities used to throttle enemy firing rate. |

The remaining sections are not applicable to this entity. 