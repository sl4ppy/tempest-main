# Tempest: Core Systems Documentation

## 1. Sound Engine (`ALSOUN.MAC`)

This document details the architecture and function of the Tempest sound engine. It is a complete, data-driven, procedural audio subsystem responsible for generating all in-game sound effects.

---

### 1.1. Purpose and Overview

The purpose of the sound engine is to provide a flexible, memory-efficient system for generating complex audio events on the Atari POKEY sound chip. Unlike sample-based audio, this engine synthesizes sounds in real-time by procedurally modulating the POKEY chip's registers frame-by-frame according to predefined data patterns.

-   **Context of Use:** The engine is used throughout the game to generate all audio, from the player's laser fire and explosions to enemy sound effects and UI feedback clicks. The main game loop calls the `MODSND` routine every frame to continue sound playback, and various game logic routines call `SNDON` to trigger new sounds.
-   **Interfaces:** The engine exposes three primary subroutines to the rest of the game code:
    -   `INISOU`: Initializes the two POKEY sound chips and all software channel states. Called once at game startup.
    -   `SNDON`: Triggers a new sound effect. The caller loads the desired Sound ID into the accumulator and jumps to this routine.
    -   `MODSND`: The main update routine. It is called once per frame (e.g., from the VBLANK interrupt) to advance the state of all currently playing sounds.

---

### 1.2. Architecture and Structure

The sound system is a hybrid software/hardware architecture. The `ALSOUN.MAC` file contains the software driver, which controls two hardware POKEY sound chips.

-   **Main Components:**
    1.  **POKEY Sound Chips (x2):** The hardware that produces the audio signals. Each POKEY chip provides 4 independent audio channels, for a total of 8 physical channels.
    2.  **Sound Data Tables:** Located in `ALSOUN.MAC`, these are read-only data structures that define the sequence of changes for each sound effect.
    3.  **Channel State RAM:** A set of arrays in RAM used by the driver to track the current state of each of the 10 logical sound channels (the driver supports 10, but maps them to the 8 physical POKEY channels).
    4.  **Software Driver:** The collection of subroutines (`INISOU`, `SNDON`, `MODSND`) that reads the data tables, updates the state RAM, and writes the final values to the POKEY hardware registers.

-   **Control Flow Diagram (MODSND Routine):**

    ```mermaid
    graph TD
        A[Start MODSND for Frame] --> B{For each Channel (0 to 9)};
        B --> C{Channel Active? (POINT != 0)};
        C -- No --> B;
        C -- Yes --> D{Frame Counter Expired?};
        D -- No --> E[Update POKEY Register with CURRENT value];
        D -- Yes --> F{Sequence Step Counter Expired?};
        F -- No --> G[Apply CHANGE to CURRENT value];
        G --> H[Reset FRAMES Counter];
        H --> E;
        F -- Yes --> I[Load Next 4-Byte Sequence from Sound Data];
        I --> J{End of Sound? (Sequence is 0,0)};
        J -- No --> K[Load new STVAL, FRCNT, CHANGE, NUMBER];
        K --> H;
        J -- Yes --> L[Deactivate Channel (POINT = 0)];
        L --> E;
        E --> B;
    ```

---

### 1.3. Data Structures and Storage

The engine relies on two main categories of data: the static sound definitions in ROM and the dynamic channel state variables in RAM.

#### 1.3.1. Sound Definition Format (ROM)

Each sound effect is composed of one or more 4-byte sequences. This format is the core of the engine's data-driven design.

| Offset | Name     | Size  | Type  | Description                                                                                             |
| :----- | :------- | :---- | :---- | :------------------------------------------------------------------------------------------------------ |
| `+0`   | `STVAL`  | 1-byte| u8    | The initial value to write to the POKEY register for this sequence.                                     |
| `+1`   | `FRCNT`  | 1-byte| u8    | The number of frames to wait *before* the next change. A value of `N` holds for `N+1` frames.            |
| `+2`   | `CHANGE` | 1-byte| i8    | A signed 8-bit value to add to the `CURRENT` register value at each step.                               |
| `+3`   | `NUMBER` | 1-byte| u8    | The total number of steps in this sequence. A value of `N` means the sequence has `N+1` different values. |

-   **Sequence Termination:** A sequence is considered finished when its step counter (`COUNT`) expires. The driver then reads the next 4-byte packet. A sound effect is terminated by a special sequence of `0,0`.
-   **Looping:** A sequence of `X, 0` where `X` is non-zero is a loop command. `X` is a 16-bit offset from the `SOUND` base address, divided by 2, pointing to the location to restart the sound from.

