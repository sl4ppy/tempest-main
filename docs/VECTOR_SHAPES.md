# Tempest: Vector Shape Data

This document contains the raw data for every vector object in Tempest, sufficient for a full, high-fidelity reconstruction of the game's graphics. All data is sourced from `ALVROM.MAC` (game objects) and `ANVGAN.MAC` (alphanumeric font).

The shapes are drawn by a custom hardware processor called the Vector Generator (VG). To understand the data, it is first necessary to understand the VG's instruction set.

## Vector Generator Instruction Set

The instructions for the Vector Generator are defined as macros in `VGMC.MAC`. These macros assemble into 16-bit words that the VG processor executes to draw on screen. The game creates "display lists" of these instructions in RAM, which the VG then processes.

Here are the primary commands:

| Macro | Opcode (Hex) | Parameters | Description |
| :--- | :--- | :--- | :--- |
| `VCTR` | `4000` or `----` | `DX, DY, ZZ` | The most common instruction. Draws a vector (a line) from the current beam position to a new relative position `(X+DX, Y+DY)`. `ZZ` is the brightness (intensity) from 0 (off) to 7 (brightest). The hardware supports a "short vector" format for small delta values (opcode `0x4000`) and a two-word "long vector" format for larger ones. |
| `JSRL` | `A000` | `LABL` | **Jump to Subroutine**. Pushes the current address onto the VG's stack and jumps to the address specified by `LABL` in the Vector ROM. Used to draw complex, reusable shapes. |
| `RTSL` | `C000` | (none) | **Return from Subroutine**. Pops an address from the VG's stack and returns to it. |
| `JMPL` | `E000` | `LABL` | **Jump**. Unconditionally jumps to the address specified by `LABL`. Used to chain display lists. |
| `SCAL` | `7000` | `S, LS` | **Set Scale**. Sets the scaling for all subsequent vectors. `S` is a power-of-2 scale (0=full, 7=1/128th). `LS` is a fine-grained linear scale. |
| `CSTAT`| `68C0` | `CLR` | **Set Color Status**. A Tempest-specific macro that sets the color for subsequent vectors. In the original black-and-white hardware, this controlled intensity. |
| `STAT` | `6000` | `Z, HI, IN` | **Set Status**. The generic version of `CSTAT`. Sets intensity (`Z`) and clipping window parameters. |
| `CNTR` | `8040` | (none) | **Center**. Moves the beam to the center of the screen `(0,0)`. |
| `HALT` | `2000` | (none) | **Halt**. Stops the Vector Generator processor. |

---

## Alphanumeric Font Data

The vector shapes for the standard character set (A-Z, 0-9, and symbols) are defined in `ANVGAN.MAC`. Each character is a subroutine that can be called with `JSRL`. The default brightness is typically set to 4, but can be overridden.

### `CHAR.A`
```
VCTR 0,16,4
VCTR 8,8,4
VCTR 8,-8,4
VCTR 0,-16,4
VCTR -16,8,0
VCTR 16,0,4
VCTR 8,-8,0
RTSL
```

### `CHAR.B`
```
VCTR 0,24,4
VCTR 12,0,4
VCTR 4,-4,3
VCTR 0,-4,3
VCTR -4,-4,3
VCTR -12,0,4
VCTR 12,0,0
VCTR 4,-4,3
VCTR 0,-4,3
VCTR -4,-4,3
VCTR -12,0,4
VCTR 24,0,0
RTSL
```

### `CHAR.C`
```
VCTR 0,24,4
VCTR 16,0,4
VCTR -16,-24,0
VCTR 16,0,4
VCTR 8,0,0
RTSL
```

### `CHAR.D`
```
VCTR 0,24,4
VCTR 8,0,4
VCTR 8,-8,4
VCTR 0,-8,4
VCTR -8,-8,4
VCTR -8,0,4
VCTR 24,0,0
RTSL
```

### `CHAR.E`
```
VCTR 0,24,4
VCTR 16,0,4
VCTR -4,-12,0
VCTR -12,0,4
VCTR 0,-12,0
VCTR 16,0,4
VCTR 8,0,0
RTSL
```

### `CHAR.F`
```
VCTR 0,24,4
VCTR 16,0,4
VCTR -4,-12,0
VCTR -12,0,4
VCTR 0,-12,0
VCTR 24,0,0
RTSL
```

### `CHAR.G`
```
VCTR 0,24,4
VCTR 16,0,4
VCTR 0,-8,4
VCTR -8,-8,0
VCTR 8,0,4
VCTR 0,-8,4
VCTR -16,0,4
VCTR 24,0,0
RTSL
```

