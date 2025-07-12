# Entity: Tanker

This document provides a complete technical specification for the Tanker enemy.

## 1. Identification and Purpose

-   **Entity Name:** Tanker.
-   **Identifier:** Type `ZABTAN` (2) in the `INVADER` object array.
-   **Role and Purpose:** A slow-moving, durable enemy that acts as a "carrier." Its primary threat is not its own attack, but the two Flippers it releases upon destruction.
-   **Asset References:**
    -   **Vector Shape Data:** `ALVROM.MAC`, labels `PTTANK`, `PTTANP`, `PTTANF`.
    -   **Primary Logic:** `ALWELG.MAC` (routines: `MOVINV`, `KILINV`, `NEWTAN`).
    -   **Drawing Logic:** `ALDIS2.MAC` (routine: `TANPIC`).
    -   **State Variables:** `ALCOMN.MAC` (within the `INVADER` object array).

## 2. Physical Attributes

-   **Size and Shape:** The Tanker is a large, diamond-like shape. Its appearance can vary slightly based on the contents it carries (see Section 4).
-   **Hitbox:** Collision with a player shot is determined by the `ENSIZE` table for its type (`ZABTAN`).
-   **Color:** Purple (`TANCOL`).
-   **Rendering Layer:** Drawn by `DSPINV` with other invaders on the well's rim.

## 3. Movement and Locomotion

-   **Movement Type:** Linear, constant velocity.
-   **Behavior:** The Tanker is always assigned the `NOJUMP` CAM script. This means its only behavior is to move straight up the well from the bottom (`VSMOVE` opcode). It never flips or deviates from its path.
-   **Speed:** Its upward velocity (`WINVIL`/`WINVIN`) is set per wave and is identical to the Flipper's base speed for that wave.

## 4. Animation Data

-   The `TANPIC` drawing routine selects a vector shape for the Tanker based on the contents it is carrying, as defined by the `INVCAR` bits in its `INVAC2` status byte.
-   The `TANTAB` table maps contents to shapes:
    -   **Carrying Nothing (0):** `PTTANK`
    -   **Carrying Flippers (1):** `PTTANK`
    -   **Carrying Pulsars (2):** `PTTANP` (Pulsar Tanker)
    -   **Carrying Fuses (3):** `PTTANF` (Fuse Tanker)
-   By default, Tankers are created carrying Flippers. The ability to carry other enemy types is determined by the `WWTAC` tables for later waves.

## 4.1. Programmatic Animation: State-Based Shape Selection

The Tanker's only form of animation is programmatic and state-based. It does not have articulated motion or transformational animations. Instead, its vector shape is selected based on the type of enemy it is designated to be "carrying."

-   **Animation Type:** Programmatic, State-Based Frame Selection.
-   **Trigger:** The animation is determined at the time of rendering by the `TANPIC` routine. It is not a timed sequence.
-   **Core Logic:** `TANPIC`, `TANTAB`.

### Algorithm

The rendering process for a Tanker involves selecting a vector shape from a table based on a state variable.

1.  **Read State:** The `TANPIC` routine reads the `INVAC2` byte for the specific Tanker being drawn.
2.  **Isolate Cargo Bits:** It performs a bitwise `AND` with the `INVCAR` mask to isolate the bits that define what cargo the Tanker is carrying.
3.  **Table Lookup:** The resulting value (0-3) is used as a direct index into the `TANTAB` data table.
4.  **Retrieve Shape Pointer:** `TANTAB` contains a list of pointers to different vector shape data (`PTTANK`, `PTTANP`, `PTTANF`). The appropriate pointer is retrieved.
5.  **Render:** The `SCAPIC` routine is called, which uses the retrieved pointer to draw the selected shape on the screen, handling the necessary 3D projection and scaling.

### Parameters and Data Tables

-   **`INVAC2` (bitmask `INVCAR`):** A state variable for each Tanker that holds its cargo type.
    -   `0` or `1`: Carrying Flippers.
    -   `2`: Carrying Pulsars.
    -   `3`: Carrying Fuses.
-   **`TANTAB`:** A lookup table mapping the cargo type to a specific vector shape data address.
    -   `TANTAB[0] -> PTTANK`
    -   `TANTAB[1] -> PTTANK`
    -   `TANTAB[2] -> PTTANP`
    -   `TANTAB[3] -> PTTANF`

This mechanism allows the game to display visually distinct Tankers for different waves or game events without requiring separate enemy types, using a simple, efficient, state-driven approach to animation.

## 5. State Machine / Behavior Logic

-   **State: Spawning**
    -   Tankers are created when a "Nymph" is converted via the `NEWTAN` routine.
    -   Upon creation, its `INVAC2` status byte is set to `ZCARFL | ZFIRYE | ZDIRUP`, marking it as a Flipper carrier that can fire and moves up.
-   **State: Advancing**
    -   The Tanker executes the `NOJUMP` CAM script, moving straight up the well.
-   **State: Firing**
    -   A Tanker can fire a standard enemy projectile. Firing logic is identical to the Flipper: it can fire if not flipping (which it never does), its fire timer (`INVACT`) has expired, and a random check passes.
-   **State: Destroyed (Splitting)**
    -   **Trigger:** The Tanker is hit by a player shot. The `INCISQ` routine is called, which then calls `KILINV`.
    -   **Behavior (`KILINV`):**
        1.  The original Tanker entity is deactivated.
        2.  The code checks the `INVCAR` flag. Since the Tanker is a carrier, the splitting logic is executed.
        3.  The `ACTINV` (Activate Invader) routine is called **twice**.
        4.  Two new **Flipper** entities are created and activated.
        5.  The new Flippers are positioned to the left and right of the Tanker's death location, inheriting its depth (`INVAY`). One is given the `ZROCCW` rotation flag to encourage them to split apart.

## 7. Combat / Ability Data

-   **Attack:** Can fire a standard enemy projectile.
-   **Death:** The Tanker's death action is its defining characteristic: it splits into two new Flipper enemies.

## 11. Parameter Tables

| Parameter | Variable/Constant | Description |
|---|---|---|
| **Type ID** | `ZABTAN` | 2 | The internal identifier for a Tanker. |
| **Upward Velocity**| `WINVIL`/`WINVIN` | The speed at which the Tanker moves up the well. |
| **CAM Script** | (Hardcoded) | `NOJUMP-CAM`. | The only behavior script used by the Tanker. |
| **Carrier Flag** | `INVCAR` | Bits in `INVAC2` that identify the entity as a carrier. |
| **Contents** | `ZCARFL` | The value in the `INVCAR` bits indicating it carries Flippers. |

The remaining sections are not applicable to this entity. 