#### 1.3.2. Channel State Variables (RAM)

These arrays, located at the start of `ALSOUN.MAC`'s RAM allocation, track the state of the 10 logical channels.

| Label     | Size      | Description                                                                      |
| :-------- | :-------- | :------------------------------------------------------------------------------- |
| `POINT`   | 10 bytes  | For each channel, an offset into the `SOUND` data table. A value of 0 means the channel is inactive. |
| `CURRENT` | 10 bytes  | The current output value for the channel, which will be written to a POKEY register. |
| `FRAMES`  | 10 bytes  | A countdown timer (in frames) until the next change in the channel's `CURRENT` value. |
| `COUNT`   | 10 bytes  | A countdown of the number of steps remaining in the current 4-byte sequence.      |

#### 1.3.3. Sound ID Lookup Table (PNTRS)

This is a master table that maps logical Sound IDs to the actual sound data. Game code does not reference sound data addresses directly, but uses these IDs.

-   **Structure:** The `PNTRS` table is a series of 6-byte entries. Each entry corresponds to one sound effect. The `OFFSET` macro builds this table.
-   **Entry Format:** Each of the 6 bytes is an offset pointer into the `SOUND` data region. The engine seems to use pairs of channels for each sound (e.g., Frequency and Amplitude/Control), so one 6-byte entry can define up to 3 channel pairs. A zero value indicates the channel is not used for that sound effect.
-   **Sound IDs:** The code defines symbolic names for these IDs, for example:
    -   `SIDDI`: Player Dies
    -   `SIDEX`: Enemy Explosion
    -   `SIDLA`: Player Fires (Launch)
    -   `SIDPU`: Pulsar sound

---

### 1.4. Algorithms and Processes

#### 1.4.1. `INISOU` (Sound Initialization)

This routine is called once at startup to prepare the audio system.

1.  Stop all POKEY sound generation by writing to control registers.
2.  Set the master POKEY audio control registers (`AUDCTL`) to their default game values.
3.  Iterate through all 10 logical channels and zero out their state variables in RAM (`POINT`, `CURRENT`, `FRAMES`, `COUNT`). This marks all channels as inactive.

#### 1.4.2. `SNDON` (Start New Sound)

This routine triggers a new sound effect.

1.  **Input:** Expects a Sound ID in the accumulator (e.g., `LDA I,SIDDI`).
2.  Check `QSTATUS` flag. If in Attract Mode, sound is suppressed and the routine exits.
3.  Use the Sound ID as an index into the `PNTRS` lookup table.
4.  For each non-zero channel pointer found in the `PNTRS` entry for this sound:
    a.  Set the `POINT` state variable for that channel to the pointer value.
    b.  Initialize `FRAMES` and `COUNT` to 1. This "wakes up" the channel. `MODSND` will handle loading the real data on the next frame.

#### 1.4.3. `MODSND` (Update Playing Sounds)

This is the core of the engine, executed every frame.

1.  Iterate through all 10 logical channels from 9 down to 0.
2.  For each channel, check if its `POINT` value is non-zero. If it is zero, the channel is inactive, so skip to the next.
3.  Decrement the channel's `FRAMES` counter.
4.  **If `FRAMES` is not zero:** The channel holds its current value. Proceed to step 8.
5.  **If `FRAMES` is zero:** It's time for a value change.
    a.  Decrement the `COUNT` (sequence step) counter.
    b.  **If `COUNT` is not zero:**
        i.  The current 4-byte sequence is still active.
        ii. Reload `FRAMES` from the sequence's `FRCNT` field.
        iii.Add the sequence's `CHANGE` value to the `CURRENT` state variable.
        iv. Proceed to step 8.
    c.  **If `COUNT` is zero:**
        i.  The current 4-byte sequence is complete. Advance `POINT` by 2 (to point to the next 4-byte sequence, since addresses are 16-bit).
        ii. Load the new 4-byte sequence from the `SOUND` data table.
        iii.Check the new `FRCNT` value. If it is 0, this is a command.
            - If `STVAL` is also 0, this is the end of the sound. Set `POINT` to 0 to deactivate the channel.
            - If `STVAL` is non-zero, this is a loop. Set `POINT` to `STVAL` (as a restart address) and re-evaluate from step 5c.
        iv. If the new sequence is valid data (not a command), load `CURRENT`, `COUNT`, and `FRAMES` from the new sequence's `STVAL`, `NUMBER`, and `FRCNT` fields respectively.