### `CHAR.H`
```
VCTR 0,24,4
VCTR 0,-12,0
VCTR 16,0,4
VCTR 0,12,0
VCTR 0,-24,4
VCTR 8,0,0
RTSL
```

### `CHAR.I`
```
VCTR 16,0,4
VCTR -8,0,0
VCTR 0,24,4
VCTR 8,0,0
VCTR -16,0,4
VCTR 24,-24,0
RTSL
```

### `CHAR.J`
```
VCTR 0,8,0
VCTR 8,-8,4
VCTR 8,0,4
VCTR 0,24,4
VCTR 8,-24,0
RTSL
```

### `CHAR.K`
```
VCTR 0,24,4
VCTR 12,0,0
VCTR -12,-12,4
VCTR 12,-12,4
VCTR 12,0,0
RTSL
```

### `CHAR.L`
```
VCTR 0,24,0
VCTR 0,-24,4
VCTR 16,0,4
VCTR 8,0,0
RTSL
```

### `CHAR.M`
```
VCTR 0,24,4
VCTR 8,-8,4
VCTR 8,8,4
VCTR 0,-24,4
VCTR 8,0,0
RTSL
```

### `CHAR.N`
```
VCTR 0,24,4
VCTR 16,-24,4
VCTR 0,24,4
VCTR 8,-24,0
RTSL
```

### `CHAR.O` and `CHAR.0`
```
VCTR 0,24,4
VCTR 16,0,4
VCTR 0,-24,4
VCTR -16,0,4
VCTR 24,0,0
RTSL
```

### `CHAR.P`
```
VCTR 0,24,4
VCTR 16,0,4
VCTR 0,-12,4
VCTR -16,0,4
VCTR 12,-12,0
VCTR 12,0,0
RTSL
```

### `CHAR.Q`
```
VCTR 0,24,4
VCTR 16,0,4
VCTR 0,-16,4
VCTR -8,-8,4
VCTR -8,0,4
VCTR 8,8,0
VCTR 8,-8,4
VCTR 8,0,0
RTSL
```

### `CHAR.R`
```
VCTR 0,24,4
VCTR 16,0,4
VCTR 0,-12,4
VCTR -16,0,4
VCTR 4,0,0
VCTR 12,-12,4
VCTR 8,0,0
RTSL
```

### `CHAR.S` and `CHAR.5`
```
VCTR 16,0,4
VCTR 0,12,4
VCTR -16,0,4
VCTR 0,12,4
VCTR 16,0,4
VCTR 8,-24,0
RTSL
```

### `CHAR.T`
```
VCTR 8,0,0
VCTR 0,24,4
VCTR -8,0,0
VCTR 16,0,4
VCTR 8,-24,0
RTSL
```

### `CHAR.U`
```
VCTR 0,24,0
VCTR 0,-24,4
VCTR 16,0,4
VCTR 0,24,4
VCTR 8,-24,0
RTSL
```

### `CHAR.V`
```
VCTR 0,24,0
VCTR 8,-24,4
VCTR 8,24,4
VCTR 8,-24,0
RTSL
```

### `CHAR.W`
```
VCTR 0,24,0
VCTR 0,-24,4
VCTR 8,8,4
VCTR 8,-8,4
VCTR 0,24,4
VCTR 8,-24,0
RTSL
```

### `CHAR.X`
```
VCTR 16,24,4
VCTR -16,0,0
VCTR 16,-24,4
VCTR 8,0,0
RTSL
```

### `CHAR.Y`
```
VCTR 8,0,0
VCTR 0,16,4
VCTR -8,8,4
VCTR 16,0,0
VCTR -8,-8,4
VCTR 16,-16,0
RTSL
```

### `CHAR.Z`
```
VCTR 0,24,0
VCTR 16,0,4
VCTR -16,-24,4
VCTR 16,0,4
VCTR 8,0,0
RTSL
```

### `CHAR. ` (Space)
```
VCTR 24,0,0
RTSL
```

### `CHAR.1`
```
VCTR 8,0,0
VCTR 0,24,4
VCTR 16,-24,0
RTSL
```

### `CHAR.2`
```
VCTR 0,24,0
VCTR 16,0,4
VCTR 0,-12,4
VCTR -16,0,4
VCTR 0,-12,4
VCTR 16,0,4
VCTR 8,0,0
RTSL
```

### `CHAR.3`
```
VCTR 16,0,4
VCTR 0,24,4
VCTR -16,0,4
VCTR 0,-12,0
VCTR 16,0,4
VCTR 8,-12,0
RTSL
```

