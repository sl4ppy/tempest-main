# Entity: Flipper

This document provides a complete technical specification for the Flipper enemy.

## 1. Identification and Purpose

-   **Entity Name:** Flipper.
-   **Identifier:** Type `ZABFLI` (0) in the `INVADER` object array.
-   **Role and Purpose:** The most common and basic enemy in Tempest. Flippers continuously move up the well from the bottom towards the player, periodically "flipping" to adjacent web lines. When they reach the top rim, they become "Chasers" and move along the rim, attempting to collide with the player.
-   **Asset References:**
    -   **Vector Shape Data:** `ALVROM.MAC`, labels `ENER11` to `ENER14` (which are aliases for `CINVA1`).
    -   **Primary Logic:** `ALWELG.MAC` (routines: `MOVINV`, various CAM scripts).
    -   **Drawing Logic:** `ALDIS2.MAC` (routine: `FLIPIC`).
    -   **State Variables:** `ALCOMN.MAC` (within the `INVADER` object array).

## 2. Physical Attributes

-   **Size and Shape:** The Flipper is a two-pronged shape defined by `CINVA1`. It does not have different animation frames for its body.
-   **Hitbox:** Collision with a player shot is determined by the `ENSIZE` table for its type (`ZABFLI`). Collision with the player ship at the top of the well is determined by sharing the same line segment (`INVAL1` or `INVAL2` matching `CURSL1` or `CURSL2`).
-   **Color:** Red (`FLICOL`).
-   **Rendering Layer:** Drawn by `DSPINV` with other invaders on the well's rim.

## 3. Movement and Locomotion

The Flipper's movement is dictated by a "Cameo" (CAM) script assigned to it for the current wave.

-   **Base Movement:** All Flippers have a base upward velocity (`WINVIL`/`WINVIN`) that moves them from the bottom of the well (`ILINDDY`) towards the top (`ILINLIY`). This velocity is constant per wave.
-   **Flipping:** The Flipper's signature behavior is flipping to an adjacent line. A flip is a scripted movement between its current line (`INVAL1`) and a new line. During a flip, the `INVMOT` flag in `INVAC1` is set, and the `FLIPIC` drawing routine renders it at an interpolated position between the two lines.

## 4. Animation Data

-   **Stationary/Moving:** The Flipper uses a single vector shape (`CINVA1`) for its body. It does not have a walk or idle animation.
-   **Flipping:** The "animation" of flipping is procedural. The `IJMPDS` routine calculates the object's world coordinates while it is between lines, and it is rendered there. The shape itself does not change.

## 4.1. Procedural Animation: The Flip

The Flipper's signature "flip" is a procedural transform animation. The Flipper does not have distinct animation frames for this action. Instead, the game dynamically calculates its position in 3D space as it traverses the gap between two well lines, then projects that position to the screen.

-   **Animation Type:** Procedural 3D Transform.
-   **Trigger:** A Flipper's assigned CAM script initiates a flip, setting the `INVMOT` flag. This directs the `FLIPIC` drawing routine to use an alternate rendering path.
-   **Core Logic:** `FLIPIC`, `IJMPDS`, `ONELN2`.

### Algorithm

When the `INVMOT` flag is set, the `FLIPIC` routine calls `IJMPDS` to calculate the Flipper's world coordinates for the current frame of the animation.

1.  **Get Base Position:** `IJMPDS` retrieves the 3D world coordinates (X and Z) of the Flipper's current line (`INVAL1`) from the `LINEX` and `LINEZ` tables.
2.  **Calculate Offset:** It then determines a positional offset based on the *destination* line (`INVAL2`). The destination line number is used as an index into two lookup tables: `JUMPX` and `JUMPZ`.
3.  **Apply Offset:** The values from `JUMPX` and `JUMPZ` are added to the base line's coordinates. This results in a new, interpolated (X, Z) coordinate pair that lies on a predefined arc between the source and destination lines. The Y-depth is unchanged.
4.  **Render at Interpolated Position:** The `ONELN2` routine is called, which takes these newly calculated 3D coordinates, performs the standard perspective projection and scaling via the `WORSCR` and `CASCAL` routines, and draws the Flipper's static `CINVA1` vector shape.

