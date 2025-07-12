# Entity: Spiker & Spike

This document provides a complete technical specification for the Spiker enemy and the "Spike" hazard it creates.

## 1. Identification and Purpose

-   **Entity Name:** Spiker. Referred to in the source code as "Trailer."
-   **Identifier:** Type `ZABTRA` (3) in the `INVADER` object array.
-   **Role and Purpose:** A specialized enemy that moves up and down a single line of the well. Its primary purpose is to create a "Spike," a persistent line on the web that is lethal to the player during the drop between levels.
-   **Asset References:**
    -   **Vector Shape Data:** `ALVROM.MAC`, labels `PTSPI1` through `PTSPI1+6` (for its 4-frame animation).
    -   **Primary Logic:** `ALWELG.MAC` (routines: `NEWSPI`, `JSTRAI`, `TRALUP-CAM`).
    -   **Drawing Logic:** `ALDIS2.MAC` (routine: `TRAPIC`).
    -   **State Variables:** `ALCOMN.MAC` (within `INVADER` and `LINEY` arrays).

## 2. Physical Attributes (Spiker)

-   **Size and Shape:** The Spiker is a long, thin, segmented object. It has a 4-frame animation sequence.
-   **Hitbox:** Collision with a player shot is determined by the `ENSIZE` table for its type (`ZABTRA`).
-   **Color:** Green (`TRACOL`).
-   **Rendering Layer:** Drawn by `DSPINV` with other invaders.

## 3. Movement and Locomotion (Spiker)

-   **Movement Type:** Linear, bi-directional.
-   **Behavior:** The Spiker is always assigned the `TRALUP-CAM` script.
    -   This script consists of a tight loop: move one step (`VSMOVE`), then process special logic (`VSTRAI`).
    -   The Spiker only moves up and down along a single line of the well (`INVAL1`). It never flips to adjacent lines.
-   **Movement Logic (`JSTRAI`):**
    -   The Spiker moves up the well until its `INVAY` reaches a maximum height of `$20`.
    -   Once it hits the top, its direction is reversed (`INVDIR` flag is set), and it begins moving down.
    -   When it reaches the bottom (`$F2`), the `ASTRAL` routine is called to find a new, empty line for it to spawn on, and its direction is reversed again.
    -   If a Spiker reaches the bottom and there are no Nymphs left on screen (`NYMCOU`=0), it converts itself into a Tanker.

## 4. Animation Data (Spiker)

-   The Spiker has a 4-frame animation, controlled by the `TRAPIC` drawing routine.
-   `TRAPIC` uses the global frame counter (`QFRAME`) to cycle through four vector shapes: `PTSPI1`, `PTSPI1+2`, `PTSPI1+4`, and `PTSPI1+6`. This gives the Spiker a constantly writhing appearance.

## 4.1. Procedural and Programmatic Animations

The Spiker is associated with two distinct animations: a programmatic, frame-cycled animation for its own body, and a procedural, transform-based animation that creates the "Spike" hazard on the well.

### A. Spiker Body: Programmatic Frame-Cycle Animation

The Spiker's body is in constant motion, giving it a writhing or undulating appearance. This is a simple, time-based programmatic animation.

-   **Animation Type:** Programmatic, Frame-Sequence.
-   **Trigger:** The animation runs continuously and is updated every time the Spiker is drawn via the `TRAPIC` routine. It is not dependent on the Spiker's state or movement.
-   **Core Logic:** `TRAPIC`.

#### Algorithm

1.  **Get Frame Counter:** The `TRAPIC` routine loads the global 8-bit frame counter, `QFRAME`.
2.  **Calculate Frame Index:** It performs a bitwise `AND` with `3` (`AND I,3`). This isolates the two least significant bits of the frame counter, resulting in a value that cycles `0, 1, 2, 3, 0, 1, ...` on every frame.
3.  **Retrieve Shape Pointer:** This 2-bit index is used to look up a pointer to one of the four pre-defined Spiker vector shapes (`PTSPI1` to `PTSPI1+6`).
4.  **Render:** The `SCAPIC` routine is called to draw the selected vector shape at the Spiker's current location.

