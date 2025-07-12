# Entity: Player Ship ("Blaster" / "Cursor")

This document provides a complete technical specification for the player-controlled entity in Tempest, referred to in the source code as the "Cursor."

## 1. Identification and Purpose

-   **Entity Name:** Player Ship. Also known as the "Blaster" in promotional materials and the "Cursor" throughout the source code.
-   **Role and Purpose:** The player's avatar in the game. The player controls the Blaster to shoot enemies, evade projectiles, and survive waves. Its movement is restricted to the outer rim of the geometric "well."
-   **Asset References:**
    -   **Vector Shape Data:** `ALVROM.MAC`, label `PTCURS`.
    -   **Death Explosion Animation:** `ALVROM.MAC`, labels `SPLAT1` through `SPLAT6`.
    -   **Primary Logic:** `ALWELG.MAC` (routines: `INICUR`, `MOVCUR`, `FIREPC`, `DEADCU`).
    -   **State Variables:** `ALCOMN.MAC`.
    -   **Sound Effects:**
        -   Fire Shot: `SLAUNC`
        -   Movement: `SBOING`
        -   Death Explosion: `CPEXPL`
        -   Thrust (inter-level drop): `SOUTS2`

## 2. Physical Attributes

-   **Size and Shape:** The Blaster is a "U"-shaped vector object defined by the `PTCURS` data in `ALVROM.MAC`.
-   **Hitbox:** Collision detection is not based on a simple hitbox. It is based on the line segments the player occupies. An enemy or projectile on the same line segment (`CURSL1`) is considered a collision. For player shots hitting enemies, it's based on Y-depth and line number proximity.
-   **Origin Point:** The pivot point for the Blaster is the center of its base. Its position is defined by the line numbers it straddles.
-   **Rendering Layer:** The Blaster is drawn by the `DSPCUR` routine in `ALDIS2.MAC`. It is typically rendered on top of the well but behind projectiles and explosions. The exact draw order per frame is: Blaster -> Shots -> Invaders on Rim -> Explosions -> Well -> Invaders in Well -> Starfield.

## 3. Movement and Locomotion

### On the Well Rim

-   **Movement Type:** The Blaster's movement is controlled directly by the player's input from a rotary controller (spinner).
-   **Logic:** The `MOVCUR` routine in `ALWELG.MAC` is responsible for all player movement.
-   **Speed Values:**
    -   The raw spinner delta is read from the `TBHD` variable each frame.
    -   This value is clamped to a maximum of `+/- 31` units per frame.
-   **Position Calculation:**
    1.  A master fractional position variable, `CURSPO`, tracks the ship's location around the circular rim. This value is a single byte, effectively mapping the rim to 256 units.
    2.  The clamped spinner delta from `TBHD` is added to `CURSPO` each frame.
    3.  The primary line number, `CURSL1`, is calculated by taking the master position `CURSPO` and dividing it by 16 (via a 4-position right bit-shift). This maps the 256-unit circle to the 16 lines of the well.
    4.  The secondary line number, `CURSL2`, is always `(CURSL1 + 1) % 16`.

### Dropping Between Levels

-   **Movement Type:** Physics-based, automatic movement.
-   **Logic:** The `MOVCUD` routine in `ALWELG.MAC` handles the "drop" to the next level. This state is active when `CURMOD` is negative.
-   **Speed and Acceleration:**
    -   Movement is governed by a 16-bit velocity variable (`CURSVH`, `CURSVL`).
    -   The ship undergoes constant acceleration. Each frame, a value derived from the current wave number (`CURWAV`) plus a base value of 20 is added to the velocity.
    -   The maximum wave-based acceleration component is capped at 30.
    -   The ship's depth (`CURSY`, `CURSYL`) is updated by this velocity each frame.

## 4. Animation Data

-   **Animation States:** The Blaster itself has only one static shape (`PTCURS`). It does not have animation frames.
-   **Death Animation:** Upon death, a complex, multi-frame, multi-colored explosion animation is rendered at the Blaster's position.
    -   **Frame Sequence:** The `SPLAT1` through `SPLAT6` vector shapes are used.
    -   **Timing:** The `SPFTIM` variable acts as a timer to cycle through the explosion frames.

## 4.1. Procedural / Programmatic / Transform Animations

The Player Ship's appearance of motion and animation is achieved through three distinct programmatic systems: the transformation of its position along the well rim, a pre-defined sequence for its death explosion, and a physics-based algorithm for the drop between levels.

### A. Rim Movement: 3D Transform Animation

The smooth movement of the Blaster along the rim of the well is not a simple 2D translation. It is a procedural animation that transforms the player's 1D rotational input into a 3D world position, which is then projected onto the 2D screen.

