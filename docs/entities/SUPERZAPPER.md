# "Entity": Superzapper Weapon

This document provides a complete technical specification for the Superzapper, which functions as a special player-activated weapon rather than a persistent entity.

## 1. Identification and Purpose

-   **Name:** Superzapper.
-   **Role and Purpose:** A limited-use "smart bomb" weapon that destroys all enemies on the screen. It is intended as a panic button for when the player is overwhelmed.
-   **Asset References:**
    -   **Primary Logic:** `ALWELG.MAC` (routines: `PROSUZ`, `INISUZ`, `KILENE`).
    -   **State Variables:** `ALCOMN.MAC` (`SUZCNT`, `SUZTIM`).
    -   **Visual Effect Logic:** `ALDIS2.MAC` (within the `DSPWEL` routine).
    -   **Unused Vector Shape:** The shape `DIARA2` in `ALVROM.MAC` appears to be an unused asset intended for the Superzapper. **No special graphic is actually drawn around the player when the weapon is activated.**

## 2. Physical Attributes

-   The Superzapper has no physical presence or hitbox. It is a screen-wide effect.

## 3. Movement and Locomotion

-   Not applicable.

## 4. Animation Data

-   The Superzapper's primary visual effect is not a distinct animation but a modification of the well's colors.
-   **Well Color Cycling:** While the Superzapper is active (`SUZTIM` > 0), the `DSPWEL` routine cycles the color of the well vectors each frame based on the value of the global frame counter (`QFRAME`). This creates a flashing, multi-colored "strobe" effect on the well itself.

## 4.1. Procedural Animation: Well Color Cycle

The Superzapper's visual "animation" is a procedural effect that modifies the color of the entire well structure, creating a powerful strobing effect. It does not draw any new shapes, but instead manipulates the color parameter of an existing entity (the Well).

-   **Animation Type:** Procedural Color Animation.
-   **Trigger:** The effect is active whenever the Superzapper timer (`SUZTIM`) is greater than zero. The logic is located inside the `DSPWEL` routine.

### Algorithm and Parameters

The algorithm is executed for every vector of the well on every frame the Superzapper is active.

1.  **Check for Active Superzapper:** The code checks if `SUZTIM` is positive.
2.  **Get Frame Counter:** If active, it loads the global 8-bit frame counter, `QFRAME`.
3.  **Calculate Color Index:** The `QFRAME` value is masked with a bitwise `AND I,7`. This constrains the value to a 0-7 range, corresponding to the 8 available colors in the hardware palette.
4.  **Prevent Black Color:** The result is compared to `7`. If the result is `7` (the index for black), it is changed to `1` (red). This ensures the well never disappears during the effect.
5.  **Set Well Color:** This final calculated color index is used to draw all the vectors that make up the well for that frame.

-   **Mathematical Formula:**
    -   `ColorIndex = QFRAME & 7`
    -   `If ColorIndex == 7, ColorIndex = 1`

-   **State Dependencies:**
    -   The animation is entirely dependent on the `SUZTIM` variable. Once `SUZTIM` decrements to zero, this logic is no longer executed, and the well reverts to its default color.
    -   The `QFRAME` variable is a continuously incrementing counter, meaning the color changes on every single frame, resulting in a rapid, predictable strobe sequence (Red, Green, Blue, etc.).

## 5. State Machine / Behavior Logic

The Superzapper's state is managed by the `SUZTIM` timer.

-   **State: Inactive**
    -   **Condition:** `SUZTIM == 0`.
    -   **Behavior:** The weapon is idle.
-   **State: Active**
    -   **Condition:** `SUZTIM > 0`. The weapon is activated by the `PROSUZ` routine.
    -   **Behavior:**
        1.  For the duration of the timer, the `DSPWEL` routine cycles the well colors.
        2.  The `KILENE` routine is called each frame.
    -   **Duration:** The active time is defined by the `TIMAX` table in `ALWELG.MAC`, indexed by the usage counter `SUZCNT`.
        -   First use in a wave: `CSUSTA + (8 * (CSUINT+1))` frames.
        -   Second use in a wave: `CSUSTA + (1 * (CSUINT+1))` frames.
        -   (`CSUSTA`=3, `CSUINT`=1). Duration is 19 frames for the first use, and 5 frames for the second.

## 6. Interaction and Input Handling

-   **Input:** The Superzapper is triggered when the player presses the associated button.
-   **Logic:** The `PROSUZ` routine checks for the `MSUZA` (`$08`) flag in the `SWFINA` (latched input) variable.
-   **Prerequisites:** The weapon can only be fired if:
    -   The player is alive (`CURSL2` is not negative).
    -   The Superzapper is not already active (`SUZTIM == 0`).
    -   The player has uses remaining for the current wave (`SUZCNT < CSUMAX`).

## 7. Combat / Ability Data

-   **Uses:** The player is granted `CSUMAX` (a value of 2) uses per wave. This is not a persistent stock; the usage counter (`SUZCNT`) is reset to 0 at the start of each wave by `INISUZ`.
-   **Effect:** The Superzapper destroys enemies sequentially, not simultaneously.
-   **Destruction Logic (`KILENE`):**
    -   On specific frames (timed by `SUZTIM` and `CSUINT`), the routine finds the first active enemy in the `INVADER` object list.
    -   It then calls the `INCISQ` routine for that single enemy. `INCISQ` is the standard routine to kill an invader and start its death explosion.
    -   This process repeats on subsequent timed frames until all enemies are destroyed. This creates the signature "chain reaction" of explosions across the screen.

## 8. Sound and Audio Integration

-   The Superzapper does not appear to have its own unique activation sound effect in the examined logic. The sounds produced are the standard enemy explosion sounds (`CIEXPL`) triggered by the `INCISQ` routine for each enemy destroyed.

## 9. Effects and Particles

-   The weapon's effect is the chain reaction of standard enemy explosions. It does not create its own unique particles.

## 10. Timing and Frame Data

-   See Section 5 for the duration of the active effect.
-   The timing of the sequential enemy destruction is controlled by the `CSUSTA` and `CSUINT` constants. An enemy is destroyed every `CSUINT+1` (2) frames, starting after `CSUSTA` (3) frames have passed.

## 11. Parameter Tables

| Parameter | Variable/Constant | Value | Description |
|---|---|---|---|
| **Max Uses Per Wave** | `CSUMAX` | 2 | Number of times the Superzapper can be used in a single wave. |
| **Usage Counter** | `SUZCNT` | 0, 1 | Tracks uses within a wave. Reset to 0 by `INISUZ`. |
| **Active Timer** | `SUZTIM` | > 0 | A frame counter that indicates the weapon is active. |
| **Duration (1st use)** | `TIMAX[1]` | 19 frames | Active duration for the first use in a wave. |
| **Duration (2nd use)** | `TIMAX[2]` | 5 frames | Active duration for the second use in a wave. |
| **Kill Timing Start** | `CSUSTA` | 3 | Number of frames before the first enemy is killed. |
| **Kill Timing Interval**| `CSUINT` | 1 | Controls the interval between enemy kills (interval is `CSUINT`+1 frames). |

The remaining sections are not applicable to this weapon system. 