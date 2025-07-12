# Tempest: Sound and Audio Data

This document details the data-driven sound synthesis system in Tempest. All sound effects are generated programmatically by two POKEY sound chips; there are no pre-recorded samples. The data for all sounds is located in `ALSOUN.MAC`.

## Sound Data Format

The sound engine is highly data-driven. A "sound effect" is a collection of up to 8 parallel sequences, where each sequence controls one parameter of one POKEY channel (e.g., frequency of channel 1, volume of channel 1, etc.).

Each sequence is a series of 4-byte commands. These commands are processed frame-by-frame (at 60Hz) to create evolving sounds.

A single sound effect is defined by a set of tables, one for each POKEY parameter being controlled. For example, the Player Fire sound (`LA`) has a table for frequency (`LA3F`) and a table for amplitude/noise (`LA3A`).

### 4-Byte Sequence Command

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

## Sound Effect Data Tables

The following sections detail the raw data for each sound effect in the game. Each sound has one or more channels, typically for frequency (`...F`) and amplitude/noise (`...A`).

### Player Fire (`LA` - Launch)

A simple, short sound with a falling pitch.

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

### Player Dies (`DI`)

A complex, multi-stage sound with a warbling, falling tone.

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

### Enemy Explosion (`EX`)

A short, noisy burst.

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

### Thrust Sound on Tube (`T2`)

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

### Thrust Sound in Space (`T3`)

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

### Slam (`SL`)

The sound of an enemy hitting the player's ship at the rim. A loud, decaying noise. This sound loops.

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

### Pulsation (`PU` and `PO`)
A looping sound for the Pulsar enemy, and a sound to terminate it.

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