-   **Animation Type:** Procedural 3D Transform.
-   **Algorithm:**
    1.  **Input:** The player's rotary controller input (`TBHD`) is read each frame.
    2.  **Position Update:** The input is used to update a master 8-bit fractional position variable, `CURSPO`, which maps the entire circumference of the well into 256 discrete units. The `MOVCUR` routine handles this.
    3.  **3D Coordinate Calculation:** The `CURSPO` value is used to determine the Blaster's position in 3D space.
        -   The primary column (`CURSL1`) is `CURSPO / 16`.
        -   The fractional position between columns is `CURSPO % 16`.
    4.  **Vector Shape Selection:** The game does not perform real-time rotation. Instead, to give the appearance of the Blaster conforming to the well's curve, the `DSPCUR` routine selects one of 8 pre-drawn, pre-rotated vector shapes. It calculates an index from 0-7 based on the fine-grained `CURSPO` value and adds it to the base vector shape pointer `CNCURS`.
    5.  **3D-to-2D Projection:** The `ONELIN` routine is called to render the selected vector shape at the calculated 3D position. It uses the `WORSCR` (World to Screen) utility, which leverages the hardware Math Box's `PROJEC` function to perform the perspective divide, transforming the object's (X, Y, Z) coordinates into 2D screen coordinates.

-   **Parameters:**
    -   **`CURSPO`:** 8-bit master position (0-255). Defines the 1D position on the unwrapped rim.
    -   **`PTCURS` / `CNCURS`:** The base address for the 8 pre-rotated Blaster vector shapes.
    -   **`LINEX`/`LINEZ` tables:** Provide the base 3D coordinates for each of the 16 well columns.

The final animation is an emergent effect of the game continuously re-calculating the Blaster's 3D world coordinates and projecting them to the screen, creating smooth motion that perfectly follows the geometry of the well.

### B. Death: Programmatic Animation Sequence

The player's death is a scripted, programmatic animation sequence, not a dynamic or physics-based one.

-   **Animation Type:** Programmatic, Frame-Sequence.
-   **Trigger:** The `DEADCU` routine is called when the player collides with an enemy, projectile, or Spike.
-   **Algorithm:**
    1.  **Initialization:** `DEADCU` calls `GENEX2` to create an explosion object of type `CPTYPE` (value `1`). It also initializes a dedicated timer, `SPFTIM`, to `1`.
    2.  **Frame Update:** On subsequent frames, the master `PROEXP` routine processes all active explosions. For a player explosion, it increments its internal sequence counter (`EXPLOS`) by the value in `TEXINC[1]`, which is `1`.
    3.  **Termination:** The animation continues until the `EXPLOS` counter reaches the value in `TEXPDN[1]`, which is `15`. At this point, the explosion object is deactivated.

-   **Parameters and Timing:**
    -   **`CPTYPE`:** `1`. The hardcoded type for a player explosion.
    -   **`TEXINC[1]`:** `1`. The value added to the animation sequence counter each frame.
    -   **`TEXPDN[1]`:** `15`. The value at which the animation sequence terminates.
    -   **Duration:** 15 frames.
    -   **Frame Data:** The sequence counter is used by the `DSPSPL` routine to select and draw one of the 6 `SPLAT` vector shapes, creating the visual effect of the explosion.

### C. Inter-Level Drop: Procedural Physics Animation

The movement between levels is a procedural animation driven by a simple physics simulation.

-   **Animation Type:** Procedural, Physics-based (Constant Acceleration).
-   **Trigger:** The `CURMOD` variable becomes negative, activating the `PLDROP` game state.
-   **Algorithm:**
    1.  The `MOVCUD` routine governs all movement during this state.
    2.  A 16-bit integer `CURSV` stores the player's downward velocity.
    3.  Each frame, an acceleration value is added to `CURSV`.
    4.  The player's Z-depth (`CURSY`) is updated by adding the new `CURSV`.
    5.  The player can still move left and right along the rim during the drop.

-   **Mathematical Formulas:**
    -   `Acceleration = BaseAcceleration + WaveBonus`
    -   `Velocity(t) = Velocity(t-1) + Acceleration`
    -   `Position(t) = Position(t-1) + Velocity(t)`

-   **Parameters:**
    -   **`BaseAcceleration`:** `20`
    -   **`WaveBonus`:** `CurrentWaveNumber * 4` (This bonus is capped at a maximum value of `30`).
    -   **`CURSV`:** 16-bit velocity accumulator.
    -   **`CURSY`:** 16-bit Z-depth position.

This system creates a smooth, non-linear acceleration, giving a compelling sense of falling into the next level.

## 5. State Machine / Behavior Logic

The player's state is primarily managed by the `CURSL2` and `CURMOD` variables.

-   **State: Alive**
    -   **Condition:** `CURSL2` has its most significant bit (bit 7) clear (i.e., value is < `$80`).
    -   **Behavior:** The player can move via `MOVCUR` and fire via `FIREPC`. The `PLAY` routine is the active game loop.
-   **State: Dropping**
    -   **Condition:** `CURMOD` is negative.
    -   **Behavior:** The player cannot fire but can still move left and right. The ship moves automatically down the Z-axis, handled by `MOVCUD`. The `PLDROP` routine is the active game loop. During this state, the player can collide with "enemy lines" (`LINEY`).
