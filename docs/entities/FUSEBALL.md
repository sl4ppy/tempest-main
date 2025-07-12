# Entity: Fuseball

This document provides a complete technical specification for the Fuseball enemy.

## 1. Identification and Purpose

-   **Entity Name:** Fuseball. Referred to in the source code as a "Fuse".
-   **Identifier:** Type `ZABFUS` (4) in the `INVADER` object array.
-   **Role and Purpose:** A highly dangerous and erratic enemy. The Fuseball moves rapidly up and down the well and will periodically attempt to flip directly into the player's current lane, making it a high-priority threat.
-   **Asset References:**
    -   **Vector Shape Data:** `ALVROM.MAC`, labels `PTFUSE` through `PTFUSE+6` (for its 4-frame animation).
    -   **Primary Logic:** `ALWELG.MAC` (routines: `NEWFUS`, `JFUSEUP`, `FUCHPL`, `LEFRIT`, `FUSEUP-CAM`).
    -   **Drawing Logic:** `ALDIS2.MAC` (routine: `FUSPIC`).
    -   **State Variables:** `ALCOMN.MAC` (within the `INVADER` object array).

## 2. Physical Attributes

-   **Size and Shape:** The Fuseball is a circular, spiky object. It has a 4-frame animation to give it a shimmering/rotating effect.
-   **Hitbox:** Collision with a player shot is determined by the `ENSIZE` table for its type (`ZABFUS`). A Fuseball is invulnerable to player shots while it is in the middle of a flip.
-   **Color:** Green (shares `TRACOL` with the Spiker).
-   **Rendering Layer:** Drawn by `DSPINV` with other invaders.

## 3. Movement and Locomotion

-   **Movement Type:** Bi-directional with a mix of random and targeted lane-changing.
-   **CAM Script:** The Fuseball is always assigned the `FUSEUP-CAM`. This script is a simple loop that calls the main `JFUSEUP` logic routine every frame.
-   **Base Movement (`JFUSEUP`):**
    -   The Fuseball moves up and down the well at double the speed of a Flipper (`WFUSIL`/`WFUSIH`).
    -   When it reaches the top (`CURSY`), it reverses direction.
    -   When it reaches the bottom of its range (`$80`), it has a chance to change lanes.
-   **Lane Changing (`MAYBLR`):**
    -   This is the core of the Fuseball's AI. Periodically, it makes a decision to change lanes.
    -   The frequency of this decision is controlled by `WFUFRQ`. A random number is checked against this value.
    -   If the decision is made to flip, the game then checks the `WFUSCH` (Fuse Chase) flag for the current wave.
        -   **Chase Mode (`FUCHPL`):** If the flag is set, the Fuseball will intelligently determine the shortest direction to flip to reach the player's current lane.
        -   **Random Mode (`LEFRIT`):** If the flag is not set, the Fuseball will flip left or right randomly.
    -   Once a flip is initiated, the CAM script temporarily changes to `FUSELR-CAM` to execute the flip, then returns to `FUSEUP-CAM`.

## 4. Animation Data

-   The Fuseball has a 4-frame animation, controlled by the `FUSPIC` drawing routine.
-   `FUSPIC` uses the global frame counter (`QFRAME`) to cycle through four vector shapes based at `PTFUSE`.

## 4.1. Programmatic and Transform Animations

The Fuseball has two distinct animations: a simple programmatic frame-cycle for its body and a complex procedural transform for its signature "flip" attack.

### A. Body Animation: Programmatic Frame-Cycle

The Fuseball's body appears to shimmer or rotate constantly. This is a time-based programmatic animation, identical in implementation to the Spiker's animation.

-   **Animation Type:** Programmatic, Frame-Sequence.
-   **Trigger:** The animation runs continuously and is updated every time the Fuseball is drawn via the `FUSPIC` routine.
-   **Core Logic:** `FUSPIC`.

#### Algorithm

1.  **Get Frame Counter:** The `FUSPIC` routine loads the global 8-bit frame counter, `QFRAME`.
2.  **Calculate Frame Index:** It performs a bitwise `AND` with `3` (`AND I,3`) to get a value that cycles `0, 1, 2, 3` at the game's full frame rate.
3.  **Retrieve Shape Pointer:** This index is used to select one of four vector shapes, with `PTFUSE` being the base address.
4.  **Render:** The selected shape is drawn at the Fuseball's current location.