### `CHAR.4`
```
VCTR 0,24,0
VCTR 0,-12,4
VCTR 16,0,4
VCTR 0,12,0
VCTR 0,-24,4
VCTR 8,0,0
RTSL
```

### `CHAR.6`
```
VCTR 0,12,0
VCTR 16,0,4
VCTR 0,-12,4
VCTR -16,0,4
VCTR 0,24,4
VCTR 24,-24,0
RTSL
```

### `CHAR.7`
```
VCTR 0,24,0
VCTR 16,0,4
VCTR 0,-24,4
VCTR 8,0,0
RTSL
```

### `CHAR.8`
```
VCTR 16,0,4
VCTR 0,24,4
VCTR -16,0,4
VCTR 0,-24,4
VCTR 0,12,0
VCTR 16,0,4
VCTR 8,-12,0
RTSL
```

### `CHAR.9`
```
VCTR 16,0,0
VCTR 0,24,4
VCTR -16,0,4
VCTR 0,-12,4
VCTR 16,0,4
VCTR 8,-12,0
RTSL
```

---

## Game Object and UI Shapes

These shapes, defined in `ALVROM.MAC`, represent all the visible entities in the game other than the text font. This includes the player's ship, enemies, explosions, shots, and UI elements like the screen border.

Many shapes are defined using custom macros like `SCVEC` and `SCDOT` which are thin wrappers around the `VCTR` command to allow for scaling. For clarity, the parameters to these macros are shown as-is. The color of an object is set by a `CSTAT` command before its `JSRL` is called.

### Player Ship (`LIFEY`/`LIFE1`)
The player's ship is a simple "U" shape. The `LIFEY` routine sets the color to yellow before calling `LIFE1` to do the drawing.

```
LIFEY:
    CSTAT YELLOW
LIFE1:
    ; CM=6, CD=1, CB=6
    ICVEC
    SCVEC 4,-2,6
    SCVEC 1,-3,6
    SCVEC 3,-2,6
    SCVEC 0,-1,6
    SCVEC -3,-2,6
    SCVEC -1,-3,6
    SCVEC -4,-2,6
    SCVEC 0,0,6
    SCVEC 10,0,0
    RTSL
```

### Explosions (`EXPL1` - `EXPL4`, `SPLAT`, `SHRAP`)

The game uses several types of explosions. The standard explosion is a sequence of four expanding "spoke" images. Player death animations (`SPLAT`) and shrapnel are more complex.

**Standard Explosion**
These are created by calling the `SPOK16` macro (16 spokes) with progressively larger scaling factors.
- `EXPL1`: `CM=1` (Smallest)
- `EXPL2`: `CM=2`
- `EXPL3`: `CM=4`
- `EXPL4`: `CM=8` (Largest)

```
SPOK16:
    ; Input: CM=Scale, CB=Brightness
    ICVEC
    SCVEC 7,3,0
    SCVEC -7,-3,CB
    SCVEC -6,-6,0
    SCVEC 6,6,CB
    ; ... and so on for 16 spokes
    RTSL
```

**Player "Splat" Animation**
The `SPLAT` picture is a multi-colored, irregular shape used when the player collides with an enemy on the web. It's drawn at various scales (`SPLAT1` to `SPLAT6`).
```
SPLAT:
    ICVEC
    CSTAT PDIWHI ; 9
    SCVEC 18,-8,0
    SCVEC 38,8,7
    SCVEC 20,12,7
    CSTAT PDIRED ; 11
    SCVEC 24,14,7
    SCVEC 28,21,7
    ; ... many more vectors
    RTSL
```

**Shrapnel**
The `SHRAP` picture is used for certain explosions and is composed of several disconnected pieces and dots.
```
SHRAP:
    ICVEC
    JSRL SCALE
    SCVEC 20,-14,0
    SCAL 1,0
    CSTAT YELLOW
    SCVEC 25,-14,7 ; Piece 1
    SCVEC 25,-19,7
    ; ... etc
    RTSL
```

### Player Shot (`DIARA2`)
The player's shot is a collection of dots arranged in concentric octagons.
```
DIARA2:
    ICVEC
    CSTAT PSHCTR ; 8
    SCDOT 0,0
    SCDOT 7,0
    SCDOT 5,5
    SCDOT 0,7
    SCDOT -5,5
    SCDOT -7,0
    SCDOT -5,-5
    SCDOT 0,-7
    SCDOT 5,-5
    CSTAT YELLOW
    SCDOT 15,0
    SCDOT 11,11
    SCDOT 0,15
    SCDOT -11,11
    SCDOT -11,0
    SCDOT -11,-11
    SCDOT 0,-11
    SCDOT 11,-11
    RTSL
```