-   **State: Dead**
    -   **Condition:** `CURSL2` has its most significant bit set (value >= `$80`).
    -   **Behavior:** All player input is disabled. The death explosion animation plays. The `ANALYZ` routine is called to check for remaining lives and transition to the next state (new life or game over).
    -   **`CURSL2 = $81`:** A special "blasted" state set when killed by an enemy shot, which may have unique properties.

## 6. Interaction and Input Handling

-   **Input:**
    -   **Rotary Controller:** Read via `TBHD` for movement.
    -   **Fire Button:** Read from `SWSTAT` using the `MFIRE` mask (`$10`).
    -   **Superzapper Button:** Read from `SWSTAT` using the `MSUZA` mask (`$08`).
-   **Collision Responses:**
    -   **Enemy Shot Collision:** If an enemy shot (`CHARGE`) reaches the top of the well (`CURSY`) on the same line as the player (`CHARL1 == CURSL1`), the `CHATOP` routine calls `INCPSQ` to kill the player.
    -   **Enemy on Rim Collision:** The master `COLLIS` routine checks if an enemy object (e.g., Flipper) occupies the same line segment as the player. If so, the player is killed.
    -   **Enemy Line Collision (Drop Mode):** During the drop, `MOVCUD` checks if the player's `CURSY` depth passes through an active `LINEY` on the same `CURSL1` segment. If so, the player is killed.

## 7. Combat / Ability Data

-   **Attack:** The player's only standard attack is firing projectiles.
-   **Projectile Definition (Player Shot):**
    -   **Logic:** Handled by the `FIREPC` routine.
    -   **Instantiation:** A new entry is created in the `CHARGE` object table (indices 0-7 are for the player).
    -   **Velocity:** Shots have a constant Y-velocity of `PCVELO`, which is `9` units per frame.
    -   **Max on Screen:** A maximum of 8 player shots (`NPCHARG`) can be active at once.
-   **Superzapper:** A special weapon documented separately. It is triggered by the `MSUZA` input flag and handled by the `PROSUZ` routine. The player starts with 3 Superzappers (`INISUZ`).

## 8. Sound and Audio Integration

-   **Movement:** When `CURSL1` changes, `MOVCUR` calls `SBOING`.
-   **Fire Shot:** `FIREPC` calls `SLAUNC` when a shot is successfully fired.
-   **Death:** `DEADCU` calls `CPEXPL` to trigger the player explosion sound.
-   **Drop Mode:**
    -   `MOVCUD` calls `SOUTS2` to start a "rumble" or "thrust" sound when the drop begins.
    -   `MOVCUD` calls `PULSTO` to turn off the thrust sound if the player collides with a line during the drop.

## 9. Effects and Particles

-   The player entity itself does not emit particles.
-   The death sequence is a pre-defined vector animation (`SPLAT*`), not a particle system.

## 10. Timing and Frame Data

-   **Fire Rate:** The player can fire one shot per frame, as long as there is a free slot in the `CHARGE` table. There is no explicit fire rate timer or cooldown.
-   **Death Explosion:** The duration is controlled by `SPFTIM`, a simple frame counter.

## 11. Parameter Tables

All numeric values are hardcoded constants defined in `ALCOMN.MAC` or `ALWELG.MAC`.

| Parameter               | Variable/Constant | Value(s)                                   | Description                                       |
| ----------------------- | ----------------- | ------------------------------------------ | ------------------------------------------------- |
| **Initial Position**    | (in `INICUR`)     | `CURSL1`=14, `CURSL2`=15, `CURSY`=$10        | Starting position for a new life.                 |
| **Dead Flag**           | `CURSL2`          | Bit 7 set (e.g., `$80`, `$81`)             | Marks the player as dead.                         |
| **Drop Mode Flag**      | `CURMOD`          | Negative value                             | Puts the player in the "dropping" state.          |
| **Player Shot Velocity**| `PCVELO`          | 9                                          | Units of Y-depth the shot travels per frame.      |
| **Max Player Shots**    | `NPCHARG`         | 8                                          | Number of player projectile slots available.      |
| **Spinner Clamp**       | (in `MOVCUR`)     | +/- 31                                     | Max rotational speed per frame.                   |
| **Drop Base Accel.**    | (in `MOVCUD`)     | 20                                         | Base acceleration added to drop velocity.         |
| **Drop Wave Accel.**    | (in `MOVCUD`)     | `CURWAV` * 4 (capped at 30)                | Wave-dependent acceleration for the drop.         |

## 12. Inter-Entity Relationships

-   **Spawning:** The player entity is created at the start of a game and re-initialized on a new life via `INEWLI`. It does not spawn other entities directly, but its fire action creates `CHARGE` (player shot) objects.
-   **Parent/Child:** The player is a top-level entity and has no parent. Player shots can be considered children in that their initial state is derived from the player's.

The remaining sections (AI, Example Data) are not applicable as the Player Ship is fully controlled by the user and its data is stored in memory variables, not external files. 