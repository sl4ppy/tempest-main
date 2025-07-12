# Entity: Player Bullet ("Player Charge")

This document provides a complete technical specification for the player's projectile, referred to in the source code as a "Player Charge."

## 1. Identification and Purpose

-   **Entity Name:** Player Bullet / Player Charge.
-   **Role and Purpose:** The player's primary offensive tool. Fired from the Player Ship, it travels down the well to destroy enemies and enemy lines.
-   **Asset References:**
    -   **Vector Shape Data:** `ALVROM.MAC`, label `PTCURS`. Notably, the bullet uses the *same shape as the player's ship*, scaled down to a small size. The `PTSPLA` data is not used for the standard player projectile.
    -   **Primary Logic:** `ALWELG.MAC` (routines: `FIREPC`, `MOVCHA`, `LIFECT`, `COLCHK`).
    -   **State Variables:** `ALCOMN.MAC` (within the `CHARGE` object array).
    -   **Drawing Logic:** `ALDIS2.MAC` (routine: `DSPCHG`).
    -   **Sound Effect (Launch):** `SLAUNC`.

## 2. Physical Attributes

-   **Size and Shape:** A small, scaled-down version of the player's "U"-shaped ship. The color is yellow (`PCHCOL`).
-   **Hitbox:** Collision is based on proximity to an enemy on the Z-axis (depth) and whether they are on the same or adjacent web lines. The collision range against different enemy types is defined in the `ENSIZE` table. The collision range against other charges is `CHACHA`.
-   **Origin Point:** The bullet's position is tracked from its center.
-   **Rendering Layer:** Drawn by `DSPCHG`. Bullets are rendered in front of the player ship but behind enemies and explosions.

## 3. Movement and Locomotion

-   **Movement Type:** Linear, constant velocity.
-   **Logic:** The `MOVCHA` routine in `ALWELG.MAC` updates the bullet's position each frame.
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

The bullet's constant velocity can be momentarily interrupted when it collides with an enemy line (`Spike`). This is a state-dependent modification to its procedural movement.

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

-   The only sound associated with the Player Bullet is the `SLAUNC` sound effect, which is played by the `FIREPC` routine upon successful firing.

## 9. Effects and Particles

-   The Player Bullet creates no particle effects. When it destroys an enemy, an explosion entity is created at the enemy's location.

## 10. Timing and Frame Data

-   **Lifetime:** The bullet's lifetime is determined by its velocity and the depth of the well. At `PCVELO = 9`, it takes approximately `(240 - 16) / 9 = ~25` frames to cross the entire well.
-   **Cooldown:** There is no cooldown on firing. The player can fire on any frame, provided one of the `NPCHARG` (8) bullet slots is inactive.

## 11. Parameter Tables

| Parameter | Variable/Constant | Value | Description |
|---|---|---|---|
| **Shape Data** | (in `DSPCHG`) | `PTCURS` | Pointer to the vector data for the Player Ship shape. |
| **Max On-Screen** | `NPCHARG` | 8 | Number of available slots for player bullets. |
| **Velocity** | `PCVELO` | 9 | Units of Y-depth traveled per frame. |
| **Line Hit Slowdown** | (in `MOVCHA`) | 4 | Amount velocity is reduced by for one frame on line hit. |
| **Max Line Hits** | (in `LIFECT`) | 2 | The value the `CHARCO` counter must reach to deactivate the bullet. |

The remaining sections are not applicable to this entity. 