### B. Flip Attack: Procedural 3D Transform

The Fuseball's signature flip is a complex procedural animation that calculates its position between two well lines. This is a different and more dynamic system than the Flipper's simple table-based flip.

-   **Animation Type:** Procedural 3D Transform with Interpolation.
-   **Trigger:** When a Fuseball initiates a flip, its `INVAL2` register is set to a negative value representing the flip's state, and the `FUSPIC` drawing routine executes a special rendering path.
-   **Core Logic:** `FUSPIC` (at label `M10`), `DELTA8`.

#### Algorithm

When `INVAL2` is negative, the following calculation occurs each frame to determine the Fuseball's on-screen position:

1.  **Get Base and Target Coordinates:** The routine gets the 3D coordinates for both the source line (`INVAL1`) and the destination line (`INVAL1 + 1` or `INVAL1 - 1`).
2.  **Calculate Delta:** It finds the difference (delta) in the X and Z axes between the two lines.
3.  **Calculate Interpolation Factor:** The core of the animation is in the `DELTA8` subroutine. It takes the total X/Z delta and multiplies it by the flip's progress, which is stored in the lower bits of `INVAL2`. The calculation is effectively:
    -   `Offset = (TotalDelta / 8) * FlipProgress`
    -   This is accomplished via a series of bit shifts and additions, creating a linear interpolation.
4.  **Apply Offset:** The calculated X/Z offset is added to the source line's base coordinates. This gives the final, interpolated 3D position of the Fuseball for that frame.
5.  **Render:** The standard, shimmering body animation (see Part A) is then rendered at this newly calculated 3D position.

The "animation" of the flip is therefore not a change in the Fuseball's shape, but a smooth, calculated movement along a straight path between the center points of the two lines. This, combined with the perspective projection, gives the appearance of a rapid, arcing attack.

## 5. State Machine / Behavior Logic

-   **State: Spawning**
    -   Fuseballs are created when a "Nymph" is converted via the `NEWFUS` routine.
-   **State: Patrolling**
    -   The Fuseball executes the `FUSEUP` CAM, moving up and down its current lane.
    -   Periodically, it enters the "Deciding" state.
-   **State: Deciding to Flip (`MAYBLR`)**
    -   The Fuseball decides whether to flip, and if so, whether to chase the player or flip randomly.
-   **State: Flipping**
    -   The CAM changes to `FUSELR`. The `INVMOT` flag is set.
    -   During this state, the Fuseball is **invulnerable**. The `COLCHK` routine in `ALWELG.MAC` specifically ignores collisions with Fuseballs whose `INVAL2` value indicates they are flipping.
-   **State: Direct Collision**
    -   The `JFUSKI` routine constantly checks if the Fuseball is on the same line and at the same height as the player. If so, it kills the player instantly (`INFPSQ`).

## 7. Combat / Ability Data

-   The Fuseball does not fire projectiles. Its offensive capability is its high speed and its targeted lane-changing maneuver, which often results in a direct collision with the player.
-   When destroyed by a player shot, it creates a unique, multi-stage explosion defined by `GEXIFU` and gives a higher point value.

## 11. Parameter Tables

| Parameter | Variable/Constant | Description |
|---|---|---|
| **Type ID** | `ZABFUS` | 4 | The internal identifier for a Fuseball. |
| **Velocity** | `WFUSIL`/`WFUSIH` | The speed at which the Fuseball moves (2x Flipper speed). |
| **CAM Script** | (Hardcoded) | `FUSEUP-CAM` / `FUSELR-CAM`. | The behavior scripts used by the Fuseball. |
| **Flip Frequency** | `WFUFRQ` | Controls how often the Fuseball decides to change lanes. |
| **Chase Flag** | `WFUSCH` | A per-wave flag that determines if the Fuseball will chase the player or move randomly. |
| **Invulnerability**| `INVAL2` | When flipping, this is set to `$20`, which the collision logic recognizes as an invulnerable state. |

The remaining sections are not applicable to this entity. 