#### Parameters and Timing
-   **`QFRAME`:** The global frame counter, used as the animation's time source.
-   **Animation Rate:** The animation cycles at the full frame rate of the game (60Hz), with each of the 4 frames being displayed for a single game frame. The full animation loop takes 4 frames, or approximately 67 milliseconds.
-   **Vector Shapes:** `PTSPI1`, `PTSPI1+2`, `PTSPI1+4`, `PTSPI1+6` are the four shapes that comprise the animation.

### B. Spike Creation: Procedural Geometry Transform

The creation of the Spike is a procedural animation that directly transforms the geometry of the game world.

-   **Animation Type:** Procedural, Vertex Transform.
-   **Trigger:** The animation is updated every frame that a Spiker executes the `VSTRAI` opcode in its `TRALUP-CAM` script.
-   **Core Logic:** `JSTRAI`.

#### Algorithm

1.  **Identify Target Line:** The game identifies the line number the Spiker is currently on (`INVAL1`).
2.  **Get Spiker Position:** It gets the Spiker's current Y-depth (`INVAY`).
3.  **Update Well Geometry:** The `JSTRAI` routine directly writes the Spiker's `INVAY` value into the `LINEY` array at the index corresponding to the Spiker's line.
    -   `LINEY[INVAL1] = INVAY`

The `LINEY` array defines the Z-coordinate for the "end" of each of the 16 well lines. By dynamically modifying this array, the `JSTRAI` routine procedurally animates the well itself, pulling the end of one of its lines upward to create a solid, lethal wall—the Spike. This transformation persists until the Spiker is destroyed or moves to a new line.

## 5. The "Spike" Hazard

The Spike is not a mobile entity but a persistent environmental hazard created by the Spiker.

-   **Mechanism:** The `LINEY` array stores the Y-depth of the "end" of the web on each of the 16 lines. This is normally the very bottom of the well (`$F0`).
-   **Creation (`JSTRAI`):**
    -   As the Spiker moves, the `JSTRAI` routine continuously updates the `LINEY` value for the column the Spiker is in (`INVAL1`).
    -   It sets `LINEY[n]` to the Spiker's current `INVAY` position.
    -   This effectively "pulls" the end of the web up, creating a wall or "Spike" that extends from the bottom to the Spiker's current position.
-   **Player Interaction:**
    -   During the "drop" phase between levels, the player's ship is killed if it collides with a Spike.
    -   The `MOVCUD` routine checks if the player's depth (`CURSY`) is equal to or greater than the `LINEY` value for the player's current line (`CURSL1`). If so, a collision is registered, and the player is killed (`INPPSQ`).
-   **Removal:** Spikes are removed when the next wave is initialized (`INIENE` resets the `LINEY` array).

## 7. Combat / Ability Data (Spiker)

-   The Spiker does not fire projectiles. Its offensive capability is the creation of Spikes.
-   It can be destroyed by a single player shot. When destroyed, it does not split or create other enemies.

## 11. Parameter Tables

| Parameter | Variable/Constant | Description |
|---|---|---|
| **Type ID** | `ZABTRA` | 3 | The internal identifier for a Spiker ("Trailer"). |
| **Upward/Downward Velocity** | `WINVIL`/`WINVIN` | The speed at which the Spiker moves. Set per wave for the `ZABTRA` type. |
| **CAM Script** | (Hardcoded) | `TRALUP-CAM`. | The only behavior script used by the Spiker. |
| **Max Height** | (Hardcoded) | `$20` | The Y-depth the Spiker reaches before reversing. |
| **Min Height** | (Hardcoded) | `$F2` | The Y-depth the Spiker reaches before finding a new line. |

The remaining sections are not applicable to this entity. 