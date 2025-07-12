# Tempest: ALSOUN.MAC - Sound Engine

**File:** `ALSOUN.MAC`
**Purpose:** This module contains the complete sound engine for Tempest. It is a sophisticated, data-driven system responsible for generating all sound effects by controlling the two POKEY sound chips.

---

## 1. Sound Engine Architecture

Unlike a simpler sound engine that might have hard-coded instructions for each effect, Tempest's sound engine is a generic processor that interprets **sound data tables**. This allows for the creation of complex, evolving sounds (sirens, explosions, engine hums) without writing unique code for each one.

The engine has two main components:
1.  **Triggering (`SNDON`)**: A set of routines that activates a new sound effect in response to a game event (like the player firing a shot).
2.  **Processing (`MODSND`)**: A central routine, called every frame by the IRQ handler, that updates all currently active sounds and writes their new values to the POKEY hardware registers.

### 1.1. Sound Data Format

Every sound effect in the game is defined as a series of 4-byte "sequence descriptors." A single sound can be made of many such sequences. Each 4-byte descriptor defines one segment of a sound's evolution:

-   **Byte 0 (`STVAL`):** The **Starting Value** for this sequence (e.g., a high frequency for the start of a laser zap).
-   **Byte 1 (`FRCNT`):** The **Frame Count**, or the number of frames to wait before applying any change to the value.
-   **Byte 2 (`CHANGE`):** The **Change Value**, a signed number that is added to the current value on each update. A negative value creates a downward sweep, a positive value an upward sweep.
-   **Byte 3 (`NUMBER`):** The **Number of Changes** to apply in this sequence.

**Example:** A sequence of `(Start=200, Frames=1, Change=-5, Number=10)` would produce a tone that rapidly drops in pitch over 10 frames: 200, 195, 190, ... , 155.

A sound is terminated with a `(0, 0)` sequence. This data-driven approach allows for great flexibility in sound design.

### 1.2. Master Sound Table (`PNTRS`)

The `PNTRS` table is the master directory of all sounds in the game. It contains pointers to the sound data sequences for each effect. The game triggers sounds by using an ID (e.g., `SIDLA` for "Launch") which acts as an index into this table.

---

## 2. Triggering New Sounds (`SNDON`)

The `SNDON` subroutine is the primary entry point for starting a new sound effect.

-   **Input:** The Accumulator holds a Sound ID, which is an offset into the `PNTRS` table.
-   **Functionality:**
    1.  It uses the Sound ID to look up the pointers to the data for the requested sound effect.
    2.  It initializes the sound channel's state variables in RAM (`POINT`, `FRAMES`, `COUNT`) with the starting values from the first 4-byte sequence in the sound's data table.
    3.  This "activates" the sound, and the `MODSND` processing routine will begin updating it on the next frame.

### 2.1. Public Sound Interface

To make the code more readable, the game logic doesn't call `SNDON` directly. Instead, it calls one of several named entry points, whose only job is to load the correct Sound ID into the accumulator and then jump to `SNDON`.

-   **`SLAUNC`:** Plays the player's laser "launch" sound.
-   **`EXSNON`:** Plays a standard enemy explosion sound.
-   **`IPEXPL` / `CPEXPL`:** Plays the player's death explosion sound.
-   **`SBOING`:** Plays the sound of the player moving across a web segment.
-   **`SAUSON`:** Plays the "special score" sound (originally for the saucer in Asteroids).
-   **`ESLSON`:** Plays the sound of an enemy firing a shot.
-   **`SSLAMS`:** Plays the loud "slam" sound when the cabinet is tilted.

---

*(The `MODSND` processing routine, which is the heart of the engine, will be documented as the rest of the file is analyzed.)*

## 3. Sound Processing Engine (`MODSND`)

The `MODSND` subroutine is the workhorse of the sound engine. It is called on every single frame by the `IRQ` handler in `ALHAR2.MAC`. Its job is to update the state of every active sound channel and write the new values to the POKEY hardware.

**Execution Flow of `MODSND`:**
The routine iterates through all 10 available sound channels (`NCHANL`). For each channel that is currently active (i.e., its `POINT`er in RAM is not zero), it does the following:

1.  **Decrement Frame Counter:** It decrements the channel's `FRAMES` counter.
2.  **Check for Update:** If `FRAMES` reaches zero, it's time to update the sound.
3.  **Process Sequence:**
    -   It decrements the `COUNT` of changes left in the current 4-byte sequence.
    -   If the count is not yet zero, it adds the `CHANGE` value to the `CURRENT` output value and reloads the `FRAMES` counter.
    -   If the count *is* zero, the current sequence is finished. The engine advances the channel's `POINT`er to the next 4-byte sequence in the sound's data table and loads the new starting values.
4.  **Handle Termination:** If the new sequence is `(0, 0)`, the sound has finished playing. The engine deactivates the channel by setting its `POINT`er back to zero.
5.  **Write to Hardware:** After all logic is processed, the routine takes the channel's `CURRENT` value and writes it directly to the appropriate POKEY hardware register (e.g., `AUDF1`, `AUDC1` for frequency and control).

This process, repeated every frame for all active channels, is what generates the rich, evolving soundscape of the game.

## 4. Initialization (`INISOU`)

The `INISOU` subroutine is called once at the start of a new game. It resets the entire sound system to a clean state.
- It writes zeros to all POKEY frequency and control registers, silencing any sounds from a previous game.
- It clears out the sound engine's RAM variables (`POINT`, `CURRENT`, `FRAMES`, `COUNT`), ensuring all channels are marked as inactive.
- It sets the master audio control registers (`AUDCTL`) to their default configuration.

---

This concludes the documentation for `ALSOUN.MAC`. 