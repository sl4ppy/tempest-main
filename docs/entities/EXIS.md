# "Entity": The Exis (Chaser State)

This document clarifies the nature of the entity commonly referred to as the "Exis."

## 1. Identification and Purpose

-   **Name:** Exis / Chaser
-   **Role and Purpose:** Contrary to common belief, the "Exis" is not a unique enemy type within the game's code. The term refers to any standard enemy (typically a Flipper or Pulsar) that has successfully traveled the entire length of the well and reached the top rim where the player resides. Upon reaching the rim, the enemy enters a "Chaser" state.

## 2. Behavior and Logic

-   **Trigger:** An enemy enters the Chaser state when its `INVAY` position reaches the player's `CURSY` position at the top of the well. The `CHASER` routine in `ALWELG.MAC` is called.
-   **State Change:**
    1.  The enemy's `INVCAM` (Cameo/AI script pointer) is changed to point to the `TOPPER` script.
    2.  The global `INCCOU` (Invader Chaser Count) is incremented, and the `INMCOU` (Invader Mover Count) is decremented.
-   **`TOPPER` CAM Script:** This AI script dictates the Chaser's behavior:
    1.  **Wait:** The enemy waits in a crouched position for a few frames.
    2.  **Attack:** It constantly checks if it is on the same line segment as the player (`VKITST`). If it is, it kills the player instantly.
    3.  **Move:** It "jumps" or "flips" along the rim from one line to the next. The speed of this movement is controlled by the `WTTFRA` (Wave Top Flipper Rate) parameter, which is set per wave.
-   **End of Wave Condition:** The wave does not end until all enemies are destroyed. This includes any enemies that have become Chasers. The player must hunt down and destroy all Chasers on the rim to proceed to the next level.

## 3. Relation to Other Enemies

-   **Flippers:** The most common enemies to become Chasers.
-   **Pulsars:** Can also become Chasers, but their behavior is slightly different. If there are still Nymphs spawning, a Pulsar that reaches the top will simply reverse direction and go back down the well instead of entering the `TOPPER` CAM.

## 4. Procedural / Programmatic / Transform Animations

The "Exis" or "Chaser" state does not have its own unique animation logic. Instead, it reuses the animation capabilities of the base enemy that entered the state.

-   **Animation Type:** Inherited Procedural 3D Transform.
-   **Core Logic:** The `TOPPER` CAM script, `VJUMPS` and `VJUMPM` opcodes, and the animation routines of the base enemy (e.g., `FLIPIC`, `IJMPDS` for a Flipper).

### Animation Mechanism

1.  **Inheritance:** When an enemy like a Flipper becomes a Chaser, it retains its original programmatic and procedural animation systems. Its appearance and method of "flipping" between lines do not change.
2.  **Trigger:** The `TOPPER` CAM script, which governs the Chaser's movement, uses the standard `VJUMPS` opcode to initiate a flip to an adjacent line on the rim.
3.  **Execution:** The `VJUMPS` opcode triggers the exact same procedural animation logic that the Flipper uses when it is in the well. For a Flipper-turned-Chaser, this means the `FLIPIC` and `IJMPDS` routines are called to calculate its interpolated 3D position between the source and destination lines on the rim, creating the familiar "flip" animation.

In essence, the animation of an Exis is identical to the animation of the enemy it once was. A Chaser is simply a Flipper (or other enemy) executing a different AI script in a different location, but its visual method of movement and transformation is unchanged. For a byte-for-byte reconstruction, one must simply apply the base enemy's existing animation logic (e.g., the "Flipper Flip") when the `TOPPER` CAM script initiates a move.

This documentation clarifies that the end-of-level challenge is not a unique boss fight, but a confrontation with the surviving enemies that have become significantly more dangerous by reaching the player's own territory. 