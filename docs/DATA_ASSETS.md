# Tempest: Game Data Assets

This document serves as a central repository for the game's core data assets, including sound definitions, vector shape data, and all user-facing text strings.

---
---

## Sound Data

This section details the data-driven sound synthesis system in Tempest. All sound effects are generated programmatically by two POKEY sound chips; there are no pre-recorded samples. The data for all sounds is located in [`ALSOUN.MAC`](./ALSOUN.md), which is documented in the [Sound Engine](./SYSTEMS.md#sound-engine-alsounmac) section.

### Sound Data Format

The [sound engine](./SYSTEMS.md#sound-engine-alsounmac) is highly data-driven. A "sound effect" is a collection of up to 8 parallel sequences, where each sequence controls one parameter of one POKEY channel (e.g., frequency of channel 1, volume of channel 1, etc.).

Each sequence is a series of 4-byte commands. These commands are processed frame-by-frame (at 60Hz) to create evolving sounds.

A single sound effect is defined by a set of tables, one for each POKEY parameter being controlled. For example, the [Player Fire](./ENTITIES.md#player-bullet) sound (`LA`) has a table for frequency (`LA3F`) and a table for amplitude/noise (`LA3A`).

#### 4-Byte Sequence Command

This is the fundamental building block of every sound:

| Byte | Parameter | Description |
| :--- | :--- | :--- |
| 1 | `START_VALUE` | The initial 8-bit value to write to the POKEY register (e.g., frequency). |
| 2 | `FRAME_COUNT` | The number of game frames (at 60Hz) before any change is made. |
| 3 | `CHANGE_AMOUNT` | A signed 8-bit value to add to the `START_VALUE` after each step. |
| 4 | `NUMBER_OF_CHANGES`| The total number of times to apply the change. |

**Special Commands:**
- A sequence of `(0, 0, _, _)` terminates the sound on that channel.
- A sequence of `(offset, 0, _, _)` creates a loop, where `offset` is a pointer back to an earlier part of the command table.

---

### Sound Effect Data Tables

The following sections detail the raw data for each sound effect in the game. Each sound has one or more channels, typically for frequency (`...F`) and amplitude/noise (`...A`).

#### Player Fire (`LA` - Launch)

A simple, short sound with a falling pitch, used for the [Player's Bullet](./ENTITIES.md#player-bullet).

**Channel `LA3F` (Frequency)**
| Start | Frames | Change | Count | Description |
| :--- | :--- | :--- | :--- | :--- |
| `0x10` | 1 | `+7` | 20 | Rapidly rising pitch. |
| `0x00` | 0 | - | - | Terminate. |

**Channel `LA3A` (Amplitude/Noise)**
| Start | Frames | Change | Count | Description |
| :--- | :--- | :--- | :--- | :--- |
| `0xA2` | 1 | `-8` | 20 | Rapidly fading volume. |
| `0x00` | 0 | - | - | Terminate. |

#### Player Dies (`DI`)

A complex, multi-stage sound for when the [Player](./ENTITIES.md#player) is destroyed.

**Channel `DI1F` (Frequency)**
| Start | Frames | Change | Count | Description |
| :--- | :--- | :--- | :--- | :--- |
| `0x08` | 4 | `+20` | 10 | Stage 1 |
| `0x08` | 4 | `+1` | 9 | Stage 2 |
| `0x10` | 13 | `+4` | 12 | Stage 3 |
| `0x00` | 0 | - | - | Terminate. |

**Channel `DI1A` (Amplitude/Noise)**
| Start | Frames | Change | Count | Description |
| :--- | :--- | :--- | :--- | :--- |
| `0x08` | 4 | `+0` | 10 | Stage 1 |
| `0x68` | 4 | `+0` | 9 | Stage 2 |
| `0x68` | 12 | `-1` | 9 | Stage 3 |
| `0x00` | 0 | - | - | Terminate. |

#### Enemy Explosion (`EX`)

A short, noisy burst used for generic [enemy](./ENTITIES.md#enemies) explosions.

**Channel `EX2F` (Frequency)**
| Start | Frames | Change | Count | Description |
| :--- | :--- | :--- | :--- | :--- |
| `0x01` | 8 | `+2` | 16 | Low, rising tone. |
| `0x00` | 0 | - | - | Terminate. |

**Channel `EX2A` (Amplitude/Noise)**
| Start | Frames | Change | Count | Description |
| :--- | :--- | :--- | :--- | :--- |
| `0x86` | 20 | `+0` | 4 | Noisy burst. |
| `0x00` | 0 | - | - | Terminate. |

#### Thrust Sound on Tube (`T2`)

A constant, high-pitched whine. This sound loops indefinitely until stopped.

**Channel `T26F` (Frequency)**
| Start | Frames | Change | Count | Description |
| :--- | :--- | :--- | :--- | :--- |
| `0xC0` | 2 | `-1` | 255 | High-pitched whine. |
| `offset`| 0 | - | - | Loop to start. |

**Channel `T26A` (Amplitude/Noise)**
| Start | Frames | Change | Count | Description |
| :--- | :--- | :--- | :--- | :--- |
| `0x28` | 2 | `+0` | 240 | Constant noise. |
| `offset`| 0 | - | - | Loop to start. |

#### Thrust Sound in Space (`T3`)

A lower-pitched, rising thrust sound.

**Channel `T36F` (Frequency)**
| Start | Frames | Change | Count | Description |
| :--- | :--- | :--- | :--- | :--- |
| `0x10` | 11 | `+1` | 64 | Slowly rising pitch. |
| `0x00` | 0 | - | - | Terminate. |

**Channel `T36A` (Amplitude/Noise)**
| Start | Frames | Change | Count | Description |
| :--- | :--- | :--- | :--- | :--- |
| `0x86` | 64 | `+0` | 11 | Constant noise. |
| `0x00` | 0 | - | - | Terminate. |

#### Slam (`SL`)

The sound of an enemy hitting the [player's ship](./ENTITIES.md#player) at the rim. A loud, decaying noise. This sound loops.

**Channel `SL1F` (Frequency)**
| Start | Frames | Change | Count | Description |
| :--- | :--- | :--- | :--- | :--- |
| `0x18` | 4 | `+0` | 255 | Constant low tone. |
| `offset`| 0 | - | - | Loop to start. |

**Channel `SL1A` (Amplitude/Noise)**
| Start | Frames | Change | Count | Description |
| :--- | :--- | :--- | :--- | :--- |
| `0xAF` | 4 | `+0` | 255 | Loud, constant noise. |
| `offset`| 0 | - | - | Loop to start. |

#### Pulsation (`PU` and `PO`)
A looping sound for the [Pulsar](./ENTITIES.md#pulsar) enemy, and a sound to terminate it.

**Channel `PU6F` (Frequency - Pulsation ON)**
| Start | Frames | Change | Count | Description |
| :--- | :--- | :--- | :--- | :--- |
| `0xB0` | 2 | `0` | 255 | Looping sound. |
| `offset`| 0 | - | - | Loop. |

**Channel `PU6A` (Amplitude/Noise - Pulsation ON)**
| Start | Frames | Change | Count | Description |
| :--- | :--- | :--- | :--- | :--- |
| `0xC8` | 1 | `+2` | 255 | Looping sound. |
| `offset`| 0 | - | - | Loop. |

**Channel `PO6A` (Amplitude/Noise - Pulsation OFF)**
| Start | Frames | Change | Count | Description |
| :--- | :--- | :--- | :--- | :--- |
| `0xC0` | 1 | `0` | 1 | Set volume to 0. |
| `0x00` | 0 | - | - | Terminate. |

---
---

## Vector Shape Data

This document contains the raw data for every vector object in Tempest, sufficient for a full, high-fidelity reconstruction of the game's graphics. All data is sourced from `ALVROM.MAC` (game objects) and `ANVGAN.MAC` (alphanumeric font), which are part of the [Vector Graphics Engine](./SYSTEMS.md#vector-graphics-engine).

The shapes are drawn by a custom hardware processor called the [Vector Generator (VG)](./SYSTEMS.md#vector-graphics-engine). To understand the data, it is first necessary to understand the VG's instruction set.

### Vector Generator Instruction Set

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

### Alphanumeric Font Data

The vector shapes for the standard character set (A-Z, 0-9, and symbols) are defined in [`ANVGAN.MAC`](./ANVGAN.md). Each character is a subroutine that can be called with `JSRL`. The default brightness is typically set to 4, but can be overridden.

#### `CHAR.A`
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

#### `CHAR.B`
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

#### `CHAR.C`
```
VCTR 0,24,4
VCTR 16,0,4
VCTR -16,-24,0
VCTR 16,0,4
VCTR 8,0,0
RTSL
```

#### `CHAR.D`
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

#### `CHAR.E`
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

#### `CHAR.F`
```
VCTR 0,24,4
VCTR 16,0,4
VCTR -4,-12,0
VCTR -12,0,4
VCTR 0,-12,0
VCTR 24,0,0
RTSL
```

#### `CHAR.G`
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

#### `CHAR.H`
```
VCTR 0,24,4
VCTR 0,-12,0
VCTR 16,0,4
VCTR 0,12,0
VCTR 0,-24,4
VCTR 8,0,0
RTSL
```

#### `CHAR.I`
```
VCTR 16,0,4
VCTR -8,0,0
VCTR 0,24,4
VCTR 8,0,0
VCTR -16,0,4
VCTR 24,-24,0
RTSL
```

#### `CHAR.J`
```
VCTR 0,8,0
VCTR 8,-8,4
VCTR 8,0,4
VCTR 0,24,4
VCTR 8,-24,0
RTSL
```

#### `CHAR.K`
```
VCTR 0,24,4
VCTR 12,0,0
VCTR -12,-12,4
VCTR 12,-12,4
VCTR 12,0,0
RTSL
```

#### `CHAR.L`
```
VCTR 0,24,0
VCTR 0,-24,4
VCTR 16,0,4
VCTR 8,0,0
RTSL
```

#### `CHAR.M`
```
VCTR 0,24,4
VCTR 8,-8,4
VCTR 8,8,4
VCTR 0,-24,4
VCTR 8,0,0
RTSL
```

#### `CHAR.N`
```
VCTR 0,24,4
VCTR 16,-24,4
VCTR 0,24,4
VCTR 8,-24,0
RTSL
```

#### `CHAR.O` and `CHAR.0`
```
VCTR 0,24,4
VCTR 16,0,4
VCTR 0,-24,4
VCTR -16,0,4
VCTR 24,0,0
RTSL
```

#### `CHAR.P`
```
VCTR 0,24,4
VCTR 16,0,4
VCTR 0,-12,4
VCTR -16,0,4
VCTR 12,-12,0
VCTR 12,0,0
RTSL
```

#### `CHAR.Q`
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

#### `CHAR.R`
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

#### `CHAR.S` and `CHAR.5`
```
VCTR 16,0,4
VCTR 0,12,4
VCTR -16,0,4
VCTR 0,12,4
VCTR 16,0,4
VCTR 8,-24,0
RTSL
```

#### `CHAR.T`
```
VCTR 8,0,0
VCTR 0,24,4
VCTR -8,0,0
VCTR 16,0,4
VCTR 8,-24,0
RTSL
```

#### `CHAR.U`
```
VCTR 0,24,0
VCTR 0,-24,4
VCTR 16,0,4
VCTR 0,24,4
VCTR 8,-24,0
RTSL
```

#### `CHAR.V`
```
VCTR 0,24,0
VCTR 8,-24,4
VCTR 8,24,4
VCTR 8,-24,0
RTSL
```

#### `CHAR.W`
```
VCTR 0,24,0
VCTR 0,-24,4
VCTR 8,8,4
VCTR 8,-8,4
VCTR 0,24,4
VCTR 8,-24,0
RTSL
```

#### `CHAR.X`
```
VCTR 16,24,4
VCTR -16,0,0
VCTR 16,-24,4
VCTR 8,0,0
RTSL
```

#### `CHAR.Y`
```
VCTR 8,0,0
VCTR 0,16,4
VCTR -8,8,4
VCTR 16,0,0
VCTR -8,-8,4
VCTR 16,-16,0
RTSL
```

#### `CHAR.Z`
```
VCTR 0,24,0
VCTR 16,0,4
VCTR -16,-24,4
VCTR 16,0,4
VCTR 8,0,0
RTSL
```

#### `CHAR. ` (Space)
```
VCTR 24,0,0
RTSL
```

#### `CHAR.1`
```
VCTR 8,0,0
VCTR 0,24,4
VCTR 16,-24,0
RTSL
```

#### `CHAR.2`
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

#### `CHAR.3`
```
VCTR 16,0,4
VCTR 0,24,4
VCTR -16,0,4
VCTR 0,-12,0
VCTR 16,0,4
VCTR 8,-12,0
RTSL
```

#### `CHAR.4`
```
VCTR 0,24,0
VCTR 0,-12,4
VCTR 16,0,4
VCTR 0,12,0
VCTR 0,-24,4
VCTR 8,0,0
RTSL
```

#### `CHAR.6`
```
VCTR 0,12,0
VCTR 16,0,4
VCTR 0,-12,4
VCTR -16,0,4
VCTR 0,24,4
VCTR 24,-24,0
RTSL
```

#### `CHAR.7`
```
VCTR 0,24,0
VCTR 16,0,4
VCTR 0,-24,4
VCTR 8,0,0
RTSL
```

#### `CHAR.8`
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

#### `CHAR.9`
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

### Game Object and UI Shapes

These shapes, defined in `ALVROM.MAC`, represent all the visible entities in the game other than the text font. This includes the [player's ship](./ENTITIES.md#player), [enemies](./ENTITIES.md#enemies), [explosions](#explosions-expl1---expl4-splat-shrap), [shots](./ENTITIES.md#player-bullet), and UI elements like the screen border.

Many shapes are defined using custom macros like `SCVEC` and `SCDOT` which are thin wrappers around the `VCTR` command to allow for scaling. For clarity, the parameters to these macros are shown as-is. The color of an object is set by a `CSTAT` command before its `JSRL` is called.

#### Player Ship (`LIFEY`/`LIFE1`)
The [player's ship](./ENTITIES.md#player) is a simple "U" shape. The `LIFEY` routine sets the color to yellow before calling `LIFE1` to do the drawing.

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

#### Explosions (`EXPL1` - `EXPL4`, `SPLAT`, `SHRAP`)

The game uses several types of explosions. The standard explosion is a sequence of four expanding "spoke" images. [Player](./ENTITIES.md#player) death animations (`SPLAT`) and shrapnel are more complex.

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
The `SPLAT` picture is a multi-colored, irregular shape used when the [player collides with an enemy](./GAME_STATE_FLOW.md#cplay--gameplay) on the web. It's drawn at various scales (`SPLAT1` to `SPLAT6`).
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

#### Player Shot (`DIARA2`)
The [player's shot](./ENTITIES.md#player-bullet) is a collection of dots arranged in concentric octagons.
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

#### Screen Boundary (`BONDRY`)
This routine simply draws a white box around the edges of the screen, defining the [Playfield](./PLAYFIELD.md).
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

#### Enemies

The various enemies in Tempest have distinct shapes. Many have multiple frames of animation, which are separate drawing routines. The [game logic](./GAME_STATE_FLOW.md#cplay--gameplay) cycles through these routines to create the animation.

##### Flipper (aka `ENER11` - `ENER14`)
The basic ["flipper"](./ENTITIES.md#flipper) enemy that moves along the rim of the well. It has a four-frame animation sequence. The color is set by the game logic before drawing.

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

##### Tanker (`TANKR`, `TANKF`, `TANKP`)
[Tankers](./ENTITIES.md#tanker) are composed of a generic frame (`GENTNK`) and a core. There are three types: a plain tanker (`TANKR`), a [Fuse Tanker](./ENTITIES.md#tanker) (which lays [Spikers](./ENTITIES.md#spiker)), and a [Pulsar Tanker](./ENTITIES.md#tanker) (which fires [Pulsars](./ENTITIES.md#pulsar)).

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

##### Spiker (`SPIRA1` - `SPIRA4`)
The [spiker](./ENTITIES.md#spiker) has a four-frame spinning animation. It is always drawn in green.

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

##### Fuseball & Pulsar (`FUSE0`-`FUSE3`, `ENER21`-`ENER24`)
These are complex, multi-colored enemies with four animation frames each, representing the [Fuseball](./ENTITIES.md#fuseball) and [Pulsar](./ENTITIES.md#pulsar) entities.

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

##### Enemy Shots (`ESHOT1`-`ESHOT4`)
The rotating shots fired by some enemies, documented as the [Enemy Bullet](./ENTITIES.md#enemy-bullet) entity. They are two-colored (White and Red).

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

---
---

## Text and Language Data

This file contains all user-facing text strings used in Tempest, with translations for all four supported languages. This data is sourced directly from [`ALLANG.MAC`](./ALLANG.md).

The game engine uses a system of message IDs to retrieve the correct string for the currently selected language. The `ASCVH` macro is used to convert the source text into a compact, drawable format. For clarity, the raw, uncompressed text is presented here.

### Game Text and Translations

| Message ID | English | French | German | Spanish |
| :--- | :--- | :--- | :--- | :--- |
| `GAMOV` | GAME OVER | FIN DE PARTIE | SPIELENDE | JUEGO TERMINADO |
| `PLAYR` | PLAYER | JOUEUR | SPIELER | JUGADOR |
| `PLYR2` | PLAYER | JOUEUR | SPIELER | JUGADOR |
| `PRESS` | PRESS START | APPUYEZ SUR START | START DRUECKEN | PULSAR START |
| `PLAY` | PLAY | JOUEZ | SPIEL | JUEGUE |
| `ENTER` | ENTER YOUR INITIALS | SVP ENTREZ VOS INITIALES | GEBEN SIE IHRE INITIALEN EIN | ENTRE SUS INICIALES |
| `PRMOV` | SPIN KNOB TO CHANGE | TOURNEZ LE BOUTON POUR CHANGER | KNOPF DREHEN ZUM WECHSELN | GIRE LA PERILLA PARA CAMBIAR |
| `PRFIR` | PRESS FIRE TO SELECT | POUSSEZ FEU QUAND CORRECTE | FIRE DRUECKEN WENN RICHTIG | OPRIMA FIRE PARA SELECCIONAR |
| `HIGHS` | HIGH SCORES | MEILLEURS SCORES | HOECHSTZAHLEN | RECORDS |
| `RANK` | RANKING FROM 1 TO | PLACEMENT DE 1 A | RANGLISTE VON 1 ZUM | RANKING DE 1 A |
| `RATE` | RATE YOURSELF | EVALUEZ-VOUS | SELBST RECHNEN | CALIFIQUESE |
| `NOVIC` | NOVICE | NOVICE | ANFAENGER | NOVICIO |
| `EXPER` | EXPERT | EXPERT | ERFAHREN | EXPERTO |
| `BONUS` | BONUS | BONUS | BONUS | BONUS |
| `TIME` | TIME | DUREE | ZEIT | TIEMPO |
| `LEVEL` | LEVEL | NIVEAU | GRAD | NIVEL |
| `HOLE` | HOLE | TROU | LOCH | HOYO |
| `INSER` | INSERT COINS | INTRODUIRE LES PIECES | GELD EINWERFEN | INSERTE FICHAS |
| `CMODE` | FREE PLAY | FREE PLAY | FREE PLAY | FREE PLAY |
| `CMOD1` | 1 COIN 2 PLAYS | 1 PIECE 2 JOUEURS | 1 MUENZ 2 SPIELE | 1 MONEDA 2 JUEGOS |
| `CMOD2` | 1 COIN 1 PLAY | 1 PIECE 1 JOUEUR | 1 MUENZE 1 SPIEL | 1 MONEDA 1 JUEGO |
| `CMOD3` | 2 COINS 1 PLAY | 2 PIECES 1 JOUEUR | 2 MUENZEN 1 SPIEL | 2 MONEDAS 1 JUEGO |
| `ATARI` | © MCMLXXX ATARI | © MCMLXXX ATARI | © MCMLXXX ATARI | © MCMLXXX ATARI |
| `CREDI` | CREDITS | CREDITS | KREDITE | CREDITOS |
| `BONPT` | BONUS | BONUS | BONUS | BONUS |
| `2GAME` | 2 CREDIT MINIMUM | 2 JEUX MINIMUM | 2 SPIELE MINIMUM | 2 JUEGOS MINIMO |
| `BOLIF` | BONUS EVERY | BONUS CHAQUE | BONUS JEDE | BONUS CADA |
| `SPIKE` | AVOID SPIKES | ATTENTION AUX LANCES | SPITZEN AUSWEICHEN | EVITE LAS PUNTAS |
| `APROA` | LEVEL | NIVEAU | GRAD | NIVEL |
| `SUPZA` | SUPERZAPPER RECHARGE | SUPERZAPPER RECHARGE | NEUER SUPERZAPPER | NUEVO SUPERZAPPER | 