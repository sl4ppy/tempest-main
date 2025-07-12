# Entity: Pulsar

This document provides a complete technical specification for the Pulsar enemy.

## 1. Identification and Purpose

-   **Entity Name:** Pulsar.
-   **Identifier:** Type `ZABPUL` (1) in the `INVADER` object array.
-   **Role and Purpose:** A dangerous, intelligent enemy that patrols a certain depth of the well. It periodically "pulses," creating a lethal energy field on its web line that can kill the player. It actively tries to move onto the player's line before pulsing.
-   **Asset References:**
    -   **Vector Shape Data:** `ALVROM.MAC`, labels `CPULS0` through `CPULS4`.
    -   **Primary Logic:** `ALWELG.MAC` (routines: `NEWPUL`, `JPULMO`, `PULSCH-CAM`).
    -   **Drawing Logic:** `ALDIS2.MAC` (routine: `PULPIC`).
    -   **State Variables:** `ALCOMN.MAC` (within `INVADER` array and global `PULSON`, `PULTIM` timers).

## 2. Physical Attributes

-   **Size and Shape:** The Pulsar is a geometric shape that animates through 5 distinct frames to create a "pulsing" or "expanding" effect.
-   **Hitbox:** Collision with a player shot is determined by the `ENSIZE` table for its type (`ZABPUL`). The pulsing energy field itself is a separate collision check.
-   **Color:** Turquoise (`TURQOI`) when idle, White when pulsing.
-   **Rendering Layer:** Drawn by `DSPINV` with other invaders.

## 3. Movement and Locomotion

-   **Movement Type:** A mix of linear patrol and targeted flipping, dictated by the `PULSCH-CAM` script.
-   **Base Movement (`JPULMO`):**
    -   The Pulsar moves up and down the well. Its speed is much faster than a Flipper's.
    -   It patrols a "power zone" defined by the `PULPOT` (Pulsar Potency) height. When it reaches this line, it reverses direction.
    -   If all Nymphs are gone (`NYMCOU`=0), the Pulsar will stop patrolling and head for the top of the well.
-   **Targeted Flipping (`PULSCH-CAM`):**
    -   The Pulsar's CAM script makes it an intelligent hunter.
    -   It waits for a delay (`PUCHDE`), moves, checks if a pulse is imminent (`VCHKPU`), then sets its flip direction towards the player (`VCHPLA`) and executes a flip (`VJUMPS`).
    -   This means it actively tries to get on the player's line before it pulses.

## 4. Animation Data

-   **Drawing Logic (`PULPIC`):** The `PULPIC` routine controls the Pulsar's animation.
-   **Animation Frames:** The global `PULSON` timer not only controls the pulse state but also the animation frame. Its value is used as an index into the `PULTAB` table.
-   **Frame Sequence:** `CPULS0` (smallest) -> `CPULS1` -> `CPULS2` -> `CPULS3` -> `CPULS4` (largest). This creates the visual effect of the Pulsar expanding.

## 4.1. Programmatic Animation: The Pulse

The Pulsar's signature "pulse" is a programmatic, state-driven animation that affects both its color and its shape simultaneously. The entire animation is controlled by a single global timer.

-   **Animation Type:** Programmatic, State-Driven Frame Sequence and Color Change.
-   **Trigger:** The animation is driven by the value of the global `PULSON` timer. It is updated every time a Pulsar is drawn by the `PULPIC` routine.
-   **Core Logic:** `PULPIC`, `PULTAB`.

### Algorithm

The `PULPIC` routine executes the following logic each time it renders a Pulsar:

1.  **Check Pulse State:** It checks if the `PULSON` timer is positive.
2.  **Set Color:**
    -   If `PULSON > 0`, the color is set to White (`WHITE`).
    -   If `PULSON == 0`, the color is set to Turquoise (`TURQOI`).