### Screen Boundary (`BONDRY`)
This routine simply draws a white box around the edges of the screen.
```
BONDRY:
    CSTAT WHITE
    SCAL 1,0
    CNTR
    VCTR -500,-540,0
    VCTR 1000,0,6
    VCTR 0,1080,6
    VCTR -1000,0,6
    VCTR 0,-1080,6
    RTSL
```

---

### Enemies

The various enemies in Tempest have distinct shapes. Many have multiple frames of animation, which are separate drawing routines. The game logic cycles through these routines to create the animation.

#### Flipper (aka `ENER11` - `ENER14`)
The basic "flipper" enemy that moves along the rim of the well. It has a four-frame animation sequence. The color is set by the game logic before drawing.

```
ENER11: ; Frame 1
	ICALVE
	CALVEC -4,4
	.BRITE=VARBRT
	CALVEC -17,3
	CALVEC -4,29
	CALVEC -44,-21
	CALVEC -17,-2
    ; ... more vectors
	RTSL

ENER12: ; Frame 2
	ICALVE
	CALVEC -4,-3
	.BRITE=VARBRT
	CALVEC -15,4
	CALVEC -4,24
    ; ... more vectors
	RTSL

ENER13: ; Frame 3
	ICALVE
	CALVEC -4,2
	.BRITE=VARBRT
	CALVEC -13,5
	CALVEC -6,18
    ; ... more vectors
	RTSL

ENER14: ; Frame 4
	ICALVE
	CALVEC -4,1
	.BRITE=VARBRT
	CALVEC -11,5
	CALVEC -12,13
    ; ... more vectors
	RTSL
```

#### Tanker (`TANKR`, `TANKF`, `TANKP`)
Tankers are composed of a generic frame (`GENTNK`) and a core. There are three types: a plain tanker (`TANKR`), a Fuse Tanker (`TANKF`), and a Pulsar Tanker (`TANKP`).

```
GENTNK: ; Generic Tanker Frame
	CSTAT PURPLE
	SCVEC 0,20,CB
	SCVEC 0,12,CB
    ; ... more vectors
	RTSL

TANKF: ; Fuse Tanker Core
	ICVEC
	CSTAT BLUE
	SCVEC -12,0,CB
	SCVEC 0,12,0
	CSTAT RED
	SCVEC 0,0,CB
    ; ... more vectors
	JMPL GENTNK

TANKP: ; Pulsar Tanker Core
	ICVEC
	CSTAT TURQOI
	SCVEC -5,-2,0
	SCVEC -3,6,CB
	SCVEC 0,-6,CB
    ; ... more vectors
	JMPL GENTNK
```

#### Spiker (`SPIRA1` - `SPIRA4`)
The spiker has a four-frame spinning animation. It is always drawn in green.

```
SPIRA1: ; Frame 1
	CSTAT GREEN
	ICVEC
	SCVEC 1,-1,1
	SCVEC 0,-2,1
	SCVEC -2,-2,1
    ; ... more vectors
	RTSL

SPIRA2: ; Frame 2
	CSTAT GREEN
	ICVEC
	SCVEC 1,1,1
	SCVEC 2,0,1
	SCVEC 2,-2,1
    ; ... more vectors
	RTSL

SPIRA3: ; Frame 3
	CSTAT GREEN
	ICVEC
	SCVEC -1,1,1
	SCVEC 0,2,1
	SCVEC 2,2,1
    ; ... more vectors
	RTSL

SPIRA4: ; Frame 4
	CSTAT GREEN
	ICVEC
	SCVEC -1,-1,1
	SCVEC -2,0,1
	SCVEC -2,2,1
    ; ... more vectors
	RTSL
```

#### Fuseball & Pulsar (`FUSE0`-`FUSE3`, `ENER21`-`ENER24`)
These are complex, multi-colored enemies with four animation frames each.

```
FUSE0: ; Fuseball Frame 0
	ICVEC
	CSTAT RED
	SCVEC -4,6,7
	SCVEC 1,12,7
    ; ... more vectors
	RTSL

ENER21: ; Pulsar Frame 1
	ICALVE
	CALVEC -1,-3
	.BRITE=VARBRT
	CALVEC -4,24
    ; ... more vectors
	RTSL
```

#### Enemy Shots (`ESHOT1`-`ESHOT4`)
The rotating shots fired by some enemies. They are two-colored (White and Red).

```
ESHOT1: ; Frame 1
	ICVEC
	CSTAT WHITE
	SCVEC -11,11,0
	SCVEC -17,17,6
    ; ...
	CSTAT RED
	VCTR 0,0,6
	SCVEC -6,6,0
    ; ... more vectors
	RTSL
``` 