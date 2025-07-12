# Tempest: Entity Reference

This document provides a central reference to the detailed, reconstruction-grade documentation for every entity in the Atari Tempest project. The goal of this documentation is to capture every detail necessary to perfectly reproduce each entity's behavior, movement, animation, and state logic.

---
---

# Entity: Player Ship ("Blaster" / "Cursor")

This document provides a complete technical specification for the player-controlled entity in Tempest, referred to in the source code as the "Cursor."

## 1. Identification and Purpose

-   **Entity Name:** Player Ship. Also known as the "Blaster" in promotional materials and the "Cursor" throughout the source code.
-   **Role and Purpose:** The player's avatar in the game. The player controls the Blaster to shoot enemies, evade projectiles, and survive waves. Its movement is restricted to the outer rim of the geometric "well."
-   **Asset References:**
    -   **Vector Shape Data:** [`ALVROM.MAC`](./ALVROM.md), label [`PTCURS`](./DATA_ASSETS.md#player-ship-lifeylife1).
    -   **Death Explosion Animation:** [`ALVROM.MAC`](./ALVROM.md), labels [`SPLAT1` through `SPLAT6`](./DATA_ASSETS.md#explosions-expl1---expl4-splat-shrap).
    -   **Primary Logic:** [`ALWELG.MAC`](./ALWELG.md) (routines: `INICUR`, `MOVCUR`, `FIREPC`, `DEADCU`).
    -   **State Variables:** [`ALCOMN.MAC`](./ALCOMN.md).
    -   **Sound Effects:** See [Sound Data](./DATA_ASSETS.md#sound-data).
        -   Fire Shot: `SLAUNC`
        -   Movement: `SBOING`
        -   Death Explosion: `CPEXPL`
        -   Thrust (inter-level drop): `SOUTS2`

## 2. Physical Attributes

-   **Size and Shape:** The Blaster is a "U"-shaped vector object defined by the `PTCURS` data in [`ALVROM.MAC`](./ALVROM.md).
-   **Hitbox:** Collision detection is not based on a simple hitbox. It is based on the line segments the player occupies. An enemy or projectile on the same line segment (`CURSL1`) is considered a collision. For [player shots](./ENTITIES.md#entity-player-bullet-player-charge) hitting enemies, it's based on Y-depth and line number proximity.
-   **Origin Point:** The pivot point for the Blaster is the center of its base. Its position is defined by the line numbers it straddles.
-   **Rendering Layer:** The Blaster is drawn by the `DSPCUR` routine in [`ALDIS2.MAC`](./ALDIS2.md). It is typically rendered on top of the well but behind projectiles and explosions. The exact draw order per frame is: Blaster -> [Shots](./ENTITIES.md#entity-player-bullet-player-charge) -> [Invaders on Rim](./ENTITIES.md#entity-the-exis-chaser-state) -> Explosions -> [Well](./PLAYFIELD.md) -> [Invaders in Well](./ENTITIES.md#4-common-enemy-ai-and-behavior) -> Starfield.

## 3. Movement and Locomotion

### On the Well Rim

-   **Movement Type:** The Blaster's movement is controlled directly by the player's input from a rotary controller (spinner).
-   **Logic:** The `MOVCUR` routine in [`ALWELG.MAC`](./ALWELG.md) is responsible for all player movement.
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
-   **Logic:** The `MOVCUD` routine in [`ALWELG.MAC`](./ALWELG.md) handles the "drop" to the next level. This state is active when `CURMOD` is negative.
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
    5.  **3D-to-2D Projection:** The `ONELIN` routine is called to render the selected vector shape at the calculated 3D position. It uses the `WORSCR` (World to Screen) utility, which leverages the hardware [Math Box's](./SYSTEMS.md#mathbox-co-processor) `PROJEC` function to perform the perspective divide, transforming the object's (X, Y, Z) coordinates into 2D screen coordinates.

-   **Parameters:**
    -   **`CURSPO`:** 8-bit master position (0-255). Defines the 1D position on the unwrapped rim.
    -   **`PTCURS` / `CNCURS`:** The base address for the 8 pre-rotated Blaster [vector shapes](./DATA_ASSETS.md#player-ship-lifeylife1).
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
-   **Trigger:** The `CURMOD` variable becomes negative, activating the [`PLDROP` game state](./GAME_STATE_FLOW.md#cendwav--cnewv2--end-of-wave--warp-transition).
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

## 5. State Machine / Behavior Logic

The player's state is primarily managed by the `CURSL2` and `CURMOD` variables.

-   **State: Alive**
    -   **Condition:** `CURSL2` has its most significant bit (bit 7) clear (i.e., value is < `$80`).
    -   **Behavior:** The player can move via `MOVCUR` and fire via `FIREPC`. The [`PLAY` routine](./GAME_STATE_FLOW.md#cplay--gameplay) is the active game loop.
-   **State: Dropping**
    -   **Condition:** `CURMOD` is negative.
    -   **Behavior:** The player cannot fire but can still move left and right. The ship moves automatically down the Z-axis, handled by `MOVCUD`. The `PLDROP` routine is the active game loop. During this state, the player can collide with "enemy lines" (`LINEY`).
-   **State: Dead**
    -   **Condition:** `CURSL2` has its most significant bit set (value >= `$80`).
    -   **Behavior:** All player input is disabled. The death explosion animation plays. The `ANALYZ` routine is called to check for remaining lives and transition to the next state ([new life](./GAME_STATE_FLOW.md#cnewli--new-life-transition) or [game over](./GAME_STATE_FLOW.md#cendga--game-over)).
    -   **`CURSL2 = $81`:** A special "blasted" state set when killed by an enemy shot, which may have unique properties.

## 6. Interaction and Input Handling

-   **Input:**
    -   **Rotary Controller:** Read via `TBHD` for movement.
    -   **Fire Button:** Read from `SWSTAT` using the `MFIRE` mask (`$10`).
    -   **Superzapper Button:** Read from `SWSTAT` using the `MSUZA` mask (`$08`).
-   **Collision Responses:**
    -   **Enemy Shot Collision:** If an [enemy shot](./ENTITIES.md#entity-enemy-bullet-invader-charge) (`CHARGE`) reaches the top of the well (`CURSY`) on the same line as the player (`CHARL1 == CURSL1`), the `CHATOP` routine calls `INCPSQ` to kill the player.
    -   **Enemy on Rim Collision:** The master `COLLIS` routine checks if an enemy object (e.g., [Flipper](./ENTITIES.md#entity-flipper)) occupies the same line segment as the player. If so, the player is killed.
    -   **Enemy Line Collision (Drop Mode):** During the drop, `MOVCUD` checks if the player's `CURSY` depth passes through an active `LINEY` on the same `CURSL1` segment. If so, the player is killed.

## 7. Combat / Ability Data

-   **Attack:** The player's only standard attack is firing projectiles.
-   **Projectile Definition (Player Shot):**
    -   **Logic:** Handled by the `FIREPC` routine.
    -   **Instantiation:** A new entry is created in the `CHARGE` object table (indices 0-7 are for the player).
    -   **Velocity:** Shots have a constant Y-velocity of `PCVELO`, which is `9` units per frame.
    -   **Max on Screen:** A maximum of 8 player shots (`NPCHARG`) can be active at once.
-   **Superzapper:** A special weapon documented separately as the [Superzapper Weapon](./ENTITIES.md#entity-superzapper-weapon). It is triggered by the `MSUZA` input flag and handled by the `PROSUZ` routine. The player starts with 3 Superzappers (`INISUZ`).

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

-   **Spawning:** The player entity is created at the start of a game and re-initialized on a new life via [`INEWLI`](./GAME_STATE_FLOW.md#cnewli--new-life-transition). It does not spawn other entities directly, but its fire action creates `CHARGE` ([player shot](./ENTITIES.md#entity-player-bullet-player-charge)) objects.
-   **Parent/Child:** The player is a top-level entity and has no parent. Player shots can be considered children in that their initial state is derived from the player's.

The remaining sections (AI, Example Data) are not applicable as the Player Ship is fully controlled by the user and its data is stored in memory variables, not external files.

---
---

# Entity: Player Bullet ("Player Charge")

This document provides a complete technical specification for the player's projectile, referred to in the source code as a "Player Charge."

## 1. Identification and Purpose

-   **Entity Name:** Player Bullet / Player Charge.
-   **Role and Purpose:** The player's primary offensive tool. Fired from the [Player Ship](./ENTITIES.md#entity-player-ship-blaster--cursor), it travels down the well to destroy enemies and enemy lines.
-   **Asset References:**
    -   **Vector Shape Data:** [`ALVROM.MAC`](./ALVROM.md), label `PTCURS`. Notably, the bullet uses the *same shape as the player's ship*, scaled down to a small size. The `PTSPLA` data is not used for the standard player projectile.
    -   **Primary Logic:** [`ALWELG.MAC`](./ALWELG.md) (routines: `FIREPC`, `MOVCHA`, `LIFECT`, `COLCHK`).
    -   **State Variables:** [`ALCOMN.MAC`](./ALCOMN.md) (within the `CHARGE` object array).
    -   **Drawing Logic:** [`ALDIS2.MAC`](./ALDIS2.md) (routine: `DSPCHG`).
    -   **Sound Effect (Launch):** [`SLAUNC`](./DATA_ASSETS.md#player-fire-la---launch).

## 2. Physical Attributes

-   **Size and Shape:** A small, scaled-down version of the player's "U"-shaped ship. The color is yellow (`PCHCOL`).
-   **Hitbox:** Collision is based on proximity to an enemy on the Z-axis (depth) and whether they are on the same or adjacent web lines. The collision range against different enemy types is defined in the `ENSIZE` table. The collision range against other charges is `CHACHA`.
-   **Origin Point:** The bullet's position is tracked from its center.
-   **Rendering Layer:** Drawn by `DSPCHG`. Bullets are rendered in front of the [player ship](./ENTITIES.md#entity-player-ship-blaster--cursor) but behind enemies and explosions.

## 3. Movement and Locomotion

-   **Movement Type:** Linear, constant velocity.
-   **Logic:** The `MOVCHA` routine in [`ALWELG.MAC`](./ALWELG.md) updates the bullet's position each frame.
-   **Speed Values:**
    -   The bullet travels at a constant velocity of `PCVELO`, which is `9` units of depth (Y-axis) per frame.
    -   If the bullet is determined to be in collision with an enemy line (`LIFECT` routine), its velocity is temporarily slowed by `4` units for that frame.

## 4. Animation Data

-   The Player Bullet has no animation states. It is a static, scaled shape.

## 4.1. Procedural Animation: Movement and Line Interaction

While the Player Bullet itself is a static vector shape, its movement is a key procedural element of the game, governed by a simple but effective algorithm.

-   **Animation Type:** Procedural, Constant Velocity Transform.
-   **Core Logic:** The `MOVCHA` routine updates the position of all active player bullets each frame.

### A. Constant Velocity Movement

The primary "animation" of the bullet is its continuous movement down the Z-axis of the well.

-   **Algorithm:**
    1.  On each frame, the bullet's current depth (`CHARY`) is increased by a fixed velocity value.
    2.  The new 3D position is then projected to the screen by the `DSPCHG` and `ONELIN` routines, creating the appearance of smooth travel away from the player.

-   **Mathematical Formula:**
    -   `Position(t) = Position(t-1) + Velocity`

-   **Parameters:**
    -   **`PCVELO` (Player Charge Velocity):** `9`. This is the constant value added to the bullet's `CHARY` position each frame. The bullet travels 9 units of depth per frame.

### B. State-Dependent Velocity Change (Line Collision)

The bullet's constant velocity can be momentarily interrupted when it collides with an enemy line ([Spike](./ENTITIES.md#entity-spiker--spike)). This is a state-dependent modification to its procedural movement.

-   **Trigger:** The `LIFECT` routine detects when a bullet's `CHARY` position matches a `LINEY` position on the same column.
-   **Algorithm:**
    1.  When a collision with a line is detected, the `MOVCHA` routine subtracts a slowdown value from the bullet's velocity *for that single frame*.
    2.  The bullet's `CHARCO` (collision counter) is incremented. If this counter reaches `2`, the bullet is deactivated.

-   **Parameters:**
    -   **`Line Hit Slowdown`:** `4`. The bullet's effective velocity on a frame where it hits a line is `9 - 4 = 5`.
    -   **`CHARCO` (Charge Collision Counter):** A per-bullet counter. After it hits two lines, it is removed.

## 5. State Machine / Behavior Logic

A Player Bullet is either `Active` or `Inactive`.

-   **State: Inactive**
    -   **Condition:** The `CHARY` (Y-position) for its slot in the `CHARGE` array is `0`.
    -   **Behavior:** The object is available to be used. It is not processed or drawn.
-   **State: Active**
    -   **Condition:** `CHARY` is non-zero.
    -   **Instantiation:** A bullet becomes active when the player fires (`FIREPC`). Its initial `CHARY`, `CHARL1`, and `CHARL2` are set to the player's current position. The `CHACOU` (player charge counter) is incremented.
    -   **Behavior:**
        1.  Its `CHARY` position is increased by `PCVELO` each frame.
        2.  It is checked for collisions with enemies and enemy lines.
    -   **Deactivation Conditions:**
        -   Reaches the bottom of the well (`CHARY` >= `ILINDDY`, which is `$F0`).
        -   Collides with an enemy (handled in `COLCHK`).
        -   Its collision counter (`CHARCO`) exceeds a threshold after hitting enemy lines. The bullet is deactivated by setting its `CHARY` value back to `0` and decrementing `CHACOU`.

## 6. Interaction and Input Handling

-   Player Bullets do not respond to any player input after being fired. They are "fire-and-forget" projectiles.

## 7. Combat / Ability Data

-   **Damage:** Player Bullets do not have a damage value. They destroy most enemies in a single hit.
-   **Interaction with Enemy Lines:**
    -   The `LIFECT` routine handles the bullet's interaction with the pulsing enemy lines on the surface of the well.
    -   When a bullet's `CHARY` position overlaps with a `LINEY` position, it "damages" the line, causing the line's `LINEY` value to be set to the bullet's position (effectively erasing it).
    -   Each time a bullet hits a line, its `CHARCO` (Charge Collision) counter is incremented.
    -   If `CHARCO` reaches `2`, the bullet is considered "exhausted" and is deactivated.

## 8. Sound and Audio Integration

-   The only sound associated with the Player Bullet is the `SLAUNC` [sound effect](./DATA_ASSETS.md#player-fire-la---launch), which is played by the `FIREPC` routine upon successful firing.

## 9. Effects and Particles

-   The Player Bullet creates no particle effects. When it destroys an enemy, an [explosion entity](./DATA_ASSETS.md#explosions-expl1---expl4-splat-shrap) is created at the enemy's location.

## 10. Timing and Frame Data

-   **Lifetime:** The bullet's lifetime is determined by its velocity and the depth of the well. At `PCVELO = 9`, it takes approximately `(240 - 16) / 9 = ~25` frames to cross the entire well.
-   **Cooldown:** There is no cooldown on firing. The player can fire on any frame, provided one of the `NPCHARG` (8) bullet slots is inactive.

## 11. Parameter Tables

| Parameter | Variable/Constant | Value | Description |
|---|---|---|---|
| **Shape Data** | (in `DSPCHG`) | [`PTCURS`](./DATA_ASSETS.md#player-ship-lifeylife1) | Pointer to the vector data for the Player Ship shape. |
| **Max On-Screen** | `NPCHARG` | 8 | Number of available slots for player bullets. |
| **Velocity** | `PCVELO` | 9 | Units of Y-depth traveled per frame. |
| **Line Hit Slowdown** | (in `MOVCHA`) | 4 | Amount velocity is reduced by for one frame on line hit. |
| **Max Line Hits** | (in `LIFECT`) | 2 | The value the `CHARCO` counter must reach to deactivate the bullet. |

The remaining sections are not applicable to this entity.

---
---

# "Entity": Superzapper Weapon

This document provides a complete technical specification for the Superzapper, which functions as a special player-activated weapon rather than a persistent entity.

## 1. Identification and Purpose

-   **Name:** Superzapper.
-   **Role and Purpose:** A limited-use "smart bomb" weapon that destroys all enemies on the screen. It is intended as a panic button for when the player is overwhelmed.
-   **Asset References:**
    -   **Primary Logic:** [`ALWELG.MAC`](./ALWELG.md) (routines: `PROSUZ`, `INISUZ`, `KILENE`).
    -   **State Variables:** [`ALCOMN.MAC`](./ALCOMN.md) (`SUZCNT`, `SUZTIM`).
    -   **Visual Effect Logic:** [`ALDIS2.MAC`](./ALDIS2.md) (within the `DSPWEL` routine).
    -   **Unused Vector Shape:** The shape [`DIARA2`](./DATA_ASSETS.md#player-shot-diara2) in [`ALVROM.MAC`](./ALVROM.md) appears to be an unused asset intended for the Superzapper. **No special graphic is actually drawn around the player when the weapon is activated.**

## 2. Physical Attributes

-   The Superzapper has no physical presence or hitbox. It is a screen-wide effect.

## 3. Movement and Locomotion

-   Not applicable.

## 4. Animation Data

-   The Superzapper's primary visual effect is not a distinct animation but a modification of the well's colors.
-   **Well Color Cycling:** While the Superzapper is active (`SUZTIM` > 0), the `DSPWEL` routine cycles the color of the [well](./PLAYFIELD.md) vectors each frame based on the value of the global frame counter (`QFRAME`). This creates a flashing, multi-colored "strobe" effect on the well itself.

## 4.1. Procedural Animation: Well Color Cycle

The Superzapper's visual "animation" is a procedural effect that modifies the color of the entire [well structure](./PLAYFIELD.md), creating a powerful strobing effect. It does not draw any new shapes, but instead manipulates the color parameter of an existing entity (the Well).

-   **Animation Type:** Procedural Color Animation.
-   **Trigger:** The effect is active whenever the Superzapper timer (`SUZTIM`) is greater than zero. The logic is located inside the `DSPWEL` routine.

### Algorithm and Parameters

The algorithm is executed for every vector of the well on every frame the Superzapper is active.

1.  **Check for Active Superzapper:** The code checks if `SUZTIM` is positive.
2.  **Get Frame Counter:** If active, it loads the global 8-bit frame counter, `QFRAME`.
3.  **Calculate Color Index:** The `QFRAME` value is masked with a bitwise `AND I,7`. This constrains the value to a 0-7 range, corresponding to the 8 available colors in the hardware palette.
4.  **Prevent Black Color:** The result is compared to `7`. If the result is `7` (the index for black), it is changed to `1` (red). This ensures the well never disappears during the effect.
5.  **Set Well Color:** This final calculated color index is used to draw all the vectors that make up the [well](./PLAYFIELD.md) for that frame.

-   **Mathematical Formula:**
    -   `ColorIndex = QFRAME & 7`
    -   `If ColorIndex == 7, ColorIndex = 1`

-   **State Dependencies:**
    -   The animation is entirely dependent on the `SUZTIM` variable. Once `SUZTIM` decrements to zero, this logic is no longer executed, and the [well](./PLAYFIELD.md) reverts to its default color.
    -   The `QFRAME` variable is a continuously incrementing counter, meaning the color changes on every single frame, resulting in a rapid, predictable strobe sequence (Red, Green, Blue, etc.).

## 5. State Machine / Behavior Logic

The Superzapper's state is managed by the `SUZTIM` timer. The active state corresponds to the [`CBOOM` game state](./GAME_STATE_FLOW.md#47-state-cboom-superzapper).

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

-   The weapon's effect is the chain reaction of standard [enemy explosions](./DATA_ASSETS.md#explosions-expl1---expl4-splat-shrap). It does not create its own unique particles.

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

---
---

# Entity: Flipper

This document provides a complete technical specification for the Flipper enemy.

## 1. Identification and Purpose

-   **Entity Name:** Flipper.
-   **Identifier:** Type `ZABFLI` (0) in the `INVADER` object array.
-   **Role and Purpose:** The most common and basic enemy in Tempest. Flippers continuously move up the well from the bottom towards the player, periodically "flipping" to adjacent web lines. When they reach the top rim, they become ["Chasers"](./ENTITIES.md#entity-the-exis-chaser-state) and move along the rim, attempting to collide with the player.
-   **Asset References:**
    -   **Vector Shape Data:** [`ALVROM.MAC`](./ALVROM.md), labels [`ENER11` to `ENER14`](./DATA_ASSETS.md#flipper-aka-ener11---ener14) (which are aliases for `CINVA1`).
    -   **Primary Logic:** [`ALWELG.MAC`](./ALWELG.md) (routines: `MOVINV`, various CAM scripts).
    -   **Drawing Logic:** [`ALDIS2.MAC`](./ALDIS2.md) (routine: `FLIPIC`).
    -   **State Variables:** [`ALCOMN.MAC`](./ALCOMN.mac) (within the `INVADER` object array).

## 2. Physical Attributes

-   **Size and Shape:** The Flipper is a two-pronged shape defined by `CINVA1`. It does not have different animation frames for its body.
-   **Hitbox:** Collision with a [player shot](./ENTITIES.md#entity-player-bullet-player-charge) is determined by the `ENSIZE` table for its type (`ZABFLI`). Collision with the player ship at the top of the well is determined by sharing the same line segment (`INVAL1` or `INVAL2` matching `CURSL1` or `CURSL2`).
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
4.  **Render at Interpolated Position:** The `ONELN2` routine is called, which takes these newly calculated 3D coordinates, performs the standard perspective projection and scaling via the `WORSCR` and `CASCAL` routines, and draws the Flipper's static `CINVA1` [vector shape](./DATA_ASSETS.md#flipper-aka-ener11---ener14).

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
    -   **Condition:** The Flipper's `INVAY` position reaches the top of the well (`CURSY`). The [`CHASER` routine](./ENTITIES.md#entity-the-exis-chaser-state) is called.
    -   **Behavior:** The Flipper's CAM script is changed to `TOPPER`. In this state, it no longer moves down the well. It continuously checks if it's on the same line as the player (`VKITST`) to cause a collision. It flips along the rim at a speed controlled by `WTTFRA` (Wave Top Flipper Rate).
-   **Firing:** A Flipper can fire an [enemy projectile](./ENTITIES.md#entity-enemy-bullet-invader-charge) if it is not currently flipping (`INVMOT`=0), its personal fire timer (`INVACT`) has expired, and a random check passes.

## 7. Combat / Ability Data

-   **Attack:** Can fire a standard [enemy projectile](./ENTITIES.md#entity-enemy-bullet-invader-charge).
-   **Projectile Definition:** Firing is handled by the `FIREIC` routine. See the [Enemy Bullet](./ENTITIES.md#entity-enemy-bullet-invader-charge) documentation for details.
-   **Death:** When hit by a player shot, the `INCISQ` routine is called, which deactivates the Flipper and creates a standard [explosion entity](./DATA_ASSETS.md#explosions-expl1---expl4-splat-shrap) at its position.

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

---
---

# Entity: Tanker

This document provides a complete technical specification for the Tanker enemy.

## 1. Identification and Purpose

-   **Entity Name:** Tanker.
-   **Identifier:** Type `ZABTAN` (2) in the `INVADER` object array.
-   **Role and Purpose:** A slow-moving, durable enemy that acts as a "carrier." Its primary threat is not its own attack, but the two [Flippers](./ENTITIES.md#entity-flipper) it releases upon destruction.
-   **Asset References:**
    -   **Vector Shape Data:** [`ALVROM.MAC`](./ALVROM.md), labels [`PTTANK`, `PTTANP`, `PTTANF`](./DATA_ASSETS.md#tanker-tankr-tankf-tankp).
    -   **Primary Logic:** [`ALWELG.MAC`](./ALWELG.md) (routines: `MOVINV`, `KILINV`, `NEWTAN`).
    -   **Drawing Logic:** [`ALDIS2.MAC`](./ALDIS2.md) (routine: `TANPIC`).
    -   **State Variables:** [`ALCOMN.MAC`](./ALCOMN.md) (within the `INVADER` object array).

## 2. Physical Attributes

-   **Size and Shape:** The Tanker is a large, diamond-like shape. Its appearance can vary slightly based on the contents it carries (see Section 4).
-   **Hitbox:** Collision with a [player shot](./ENTITIES.md#entity-player-bullet-player-charge) is determined by the `ENSIZE` table for its type (`ZABTAN`).
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
-   By default, Tankers are created carrying [Flippers](./ENTITIES.md#entity-flipper). The ability to carry other enemy types is determined by the `WWTAC` tables for later waves.

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
4.  **Retrieve Shape Pointer:** `TANTAB` contains a list of pointers to different [vector shape data](./DATA_ASSETS.md#tanker-tankr-tankf-tankp) (`PTTANK`, `PTTANP`, `PTTANF`). The appropriate pointer is retrieved.
5.  **Render:** The `SCAPIC` routine is called, which uses the retrieved pointer to draw the selected shape on the screen, handling the necessary 3D projection and scaling.

### Parameters and Data Tables

-   **`INVAC2` (bitmask `INVCAR`):** A state variable for each Tanker that holds its cargo type.
    -   `0` or `1`: Carrying [Flippers](./ENTITIES.md#entity-flipper).
    -   `2`: Carrying [Pulsars](./ENTITIES.md#entity-pulsar).
    -   `3`: Carrying [Fuses](./ENTITIES.md#entity-fuseball).
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
    -   A Tanker can fire a standard [enemy projectile](./ENTITIES.md#entity-enemy-bullet-invader-charge). Firing logic is identical to the Flipper: it can fire if not flipping (which it never does), its fire timer (`INVACT`) has expired, and a random check passes.
-   **State: Destroyed (Splitting)**
    -   **Trigger:** The Tanker is hit by a player shot. The `INCISQ` routine is called, which then calls `KILINV`.
    -   **Behavior (`KILINV`):**
        1.  The original Tanker entity is deactivated.
        2.  The code checks the `INVCAR` flag. Since the Tanker is a carrier, the splitting logic is executed.
        3.  The `ACTINV` (Activate Invader) routine is called **twice**.
        4.  Two new **[Flipper](./ENTITIES.md#entity-flipper)** entities are created and activated.
        5.  The new Flippers are positioned to the left and right of the Tanker's death location, inheriting its depth (`INVAY`). One is given the `ZROCCW` rotation flag to encourage them to split apart.

## 7. Combat / Ability Data

-   **Attack:** Can fire a standard [enemy projectile](./ENTITIES.md#entity-enemy-bullet-invader-charge).
-   **Death:** The Tanker's death action is its defining characteristic: it splits into two new [Flipper](./ENTITIES.md#entity-flipper) enemies.

## 11. Parameter Tables

| Parameter | Variable/Constant | Description |
|---|---|---|
| **Type ID** | `ZABTAN` | 2 | The internal identifier for a Tanker. |
| **Upward Velocity**| `WINVIL`/`WINVIN` | The speed at which the Tanker moves up the well. |
| **CAM Script** | (Hardcoded) | `NOJUMP-CAM`. | The only behavior script used by the Tanker. |
| **Carrier Flag** | `INVCAR` | Bits in `INVAC2` that identify the entity as a carrier. |
| **Contents** | `ZCARFL` | The value in the `INVCAR` bits indicating it carries Flippers. |

The remaining sections are not applicable to this entity.

---
---

# Entity: Spiker & Spike

This document provides a complete technical specification for the Spiker enemy and the "Spike" hazard it creates.

## 1. Identification and Purpose

-   **Entity Name:** Spiker. Referred to in the source code as "Trailer."
-   **Identifier:** Type `ZABTRA` (3) in the `INVADER` object array.
-   **Role and Purpose:** A specialized enemy that moves up and down a single line of the well. Its primary purpose is to create a "Spike," a persistent line on the web that is lethal to the player during the drop between levels.
-   **Asset References:**
    -   **Vector Shape Data:** [`ALVROM.MAC`](./ALVROM.md), labels [`PTSPI1` through `PTSPI1+6`](./DATA_ASSETS.md#spiker-spira1---spira4) (for its 4-frame animation).
    -   **Primary Logic:** [`ALWELG.MAC`](./ALWELG.md) (routines: `NEWSPI`, `JSTRAI`, `TRALUP-CAM`).
    -   **Drawing Logic:** [`ALDIS2.MAC`](./ALDIS2.md) (routine: `TRAPIC`).
    -   **State Variables:** [`ALCOMN.MAC`](./ALCOMN.md) (within `INVADER` and `LINEY` arrays).

## 2. Physical Attributes (Spiker)

-   **Size and Shape:** The Spiker is a long, thin, segmented object. It has a 4-frame animation sequence.
-   **Hitbox:** Collision with a [player shot](./ENTITIES.md#entity-player-bullet-player-charge) is determined by the `ENSIZE` table for its type (`ZABTRA`).
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
-   `TRAPIC` uses the global frame counter (`QFRAME`) to cycle through four [vector shapes](./DATA_ASSETS.md#spiker-spira1---spira4): `PTSPI1`, `PTSPI1+2`, `PTSPI1+4`, and `PTSPI1+6`. This gives the Spiker a constantly writhing appearance.

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

The creation of the Spike is a procedural animation that directly transforms the geometry of the [game world](./PLAYFIELD.md).

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
-   It can be destroyed by a single [player shot](./ENTITIES.md#entity-player-bullet-player-charge). When destroyed, it does not split or create other enemies.

## 11. Parameter Tables

| Parameter | Variable/Constant | Description |
|---|---|---|
| **Type ID** | `ZABTRA` | 3 | The internal identifier for a Spiker ("Trailer"). |
| **Upward/Downward Velocity** | `WINVIL`/`WINVIN` | The speed at which the Spiker moves. Set per wave for the `ZABTRA` type. |
| **CAM Script** | (Hardcoded) | `TRALUP-CAM`. | The only behavior script used by the Spiker. |
| **Max Height** | (Hardcoded) | `$20` | The Y-depth the Spiker reaches before reversing. |
| **Min Height** | (Hardcoded) | `$F2` | The Y-depth the Spiker reaches before finding a new line. |

The remaining sections are not applicable to this entity.

---
---

# Entity: Fuseball

This document provides a complete technical specification for the Fuseball enemy.

## 1. Identification and Purpose

-   **Entity Name:** Fuseball. Referred to in the source code as a "Fuse".
-   **Identifier:** Type `ZABFUS` (4) in the `INVADER` object array.
-   **Role and Purpose:** A highly dangerous and erratic enemy. The Fuseball moves rapidly up and down the well and will periodically attempt to flip directly into the player's current lane, making it a high-priority threat.
-   **Asset References:**
    -   **Vector Shape Data:** [`ALVROM.MAC`](./ALVROM.md), labels [`PTFUSE` through `PTFUSE+6`](./DATA_ASSETS.md#fuseball--pulsar-fuse0-fuse3-ener21-ener24) (for its 4-frame animation).
    -   **Primary Logic:** [`ALWELG.MAC`](./ALWELG.md) (routines: `NEWFUS`, `JFUSEUP`, `FUCHPL`, `LEFRIT`, `FUSEUP-CAM`).
    -   **Drawing Logic:** [`ALDIS2.MAC`](./ALDIS2.md) (routine: `FUSPIC`).
    -   **State Variables:** [`ALCOMN.MAC`](./ALCOMN.md) (within the `INVADER` object array).

## 2. Physical Attributes

-   **Size and Shape:** The Fuseball is a circular, spiky object. It has a 4-frame animation to give it a shimmering/rotating effect.
-   **Hitbox:** Collision with a [player shot](./ENTITIES.md#entity-player-bullet-player-charge) is determined by the `ENSIZE` table for its type (`ZABFUS`). A Fuseball is invulnerable to player shots while it is in the middle of a flip.
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
-   `FUSPIC` uses the global frame counter (`QFRAME`) to cycle through four [vector shapes](./DATA_ASSETS.md#fuseball--pulsar-fuse0-fuse3-ener21-ener24) based at `PTFUSE`.

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
-   When destroyed by a [player shot](./ENTITIES.md#entity-player-bullet-player-charge), it creates a unique, multi-stage explosion defined by `GEXIFU` and gives a higher point value.

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

---
---

# Entity: Pulsar

This document provides a complete technical specification for the Pulsar enemy.

## 1. Identification and Purpose

-   **Entity Name:** Pulsar.
-   **Identifier:** Type `ZABPUL` (1) in the `INVADER` object array.
-   **Role and Purpose:** A dangerous, intelligent enemy that patrols a certain depth of the well. It periodically "pulses," creating a lethal energy field on its web line that can kill the player. It actively tries to move onto the player's line before pulsing.
-   **Asset References:**
    -   **Vector Shape Data:** [`ALVROM.MAC`](./ALVROM.md), labels [`CPULS0` through `CPULS4`](./DATA_ASSETS.md#fuseball--pulsar-fuse0-fuse3-ener21-ener24).
    -   **Primary Logic:** [`ALWELG.MAC`](./ALWELG.md) (routines: `NEWPUL`, `JPULMO`, `PULSCH-CAM`).
    -   **Drawing Logic:** [`ALDIS2.MAC`](./ALDIS2.md) (routine: `PULPIC`).
    -   **State Variables:** [`ALCOMN.MAC`](./ALCOMN.md) (within `INVADER` array and global `PULSON`, `PULTIM` timers).

## 2. Physical Attributes

-   **Size and Shape:** The Pulsar is a geometric shape that animates through 5 distinct frames to create a "pulsing" or "expanding" effect.
-   **Hitbox:** Collision with a [player shot](./ENTITIES.md#entity-player-bullet-player-charge) is determined by the `ENSIZE` table for its type (`ZABPUL`). The pulsing energy field itself is a separate collision check.
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
-   **`PULTAB`:** A data table containing five pointers to the `CPULS0` through `CPULS4` [vector shapes](./DATA_ASSETS.md#fuseball--pulsar-fuse0-fuse3-ener21-ener24).
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
    -   On later waves (`WPULFI` flag is set), a Pulsar can also fire standard [enemy projectiles](./ENTITIES.md#entity-enemy-bullet-invader-charge). This is separate from its pulsing attack.
-   **State: Reaching the Top (`CHASER` routine)**
    -   If a Pulsar reaches the top rim, unlike a [Flipper](./ENTITIES.md#entity-flipper), it does not become a permanent "Chaser."
    -   If Nymphs are still on screen, it simply reverses direction and goes back down the well.

## 7. Combat / Ability Data

-   **Primary Attack (Pulse):** A lethal energy field on its current web line. The player must move off the line to survive.
-   **Secondary Attack (Projectile):** Can fire a standard [enemy projectile](./ENTITIES.md#entity-enemy-bullet-invader-charge) on waves where `WPULFI` is active.

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

---
---

# "Entity": The Exis (Chaser State)

This document clarifies the nature of the entity commonly referred to as the "Exis."

## 1. Identification and Purpose

-   **Name:** Exis / Chaser
-   **Role and Purpose:** Contrary to common belief, the "Exis" is not a unique enemy type within the game's code. The term refers to any standard enemy (typically a [Flipper](./ENTITIES.md#entity-flipper) or [Pulsar](./ENTITIES.md#entity-pulsar)) that has successfully traveled the entire length of the well and reached the top rim where the player resides. Upon reaching the rim, the enemy enters a "Chaser" state.

## 2. Behavior and Logic

-   **Trigger:** An enemy enters the Chaser state when its `INVAY` position reaches the player's `CURSY` position at the top of the well. The `CHASER` routine in [`ALWELG.MAC`](./ALWELG.md) is called.
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
-   **Pulsars:** Can also become Chasers, but their behavior is slightly different. If there are still Nymphs spawning, a [Pulsar](./ENTITIES.md#entity-pulsar) that reaches the top will simply reverse direction and go back down the well instead of entering the `TOPPER` CAM.

## 4. Procedural / Programmatic / Transform Animations

The "Exis" or "Chaser" state does not have its own unique animation logic. Instead, it reuses the animation capabilities of the base enemy that entered the state.

-   **Animation Type:** Inherited Procedural 3D Transform.
-   **Core Logic:** The `TOPPER` CAM script, `VJUMPS` and `VJUMPM` opcodes, and the animation routines of the base enemy (e.g., `FLIPIC`, `IJMPDS` for a Flipper).

### Animation Mechanism

1.  **Inheritance:** When an enemy like a [Flipper](./ENTITIES.md#entity-flipper) becomes a Chaser, it retains its original programmatic and procedural animation systems. Its appearance and method of "flipping" between lines do not change.
2.  **Trigger:** The `TOPPER` CAM script, which governs the Chaser's movement, uses the standard `VJUMPS` opcode to initiate a flip to an adjacent line on the rim.
3.  **Execution:** The `VJUMPS` opcode triggers the exact same procedural animation logic that the Flipper uses when it is in the well. For a Flipper-turned-Chaser, this means the `FLIPIC` and `IJMPDS` routines are called to calculate its interpolated 3D position between the source and destination lines on the rim, creating the familiar "flip" animation.

In essence, the animation of an Exis is identical to the animation of the enemy it once was. A Chaser is simply a Flipper (or other enemy) executing a different AI script in a different location, but its visual method of movement and transformation is unchanged. For a byte-for-byte reconstruction, one must simply apply the base enemy's existing animation logic (e.g., the "Flipper Flip") when the `TOPPER` CAM script initiates a move.

This documentation clarifies that the end-of-level challenge is not a unique boss fight, but a confrontation with the surviving enemies that have become significantly more dangerous by reaching the player's own territory.

---
---

# Entity: Enemy Bullet ("Invader Charge")

This document provides a complete technical specification for the enemy projectile, referred to in the source code as an "Invader Charge."

## 1. Identification and Purpose

-   **Entity Name:** Enemy Bullet / Invader Charge.
-   **Role and Purpose:** The primary ranged attack for several enemy types ([Flippers](./ENTITIES.md#entity-flipper), [Tankers](./ENTITIES.md#entity-tanker), some [Pulsars](./ENTITIES.md#entity-pulsar)). It travels from the enemy's position at the bottom of the well towards the player at the top.
-   **Asset References:**
    -   **Vector Shape Data:** [`ALVROM.MAC`](./ALVROM.md), label [`PTESHO`](./DATA_ASSETS.md#enemy-shots-eshot1-eshot4).
    -   **Primary Logic:** [`ALWELG.MAC`](./ALWELG.md) (routines: `FIREIC`, `MOVCHA`, `CHATOP`).
    -   **State Variables:** [`ALCOMN.MAC`](./ALCOMN.md) (within the `CHARGE` object array).
    -   **Drawing Logic:** [`ALDIS2.MAC`](./ALDIS2.md) (routine: `DSPCHG`).
    -   **Sound Effect (Launch):** `ESLSON`.

## 2. Physical Attributes

-   **Size and Shape:** A small shape defined by `PTESHO`. It has a 4-frame animation sequence.
-   **Hitbox:** An enemy bullet can be destroyed by a [player bullet](./ENTITIES.md#entity-player-bullet-player-charge). This collision is checked in the `COLCHK` routine. The range is defined by `CHACHA`.
-   **Color:** White (`ICHCOL`).
-   **Rendering Layer:** Drawn by `DSPCHG`. Bullets are rendered in front of the player ship but behind other enemies and explosions.

## 3. Movement and Locomotion

-   **Movement Type:** Linear, constant velocity.
-   **Logic:** The `MOVCHA` routine in [`ALWELG.MAC`](./ALWELG.md) updates the bullet's position each frame.
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
        2.  It is checked for collisions with the player and [player bullets](./ENTITIES.md#entity-player-bullet-player-charge).
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

-   The `ESLSON` ([Enemy Shot](./ENTITIES.md#entity-enemy-bullet-invader-charge) Launch Sound) is played by `FIREIC` when a shot is successfully fired.

## 11. Parameter Tables

| Parameter | Variable/Constant | Description |
|---|---|---|
| **Shape Data** | `PTESHO` | Pointer to the vector data for the Enemy Bullet shape. |
| **Max On-Screen** | `NICHARG` | 4 | Number of available slots for enemy bullets. |
| **Velocity** | `WCHARIN`/`WCHARL` | The speed at which the bullet travels towards the player. Set per wave. |
| **Fire Chance Table**| `CHANCE`| A table of probabilities used to throttle enemy firing rate. |

The remaining sections are not applicable to this entity. 