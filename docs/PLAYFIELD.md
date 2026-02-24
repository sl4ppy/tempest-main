# Tempest Playfield ("The Well")

This document provides a complete technical specification for the playfield in Tempest, referred to in the source code as the "Well" or "Tube." It details the geometry, procedural generation, rendering, and gameplay mechanics of the 3D space where the game takes place.

## 1. Overall Geometry and Coordinate System

The Tempest playfield is a 3D vector-based tube, or "well," viewed from one end with a perspective projection. The [player's ship](./ENTITIES.md#1-player-the-claw) is constrained to the outer rim closest to the camera, while [enemies](./ENTITIES.md#4-common-enemy-ai-and-behavior) emerge from the far end and travel "up" the well's walls towards the player.

### 1.1. Coordinate System

The game uses a left-handed 3D Cartesian coordinate system.

-   **Origin (0, 0, 0):** The exact center of the screen, which is also the vanishing point for the perspective projection.
-   **X-Axis:** Horizontal. Positive values are to the right, negative values are to the left.
-   **Z-Axis:** Vertical. Positive values are "up" (closer to the top of the physical screen), negative values are "down".
-   **Y-Axis:** Depth. This axis points "into" the screen.
    -   `Y = $10` (16): The position of the player's rim (the "near" plane).
    -   `Y = $F0` (240): The position of the well's far end (the "far" plane).
    -   Enemy and projectile positions are tracked along this axis.

### 1.2. Core Geometric Structure

The well is not a true mathematical tube but a 16-sided polygonal prism. It is defined by two 16-vertex rings: an outer ring (the near plane, or "rim") and an inner ring (the far plane, or "final shape"). The walls of the well are formed by connecting the corresponding vertices of these two rings.

-   **Segments ("Lines"):** The well is composed of 16 segments, referred to in the code as "lines" or "columns." Each segment is a 3D quadrilateral defined by two vertices on the near rim and two corresponding vertices on the far rim.
-   **Vertices:** The precise 3D coordinates for the vertices of these rings are stored in a set of master data tables.

## 2. Master Geometry Data Tables

The fundamental shapes of all possible wells in the game are defined by a set of static, hardcoded [vector shape](./DATA_ASSETS.md#2-vector-shapes) data tables located in `ALVROM.MAC`. The game engine selects from these master templates and copies the data into RAM to create the active playfield for a given wave.

The primary tables define the X and Z coordinates for each of the 16 vertices of a shape's inner and outer rings.

-   **`LINEX` / `LINEZ`:** These tables define the 16 vertices of the **outer ring** (the rim at the front of the well).
-   **`LIFSX` / `LIFSZ`:** These tables define the 16 vertices of the **inner ring** (the "final shape" at the back of the well).

### 2.1. Example Data: The Standard "Circle" Well

This is the default well shape, a 16-sided regular polygon.

**Outer Ring (`LINEX`, `LINEZ`):**
-   Defines a large 16-sided polygon centered at the origin.
-   Data from `ALVROM.MAC` at label `CIRCLN`:
```
; Table of X-coordinates for the vertices of the outer rim of a circular well.
LINEX:
    .WORD $36C0, $32C0, $25C0, $14C0, $0, -$14C0, -$25C0, -$32C0
    .WORD -$36C0, -$32C0, -$25C0, -$14C0, $0, $14C0, $25C0, $32C0
; Table of Z-coordinates for the vertices of the outer rim of a circular well.
LINEZ:
    .WORD $14C0, $25C0, $32C0, $36C0, $38C0, $36C0, $32C0, $25C0
    .WORD $14C0, -$25C0, -$32C0, -$36C0, -$38C0, -$36C0, -$32C0, -$25C0
```

**Inner Ring (`LIFSX`, `LIFSZ`):**
-   Defines a smaller 16-sided polygon, creating the perspective effect of a tube.
-   Data from `ALVROM.MAC` at label `CIRCLF`:
```
; Table of X-coordinates for the vertices of the inner ring of a circular well.
LIFSX:
    .WORD $1B60, $1960, $12E0, $A60, $0, -$A60, -$12E0, -$1960
    .WORD -$1B60, -$1960, -$12E0, -$A60, $0, $A60, $12E0, $1960
; Table of Z-coordinates for the vertices of the inner ring of a circular well.
LIFSZ:
    .WORD $A60, $12E0, $1960, $1B60, $1C60, $1B60, $1960, $12E0
    .WORD -$A60, -$12E0, -$1960, -$1B60, -$1C60, -$1B60, -$1960, -$A60
```

### 2.2. Other Master Shapes

`ALVROM.MAC` contains several other sets of these tables, defining other master shapes that are used in later waves:
-   **Square Well:** `SQUARN`, `SQUARF`
-   **"Plus" Shaped Well:** `PLUSN`, `PLUSF`
-   **Jagged / Star-Shaped Well:** `JAGGN`, `JAGGF`

The procedural generation system, detailed in the next section, is responsible for selecting which of these master templates to use for a given level.

## 3. Procedural Generation and Level Selection

The game does not generate well geometry from scratch using mathematical formulas in real-time. Instead, it employs a [table-driven procedural selection](./SYSTEMS.md#7-procedural-generation-engine) and modification system to construct the playfield for each wave. The process is orchestrated by the `CONTOUR` routine in `ALWELG.MAC`, which uses a master definition table in `ALVROM.MAC`.

### 3.1. The Master Well Table (`WELTAB`)

Located in `ALVROM.MAC`, `WELTAB` is the central directory for all well shapes in the game. It is a list of records, where each record defines the properties for a range of waves.

-   **Structure of a `WELTAB` Record:**
    -   **Start Wave:** The first wave this definition applies to.
    -   **End Wave:** The last wave this definition applies to.
    -   **Outer Ring Pointer:** The memory address of the `LINEX`/`LINEZ` vertex data for the near plane.
    -   **Inner Ring Pointer:** The memory address of the `LIFSX`/`LIFSZ` vertex data for the far plane.
    -   **Well Type ID (`WELTYP`):** A byte containing flags that define the well's properties.
        -   Bit 0: **Closed Well**. If set, the well is a continuous tube where line 15 connects back to line 0.
        -   Bit 1: **Planar Well**. If set, the well is a flat surface with only 15 segments. The connection between the last and first vertex is not drawn.

-   **Example `WELTAB` Entry:**
```assembly
; Wave 1-2: Standard closed circle
.BYTE 1,2, CIRCLN,CIRCLF, 1
; Wave 3-4: Standard closed square
.BYTE 3,4, SQUARN,SQUARF, 1
; Wave 5-6: Planar (open) circle
.BYTE 5,6, CIRCLN,CIRCLF, 2
; ... and so on for all waves.
```

### 3.2. The `CONTOUR` Routine

The `CONTOUR` routine in `ALWELG.MAC` is called at the start of each new wave to configure the playfield.

-   **Algorithm:**
    1.  **Get Current Wave:** The routine reads the current wave number (`CURWAV`).
    2.  **Find Matching Record:** It iterates through `WELTAB` until it finds the record whose wave range includes the current wave number.
    3.  **Set Pointers:** It retrieves the four pointers to the selected master vertex data tables (`LINEX`, `LINEZ`, `LIFSX`, `LIFSZ`) from the matched `WELTAB` record.
    4.  **Copy and Modify Data:** It then copies the 16 pairs of X/Z coordinates from the master tables in ROM into the active well geometry tables in RAM.
    5.  **Apply Properties:** It reads the `WELTYP` flag and applies the properties. For example, if the "Closed" bit is not set, it will treat the shape as open, which affects rendering and enemy pathfinding.

This system allows the game to feature a wide variety of playfield shapes by simply selecting and slightly modifying a small set of pre-defined, high-quality [vector shapes](./DATA_ASSETS.md#2-vector-shapes), which is far more efficient than generating complex geometry from scratch on 1980s hardware.

## 4. Visual Representation and Rendering

The well's geometric data is translated into on-screen vectors through a rendering pipeline that handles 3D-to-2D projection and dynamic color assignment. This process is managed by routines in `ALDIS2.MAC`.

### 4.1. The Rendering Pipeline

The core routines responsible for drawing the well are `DSPWEL` and `BLDWEL`.

1.  **`DSPWEL` (Display Well):** This is the main entry point, called once per frame. It checks a flag (`ROTDIS`) to see if the well's shape or position has changed. If so, it calls `BLDWEL` to create a new, updated display list for the well's vectors. Otherwise, it uses the previously generated list.
2.  **`BLDWEL` (Build Well):** This routine is the primary constructor. It reads the active vertex data from the `LINEX`/`Z` and `LIFSX`/`Z` RAM tables and builds a list of vectors for the [hardware](./SYSTEMS.md#1-vector-graphics-engine) to draw.
3.  **Drawing Order:** The well is drawn as a series of individual lines in a specific order to construct the tube shape:
    -   First, it draws the 16 "spokes"—the long lines connecting each vertex of the outer rim to the corresponding vertex on the inner rim.
    -   Second, it draws the "rungs"—the short lines connecting adjacent vertices on the outer rim, and then the lines connecting adjacent vertices on the inner rim, forming the two 16-sided polygons.

### 4.2. 3D Perspective Projection

Tempest creates its convincing 3D effect using a true perspective projection, handled by the Atari ["Math Box"](./SYSTEMS.md#2-mathbox-co-processor) coprocessor.

-   **`WORSCR` (World to Screen):** Before a vector is drawn, its 3D world coordinates (X, Y, Z) are passed to the `WORSCR` routine.
-   **`PROJEC` (Project):** `WORSCR` uses the `PROJEC` function of the Math Box. This hardware function performs the perspective division calculation, which scales the X and Z coordinates based on the Y (depth) coordinate. This means objects (or vertices) farther "down the well" (with a larger Y value) are drawn smaller and closer to the center vanishing point, creating the illusion of depth.

### 4.3. Color and Effects

The color of the well is dynamic and can be affected by gameplay events.

-   **Base Color:** The default color of the well is defined by the `WELCOL` variable.
-   **Superzapper Effect:** When the [Superzapper](./ENTITIES.md#3-superzapper) is active (`SUZTIM > 0`), the `DSPWEL` routine overrides the base color. It uses the global frame counter (`QFRAME`) to rapidly cycle through the hardware color palette, creating the signature rainbow strobing effect.
-   **Bonus Flash:** When the player earns a bonus, the `BOFLASH` timer is activated, causing the well to flash a specific color for a short duration.
-   **Player "Flashlight":** The two segments of the well immediately under the player's ship are always drawn in the player's color (`CURCOL`), creating a "flashlight" effect that helps the player track their position.

## 5. Playfield Animations and Transitions

The playfield itself is not static; it animates during the transition between waves, creating the signature "warp" effect. This is a procedural animation controlled by the `NEWAV2` routine in `ALWELG.MAC`.

-   **Animation Type:** Procedural 3D Transform (Camera and Geometry).
-   **Trigger:** The [`CNEWV2` (Wave Transition)](./GAME_STATE_FLOW.md#5-cnewv2--cwavdon---wave-transition-and-bonus) state is entered after a wave is cleared, initiating the animation sequence.

The transition combines two simultaneous effects: moving the camera forward along the depth axis and shifting the center of the far plane.

### 5.1. Camera Movement (Z-Axis Push)

This effect creates the sensation of flying down the tube into the next level.

-   **Algorithm:**
    1.  The `NEWAV2` routine is called on every frame during the transition.
    2.  Each frame, it adds a constant value (`$18`) to the 16-bit `EYL`/`EYH` variable, which stores the camera's Y-coordinate (depth).
    3.  This moves the camera's viewpoint forward at a constant velocity.
    4.  The animation continues until the camera's position (`EYL`) reaches a pre-defined destination depth (`EYLDES`).
    5.  Once the destination is reached, the animation stops, and the `QSTATE` is changed to [`CPLAY`](./GAME_STATE_FLOW.md#4-cplay---gameplay) to begin the next level.

-   **Parameters:**
    -   **`EYL`/`EYH`:** The 16-bit world-space Y-coordinate of the camera.
    -   **`EYLDES`:** The target Y-coordinate for the end of the transition.
    -   **Velocity:** A constant `$18` (24) units of depth per frame.

### 5.2. Well Morphing (Vanishing Point Animation)

While the camera pushes forward, the well itself appears to twist and morph. This is achieved by animating the Z-axis center of the far plane.

-   **Algorithm:**
    1.  At the start of the transition, a target Z-coordinate for the vanishing point is selected (`ZADEST`).
    2.  Each frame, the `NEWAV2` routine calculates an incremental step to move the current Z-center (`ZADJL`) towards the target `ZADEST`.
    3.  This interpolation causes the entire far end of the well to drift across the screen during the camera push.
    4.  Because the `BLDWEL` routine uses this `ZADJL` value as the origin for rendering the far ring, the entire well appears to bend and warp.

This combination of a constant-velocity camera push and a simultaneous, programmatic morphing of the underlying geometry creates the game's distinctive and memorable transition effect.

### 5.3. NEWAV2 Algorithm Detail (from `ALWELG.MAC`, lines 56-121)

The exact frame-by-frame algorithm:

```
Each frame during wave transition:
  1. CURSY = ILINLIY                      (reset cursor to rim)
  2. Z_increment = ZADEST >> 2            (arithmetic shift right 2, signed)
  3. ZADJL += Z_increment                 (16-bit signed add to vanishing point)
  4. EYL += $18                           (add 24 to eye position, 16-bit)
  5. IF EYH >= $FC:
       PLAGRO = 1                         (disable star field)
  6. delta = EYL - EYLDES                 (check if arrived)
  7. IF delta <= 0:                       (past destination)
       EYL = EYLDES                       (snap to destination)
       EYH = $FF
       IF attract_mode:
         QSTATE = CENDGA                  (end attract game)
       ELSE:
         QSTATE = CPLAY                   (begin wave)
       ROTDIS = $FF                       (request well redraw)
       JMP MOVCUR                         (update cursor)
```

**Key constants:**
- Eye Z speed: `$18` (24 units/frame, constant velocity)
- Star field cutoff: `EYH >= $FC`
- Z adjustment divisor: 4 (2-bit right shift)
- `ZADEST` and `EYLDES` are set per-shape from `HOLZAD`/`HOLZDH` and `HOLEYL` tables (see Appendix A)

## 6. Collision Boundaries and Gameplay Integration

The playfield's geometry is not just visual; it defines the boundaries for all gameplay mechanics, including movement and collision.

### 6.1. Entity Movement and Constraints

-   **Player Movement:** The player's ship (`Cursor`) is constrained to move only along the vertices of the outer rim (the "near" plane). Its position is tracked as a fractional value along this 16-sided path.
-   **Enemy Movement:** [Enemies](./ENTITIES.md#4-common-enemy-ai-and-behavior) are constrained to move along the 16 "spokes" or "lines" of the well. An enemy's position is defined by its line number (0-15) and its depth (`INVAY`) along that line. Enemies can only move between lines by executing a "flip" or "jump" maneuver, which is a scripted transition from one line to the next.

### 6.2. The Spike Collision System

The most significant interaction with the playfield's geometry is the "Spike" hazard created by the [Spiker](./ENTITIES.md#7-spiker) enemy. This is a procedural modification of the well's collision boundaries, managed by the [Spike System](./SYSTEMS.md#8-spike-system).

-   **The `LINEY` Array:** This array of 16 bytes is the definitive source for the collision depth of the far end of the well for each of the 16 lines. By default, all entries are set to `$F0`, the bottom of the well.
-   **Spike Creation (`JSTRAI` routine):**
    -   When a [Spiker](./ENTITIES.md#7-spiker) enemy moves, it continuously writes its own Y-depth (`INVAY`) into the `LINEY` array entry corresponding to the line it is on.
    -   `LINEY[Spiker_Line] = Spiker_Y_Position`
    -   This has the effect of dynamically "pulling" the collision boundary of that specific well segment up from the bottom of the well to the Spiker's current position.
-   **Player Collision (`MOVCUD` routine):**
    -   During the inter-level drop, the game checks the player's collision status every frame.
    -   It gets the player's current line number (`CURSL1`) and reads the corresponding value from the `LINEY` array.
    -   It checks if the player's depth (`CURSY`) is greater than or equal to the value retrieved from `LINEY`.
    -   `IF Player_Y_Position >= LINEY[Player_Line]`
    -   If this condition is true, a collision is registered, and the player is destroyed. This makes the Spikes a lethal, invisible wall during the drop.

The [Spike System](./SYSTEMS.md#8-spike-system) is a highly efficient and clever piece of procedural design, creating a dynamic environmental hazard by modifying a single data value in an array that defines the playfield's core collision geometry.

---

## Appendix A: Complete Well Shape Coordinate Data

All 16 well shapes extracted from `ALDISP.MAC` (lines 1237-1371). Values are unsigned bytes where `$80` (128) represents the center origin. The shapes define the outer rim (near plane) vertices for 16 lines.

### LINEX (X Coordinates, 16 vertices per shape)

All values in hexadecimal.

| Shape | 0 | 1 | 2 | 3 | 4 | 5 | 6 | 7 | 8 | 9 | A | B | C | D | E | F |
|-------|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|
| Circle | F0 | E7 | CF | AA | 80 | 56 | 31 | 19 | 10 | 19 | 31 | 56 | 80 | AA | CF | E7 |
| Square | F0 | F0 | F0 | B8 | 80 | 48 | 10 | 10 | 10 | 10 | 10 | 48 | 80 | B8 | F0 | F0 |
| Cross | F0 | F0 | B8 | B8 | 80 | 48 | 48 | 10 | 10 | 48 | 48 | 80 | B8 | B8 | F0 | F0 |
| Peanut | EC | D5 | B1 | 90 | 70 | 4F | 2B | 14 | 14 | 2B | 4F | 90 | B1 | D5 | EC | EC |
| Key | F0 | C0 | A0 | 94 | 6C | 60 | 40 | 10 | 10 | 40 | 60 | 6C | 94 | A0 | C0 | F0 |
| Triangle | D9 | C2 | AC | 97 | 80 | 69 | 52 | 3C | 27 | 10 | 35 | 5A | 80 | A6 | CA | F0 |
| Clover | EA | E0 | 9C | 80 | 64 | 20 | 16 | 50 | 16 | 20 | 64 | 80 | 9C | E0 | EA | B0 |
| V | 10 | 1E | 2C | 3A | 48 | 56 | 64 | 70 | 90 | 9E | AC | BA | C8 | D6 | E4 | F0 |
| Stairs | 10 | 10 | 30 | 30 | 50 | 50 | 70 | 70 | 90 | 90 | B0 | B0 | D0 | D0 | F0 | F0 |
| U | 10 | 10 | 10 | 10 | 16 | 29 | 46 | 69 | 97 | BA | D7 | EA | F0 | F0 | F0 | F0 |
| Flat | 10 | 1E | 2D | 3C | 4B | 5A | 69 | 78 | 87 | 96 | A5 | B4 | C3 | D2 | E1 | F0 |
| Heart | DA | A4 | 87 | 80 | 79 | 5C | 26 | 10 | 10 | 20 | 48 | 80 | B8 | E0 | F0 | F0 |
| Star | B0 | 80 | 50 | 47 | 18 | 30 | 18 | 47 | 50 | 80 | B0 | B9 | E8 | D4 | E8 | B9 |
| Waves | 10 | 1E | 21 | 28 | 3C | 55 | 66 | 73 | 8D | 9A | AB | C4 | D8 | DF | E2 | F0 |
| Jagged | 10 | 24 | 30 | 36 | 3E | 49 | 5A | 75 | 94 | A4 | AC | BA | DA | E2 | EA | F0 |
| 8-shape | 80 | 70 | 48 | 20 | 10 | 20 | 48 | 70 | 80 | 90 | B8 | E0 | F0 | E0 | B8 | 90 |

### LINEZ (Z Coordinates, 16 vertices per shape)

| Shape | 0 | 1 | 2 | 3 | 4 | 5 | 6 | 7 | 8 | 9 | A | B | C | D | E | F |
|-------|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|
| Circle | 80 | AA | CF | E7 | F0 | E7 | CF | AA | 80 | 56 | 31 | 19 | 10 | 19 | 31 | 56 |
| Square | 80 | 80 | 80 | B8 | F0 | B8 | 80 | 80 | 80 | 10 | 10 | 10 | 48 | 80 | B8 | F0 |
| Cross | 80 | B8 | B8 | F0 | F0 | B8 | B8 | 80 | 80 | 48 | 48 | 10 | 10 | 48 | 48 | 80 |
| Peanut | 94 | B0 | B8 | A7 | A7 | B8 | B0 | 94 | 6C | 50 | 48 | 59 | 59 | 48 | 50 | 6C |
| Key | 96 | A3 | C5 | F0 | F0 | C5 | A3 | 96 | 6A | 5D | 3B | 10 | 10 | 3B | 5D | 6A |
| Triangle | 3D | 6A | 97 | C4 | F0 | C4 | 97 | 6A | 3D | 10 | 10 | 10 | 10 | 10 | 10 | 10 |
| Clover | A0 | E0 | EA | B0 | EA | E0 | A0 | 80 | 60 | 20 | 16 | 50 | 16 | 20 | 60 | 80 |
| V | F0 | D0 | B0 | 90 | 70 | 50 | 30 | 10 | 10 | 30 | 50 | 70 | 90 | B0 | D0 | F0 |
| Flat | 40 | 40 | 40 | 40 | 40 | 40 | 40 | 40 | 40 | 40 | 40 | 40 | 40 | 40 | 40 | 40 |
| U | F0 | CB | A6 | 80 | 5C | 39 | 20 | 12 | 12 | 20 | 39 | 5C | 80 | A6 | CB | F0 |
| Jagged | C0 | A6 | 8A | 6A | 4A | 2F | 14 | 24 | 20 | 39 | 59 | 75 | 72 | 90 | B0 | D0 |
| 8-shape | 80 | 57 | 48 | 57 | 80 | A9 | BA | A9 | 80 | 57 | 48 | 57 | 80 | A9 | BA | A9 |
| Heart | E4 | E8 | B7 | 80 | B7 | E8 | E4 | B2 | 7A | 47 | 20 | 10 | 20 | 47 | 7A | B2 |
| Stairs | 90 | 70 | 70 | 50 | 50 | 30 | 30 | 10 | 10 | 30 | 30 | 50 | 50 | 70 | 70 | 90 |
| Star | E6 | D0 | E6 | B9 | AE | 80 | 52 | 47 | 14 | 30 | 14 | 47 | 52 | 80 | AE | B9 |
| Waves | 7E | 6A | 51 | 3A | 2C | 2C | 38 | 4E | 4E | 38 | 2C | 2C | 3A | 51 | 6A | 7E |

### Well Shape Sequence

The `WELSEQ` table (`ALDISP.MAC`, line 1375) maps the wave progression order to shape indices:

```
WELSEQ: 0, 1, 2, 3, 4, 5, 6, 7, 0D, 9, 8, 0C, 0E, 0F, 0A, 0B
```

| Wave mod 16 | Index | Shape |
|-------------|-------|-------|
| 0 | 0 | Circle |
| 1 | 1 | Square |
| 2 | 2 | Cross |
| 3 | 3 | Peanut |
| 4 | 4 | Key |
| 5 | 5 | Triangle |
| 6 | 6 | Clover |
| 7 | 7 | V |
| 8 | D | Star |
| 9 | 9 | U |
| 10 | 8 | Stairs |
| 11 | C | Heart |
| 12 | E | Jagged |
| 13 | F | Waves |
| 14 | A | Flat |
| 15 | B | 8-shape |

### Per-Shape Camera and Rendering Parameters

From `ALDISP.MAC`, lines 1376-1383:

| Shape | HOLEYL (Eye Y) | HOLEZL (Eye Z) | HOLZAD (Z Adjust) | HOLZDH (Z Adj Hi) | HOLRAP (Planar?) | WELLIS (Lin.Scale) | WELBIN (Bin.Scale) |
|-------|-------|-------|--------|--------|--------|---------|---------|
| Circle | $18 | $50 | $40 | $FF | 0 (closed) | $00 | 4 |
| Square | $1C | $50 | $20 | $FF | 0 (closed) | $00 | 4 |
| Cross | $18 | $50 | $40 | $FF | 0 (closed) | $60 | 3 |
| Peanut | $0F | $68 | $80 | $FF | 0 (closed) | $40 | 4 |
| Key | $18 | $50 | $40 | $FF | 0 (closed) | $00 | 4 |
| Triangle | $18 | $50 | $40 | $FF | 0 (closed) | $00 | 4 |
| Clover | $18 | $68 | $70 | $FF | 0 (closed) | $48 | 3 |
| V | $18 | $B0 | $60 | $00 | -1 (planar) | $40 | 4 |
| Stairs | $0A | $A0 | $00 | $01 | -1 (planar) | $50 | 5 |
| U | $18 | $50 | $20 | $FF | -1 (planar) | $28 | 4 |
| Flat | $10 | $90 | $40 | $00 | -1 (planar) | $50 | 4 |
| Heart | $0F | $80 | $00 | $00 | 0 (closed) | $00 | 4 |
| Star | $18 | $20 | $A0 | $FE | 0 (closed) | $00 | 4 |
| Waves | $0C | $B0 | $40 | $01 | -1 (planar) | $50 | 4 |
| Jagged | $14 | $60 | $40 | $FF | 0 (closed) | $00 | 4 |
| 8-shape | $0A | $A0 | $00 | $01 | -1 (planar) | $40 | 5 |

### ILINANG (Line Angle Indices)

Maps each vertex to an angle index (0-F) used for rendering orientation. From `ALDISP.MAC`, lines 1354-1371.

| Shape | 0 | 1 | 2 | 3 | 4 | 5 | 6 | 7 | 8 | 9 | A | B | C | D | E | F |
|-------|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|
| Circle | 5 | 6 | 7 | 8 | 9 | A | B | C | D | E | F | 0 | 1 | 2 | 3 | 4 |
| Square | 4 | 4 | 8 | 8 | 8 | 8 | C | C | C | C | 0 | 0 | 0 | 0 | 4 | 4 |
| Cross | 4 | 8 | 4 | 8 | 8 | C | 8 | C | C | 0 | C | 0 | 0 | 4 | 0 | 4 |
| Peanut | 6 | 7 | 9 | 8 | 7 | 9 | A | C | E | F | 1 | 0 | F | 1 | 2 | 4 |
| Key | 7 | 6 | 5 | 8 | B | A | 9 | C | F | E | D | 0 | 3 | 2 | 1 | 4 |
| Triangle | 5 | 5 | 5 | 5 | B | B | B | B | B | 0 | 0 | 0 | 0 | 0 | 0 | 5 |
| Clover | 4 | 8 | B | 5 | 8 | C | E | 9 | C | 0 | 3 | D | 0 | 4 | 7 | 2 |
| V | D | D | D | D | D | D | D | 0 | 3 | 3 | 3 | 3 | 3 | 3 | 3 | 0 |
| Flat | 0 | 0 | 0 | 0 | 0 | 0 | 0 | 0 | 0 | 0 | 0 | 0 | 0 | 0 | 0 | 0 |
| U | C | C | C | D | E | F | F | 0 | 1 | 1 | 2 | 3 | 4 | 4 | 4 | 0 |
| Jagged | E | D | C | D | D | D | 1 | F | 2 | 3 | 3 | 0 | 3 | 3 | 3 | 0 |
| 8-shape | B | 9 | 7 | 5 | 3 | 1 | F | D | D | F | 1 | 3 | 5 | 7 | 9 | B |
| Heart | 8 | B | C | 4 | 5 | 8 | B | C | D | E | F | 1 | 2 | 3 | 4 | 5 |
| Stairs | C | 0 | C | 0 | C | 0 | C | 0 | 4 | 0 | 4 | 0 | 4 | 0 | 4 | 0 |
| Star | A | 6 | C | 8 | E | A | 0 | C | 2 | E | 4 | 0 | 6 | 2 | 8 | 4 |
| Waves | E | C | D | E | 0 | 2 | 2 | 0 | E | E | 0 | 2 | 3 | 4 | 2 | 0 |