3.  **Calculate Frame Index:** The animation frame is also derived directly from the `PULSON` timer. The calculation is:
    -   `FrameIndex = (PULSON + 64) / 16`
    -   This is implemented in the code via an addition followed by four logical right bit-shifts (`LSR`).
4.  **Clamp Index:** The calculated `FrameIndex` is capped at a maximum value of 5.
5.  **Retrieve Shape Pointer:** The clamped index is used to look up a pointer in the `PULTAB` data table. This table contains the addresses of the 5 Pulsar vector shapes (`CPULS0` through `CPULS4`).
6.  **Render:** The selected shape is drawn in the selected color.

### Parameters and Timing

-   **`PULSON`:** A global 8-bit timer that counts down to zero. It is the sole driver of the animation state. When it is positive, the Pulsar is in its "active" or "pulsing" state.
-   **`PULTIM`:** A global timer that controls how frequently `PULSON` is reset, thereby controlling the frequency of the pulse animation across all Pulsars.
-   **`PULTAB`:** A data table containing five pointers to the `CPULS0` through `CPULS4` vector shapes.
-   **Animation Behavior:** As the `PULSON` timer counts down, the calculated frame index will decrease, causing the Pulsar to cycle backward through its animation frames (`CPULS4` -> `CPULS3` -> ... -> `CPULS0`). This creates the effect of the Pulsar visually contracting after its initial expansion. The color change is binary: it is white for the entire duration of the pulse and turquoise otherwise.

## 5. State Machine / Behavior Logic

-   **State: Spawning**
    -   Pulsars are created when a "Nymph" is converted via the `NEWPUL` routine.
    -   Its CAM is set to `PULSCH-CAM`.
-   **State: Patrolling & Hunting**
    -   The Pulsar executes its CAM script, moving up and down the well and periodically flipping towards the player.
-   **State: Pulsing**
    -   **Trigger:** The global `PULSON` timer becomes positive. The rate of this timer is controlled by `PULTIM`, which is set per wave.
    -   **Lethality:** While `PULSON` is positive, the `JPULMO` routine performs a collision check. If the Pulsar's depth is within the `PULPOT` zone and it is on the same line as the player, the player is killed (`INPPSQ`).
    -   **Visuals:** The `PULPIC` routine draws the Pulsar as white and cycles through its expanding animation frames.
-   **State: Firing**
    -   On later waves (`WPULFI` flag is set), a Pulsar can also fire standard enemy projectiles. This is separate from its pulsing attack.
-   **State: Reaching the Top (`CHASER` routine)**
    -   If a Pulsar reaches the top rim, unlike a Flipper, it does not become a permanent "Chaser."
    -   If Nymphs are still on screen, it simply reverses direction and goes back down the well.

## 7. Combat / Ability Data

-   **Primary Attack (Pulse):** A lethal energy field on its current web line. The player must move off the line to survive.
-   **Secondary Attack (Projectile):** Can fire a standard enemy projectile on waves where `WPULFI` is active.

## 11. Parameter Tables

| Parameter | Variable/Constant | Description |
|---|---|---|
| **Type ID**| `ZABPUL` | 1 | The internal identifier for a Pulsar. |
| **Velocity** | `WINVIL`/`WINVIN` | The speed at which the Pulsar moves. Set per wave for the `ZABPUL` type. |
| **CAM Script** | (Hardcoded) | `PULSCH-CAM` | The only behavior script used by the Pulsar. |
| **Pulse Timer** | `PULSON`/`PULTIM` | Global timers that control the pulsing state for all Pulsars. |
| **Pulse Height** | `PULPOT` | The Y-depth a Pulsar must be below for its pulse to be lethal. |
| **Chase Delay** | `PUCHDE` | A per-wave value controlling how long a Pulsar waits before flipping. |
| **Can Fire Flag** | `WPULFI` | A per-wave flag that determines if Pulsars can fire projectiles. |

The remaining sections are not applicable to this entity. 