6.  Read the final `CURRENT` value for the channel.
7.  Write (`POKE`) the `CURRENT` value to the appropriate POKEY hardware register (`AUDFx` or `AUDCx`). The channel index determines which register is written to.
8.  Loop to the next channel.

---

### 1.5. Hardware Integration: POKEY Registers

The engine directly controls two POKEY chips. The base addresses are defined as `POKEY` and `POKEY2`. The following registers are used.

| Address Offset | Register | POKEY Channels | Description & Usage by Engine                                                                                                                                      |
| :------------- | :------- | :------------- | :----------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `+00`          | `AUDF1`  | 0, 4           | **Audio Frequency 1.** The `CURRENT` value for logical channels 0 and 4 is written here. Controls the frequency (pitch) of POKEY channels 1 on each chip.                 |
| `+01`          | `AUDC1`  | 1, 5           | **Audio Control 1.** The `CURRENT` value for logical channels 1 and 5 is written here. Controls volume (bits 0-3) and distortion/noise content (bits 5-7).            |
| `+02`          | `AUDF2`  | 2, 6           | **Audio Frequency 2.** For logical channels 2 and 6.                                                                                                               |
| `+03`          | `AUDC2`  | 3, 7           | **Audio Control 2.** For logical channels 3 and 7.                                                                                                                 |
| `+04`          | `AUDF3`  | -              | **Audio Frequency 3.** (Not explicitly mapped to a logical channel in the main loop, but available).                                                                 |
| `+05`          | `AUDC3`  | -              | **Audio Control 3.** (Not explicitly mapped).                                                                                                                      |
| `+06`          | `AUDF4`  | -              | **Audio Frequency 4.** (Not explicitly mapped).                                                                                                                      |
| `+07`          | `AUDC4`  | -              | **Audio Control 4.** (Not explicitly mapped).                                                                                                                      |
| `+08`          | `AUDCTL` | All            | **Master Audio Control.** Set once by `INISOU` with a value from `AUDCV`/`AUDCV2`. This configures the master clock rate, 16-bit modes, and high-pass filters.        |

---

### 1.6. Example Sound Data

The following is the data for the player death sound (`DI1F`/`DI1A`), which uses two channels.

**Channel 1: Frequency (`DI1F`)**
```assembly
; STVAL, FRCNT, CHANGE, NUMBER
.BYTE 8,    4,      20,     $0A  ; Start at 8, wait 5f, add 20, 11 steps
.BYTE 8,    4,      1,      $09  ; Start at 8, wait 5f, add 1,  10 steps
.BYTE 16,   $0D,    4,      $0C  ; Start at 16, wait 14f, add 4, 13 steps
.BYTE 0,0                        ; End of sound
```
*This channel creates a complex, stepped frequency slide.*

**Channel 2: Amplitude/Control (`DI1A`)**
```assembly
; STVAL, FRCNT, CHANGE, NUMBER
.BYTE $08,  4,      0,      $0A  ; Start at vol 8, wait 5f, no change, 11 steps
.BYTE $68,  4,      0,      $09  ; Start at vol 8 + noise, wait 5f, no change, 10 steps
.BYTE $68,  12,     $FF,    $09  ; Start at vol 8 + noise, wait 13f, subtract 1, 10 steps
.BYTE 0,0                        ; End of sound
```
*This channel sets an initial volume, then adds noise and fades out the volume.*

---

### 1.7. Known Limitations and Quirks

-   **Logical vs. Physical Channels:** The driver supports 10 logical channels, but the hardware only has 8 physical channels. The main update loop in `MODSND` writes to POKEY registers based on a simple channel index mapping, effectively mapping logical channels 0-7 to physical channels 1-4 on each of the two chips. Logical channels 8 and 9 are not used.
-   **Attract Mode Silence:** The `SNDON` routine has a hardcoded check to suppress all new sound triggers if the game is in Attract Mode (`QSTATUS` flag is set). Sounds that are already playing when the mode is entered will continue until they finish.
-   **No Sound Prioritization:** The engine has no concept of sound priority. If more sound effects are triggered than there are available channels, the last sound triggered will overwrite the data pointers for the channels it uses, cutting off the previous sound. 