### Parameters and Data Tables

-   **`INVAL1`:** The source line number (0-15). Used to get the base 3D position.
-   **`INVAL2`:** The destination line number (0-15). Used as an index into the offset tables.
-   **`JUMPX` / `JUMPZ`:** These tables contain 16 fixed-point values each. They define the X and Z components of the arc that all flipping enemies follow. The path is not calculated with an easing curve; it's a direct lookup of positional offsets. For any given destination line, the offset is constant.
-   **Duration:** The duration of the flip (the number of frames for which `INVMOT` is set) is not controlled by the animation logic itself, but by the Flipper's governing CAM script (e.g., `VJUMPS`, `SPIRAL`). The visual animation simply renders the Flipper at whatever point along its fixed path the CAM script dictates for that frame.

This system creates a convincing and consistent 3D "flip" effect without the need for complex real-time rotation or multiple animation frames, using just a single vector shape and a table-driven coordinate transformation.

## 5. State Machine / Behavior Logic

A Flipper's behavior is controlled by its assigned CAM script. The `WFLICAM` variable holds the starting address of the CAM script for Flippers on the current wave.

-   **State: Spawning**
    -   Flippers are created when a "Nymph" reaches the top of the well and is converted into an invader via the `CONYMP` and `ACTINV` routines.
-   **State: Advancing (In Well)**
    -   **Behavior:** The Flipper executes its assigned CAM script, which is a combination of moving up (`VSMOVE`) and flipping (`VJUMPS`/`VJUMPM`).
    -   **Common CAM Scripts for Flippers:**
        -   `NOJUMP`: Moves straight up without flipping.
        -   `MOVJMP`: Moves up for 8 frames, then flips once. Repeats.
        -   `SPIRAL`: Moves up and flips every frame, creating a spiral path.
        -   `SPIRCH`: Spirals one direction for 2 flips, reverses, spirals the other way for 3 flips.
        -   `AVOIDR`: Actively attempts to dodge the player by flipping away from their position.
-   **State: Chasing (On Rim)**
    -   **Condition:** The Flipper's `INVAY` position reaches the top of the well (`CURSY`). The `CHASER` routine is called.
    -   **Behavior:** The Flipper's CAM script is changed to `TOPPER`. In this state, it no longer moves down the well. It continuously checks if it's on the same line as the player (`VKITST`) to cause a collision. It flips along the rim at a speed controlled by `WTTFRA` (Wave Top Flipper Rate).
-   **Firing:** A Flipper can fire an enemy projectile if it is not currently flipping (`INVMOT`=0), its personal fire timer (`INVACT`) has expired, and a random check passes.

## 7. Combat / Ability Data

-   **Attack:** Can fire a standard enemy projectile.
-   **Projectile Definition:** Firing is handled by the `FIREIC` routine. See the "Enemy Bullet" documentation for details.
-   **Death:** When hit by a player shot, the `INCISQ` routine is called, which deactivates the Flipper and creates a standard explosion entity at its position.

## 11. Parameter Tables

| Parameter | Variable/Constant | Description |
|---|---|---|
| **Type ID** | `ZABFLI` | 0 | The internal identifier for a Flipper. |
| **Upward Velocity** | `WINVIL`/`WINVIN` | The speed at which the Flipper moves up the well. Set per wave. |
| **Top Flip Rate**| `WTTFRA` | Controls the speed at which a "Chaser" Flipper flips along the rim. |
| **Fire Timer**| `WCHARFR` | The base value for the personal fire timer `INVACT`. |
| **CAM Script** | `WFLICAM` | The memory address of the behavior script used for the current wave. |
| **Motion Flag** | `INVMOT` | A flag in `INVAC1` indicating the Flipper is currently between lines. |
| **Rotation Flag** | `INVROT` | A flag in `INVAC1` indicating the Flipper's preferred flip direction (CW/CCW). |

The remaining sections are not applicable to this entity. 