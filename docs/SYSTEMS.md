# Tempest: Core Systems Documentation

This document provides a detailed overview of the core systems and subsystems that power Tempest. Each section covers a major component, from the low-level hardware interfaces to the high-level graphics and audio engines.

---

## 1. Vector Graphics Engine

This document details the architecture and function of the Atari Vector Generator (VG) hardware and the software systems that drive it. The VG is a specialized co-processor that operates independently of the main CPU. Its sole purpose is to read a "display list" program from RAM and draw the corresponding vectors on the screen.

---

### 1.1. Purpose and Overview

The Vector Graphics Engine is the final stage in the rendering pipeline. After the [Mathbox](#2-mathbox-co-processor) has transformed all 3D world coordinates into 2D screen coordinates, the 6502 assembles a program for the VG to execute. This program, called a display list, contains simple instructions like "move the beam here," "draw a line there," and "jump to a subroutine." The VG reads this program and translates it into the analog signals needed to guide the CRT beam and draw the game's graphics.

-   **Context of Use:** The entire `ALDISP.MAC` module is dedicated to generating the display list. The main `DISPLAY` routine is called once per frame. It fills one of two RAM buffers with VG instructions. The [IRQ handler (`ALHARD.MAC`)](#3-hardware-abstraction--irq-handler-alhardmac) then "starts" the VG, pointing it to the newly created display list.
-   **Architecture:** The system is a classic pipelined display architecture. While the VG is busy drawing the contents of Buffer A for the current frame, the 6502 CPU and [Mathbox](#2-mathbox-co-processor) are busy creating the display list for the *next* frame in Buffer B. This prevents tearing and ensures a smooth, consistent frame rate.

---

### 1.2. The Graphics Pipeline

1.  **Game Logic ([`ALWELG.MAC`](./ALWELG.md)):** The state of all game objects ([Player](./ENTITIES.md#1-player-the-claw), [Enemies](./ENTITIES.md#4-common-enemy-ai-and-behavior), [Bullets](./ENTITIES.md#2-player-bullet)) is updated, determining their 3D positions in the world.
2.  **Mathbox Transformation ([`WORSCR`](#25-integration-and-programming-model)):** For each vertex of each object, the 3D world coordinates are sent to the [Mathbox](#2-mathbox-co-processor), which applies rotation, translation, and perspective projection, returning 2D screen coordinates.
3.  **Display List Generation (`ALDISP.MAC`):** The main `DISPLAY` routine iterates through all visible objects. It takes the 2D screen coordinates from the Mathbox and uses the macros in `VGMC.MAC` to write a sequence of binary instructions into a block of RAM (`VECRAM`). This block of instructions is the display list.
4.  **Vector Generation (Hardware):** The 6502 commands the VG to start executing the newly created display list. The VG reads the instructions from `VECRAM` and draws the vectors on the monitor, completely independent of the CPU. The CPU is now free to begin calculating the next frame.

---

### 1.3. The Vector Generator Instruction Set

The "assembly language" for the Vector Generator is defined by a series of macros in `VGMC.MAC`. These macros translate human-readable commands into the 16-bit binary opcodes that the VG hardware understands.

| Mnemonic | Parameters        | Opcode (Binary)           | Description                                                                                                                                |
| :------- | :---------------- | :------------------------ | :----------------------------------------------------------------------------------------------------------------------------------------- |
| `VCTR`   | `DX, DY, ZZ`      | `01...` or `1...`         | **Draw Vector.** Draws a line from the current beam position to a new position `(current_X + DX, current_Y + DY)`. `ZZ` is the brightness (0-7). This is the most common instruction. It has a short (1-word) and long (2-word) form. |
| `JSRL`   | `LABEL`           | `1010...`                 | **Jump to Subroutine.** Pushes the current address onto the VG's stack and jumps to `LABEL`. Used extensively to draw game objects whose vector data is stored in ROM. |
| `RTSL`   | (none)            | `1100 0000 0000 0000`     | **Return from Subroutine.** Pops an address from the VG's stack and returns to it. The stack has 5 levels.                                |
| `JMPL`   | `LABEL`           | `1110...`                 | **Jump.** Unconditionally jumps to `LABEL`.                                                                                                |
| `STAT`   | `Z, HI, IN`       | `011...`                  | **Set Status.** Sets vector generator properties. `Z` is the master brightness. `HI`/`IN` control clipping functions.                         |
| `SCAL`   | `S, LS`           | `0111...`                 | **Set Scale.** Sets the drawing scale. `S` is a power-of-2 scale (0=full, 7=1/128th). `LS` is an optional linear scale factor.           |
| `CNTR`   | (none)            | `1000 0000 0100 0000`     | **Center Beam.** Resets the beam's position to the center of the screen (0,0).                                                              |
| `HALT`   | (none)            | `0010 0000 0000 0000`     | **Halt.** Stops the VG and sets the `MHALT` flag, which the 6502 can read. This is always the last instruction in a display list.        |

---

### 1.4. Hardware Interaction and Double Buffering

The 6502 controls the VG via a few key hardware registers and a double-buffering scheme to ensure flicker-free animation.

-   **`VGSTART` (`$D501`):** A write-only register. Writing any value to this address commands the VG to begin executing the display list currently pointed to by the internal vector program counter.
-   **`VGSTOP` (`$D500`):** A write-only register. Writing any value here immediately halts the VG.
-   **`MHALT` flag (in `IN1` register):** A read-only flag that the VG sets high when it executes a `HALT` instruction. The [IRQ handler](#3-hardware-abstraction--irq-handler-alhardmac) polls this bit to know when the VG is idle and ready for the next frame's display list.

-   **Double Buffering:**
    1.  The game allocates two buffers in RAM for display lists, `VECRAM` Buffer A and Buffer B.
    2.  In Frame 1, the `DISPLAY` routine builds the display list in Buffer A.
    3.  At the end of Frame 1, the IRQ points the VG hardware at Buffer A and issues a `VGSTART` command.
    4.  In Frame 2, while the VG is busy reading from Buffer A, the `DISPLAY` routine builds the next display list in Buffer B.
    5.  At the end of Frame 2, the IRQ sees that the VG has halted, points it at Buffer B, issues a `VGSTART`, and the cycle repeats.
    6.  The `SWAPVG` routine is responsible for swapping the buffer pointers.

---

### 1.5. Data Structures

-   **Shape Tables ([`ALVROM.MAC`](./ALVROM.md)):** The [vector data](./DATA_ASSETS.md#2-vector-shapes) for all game objects (the player's ship, enemies, text characters) is stored as subroutines in the ROM. A subroutine is simply a sequence of `VCTR` commands followed by an `RTSL`.
    -   Example: The player's ship shape is at the label `PTCURS`. To draw the ship, the `DISPLAY` routine just needs to add a single `JSRL PTCURS` instruction to the display list.

-   **Display List Buffers (`VECRAM`):** These are the blocks of RAM where the display list is assembled each frame before being handed off to the hardware. The list is a mix of `JSRL` instructions pointing to the [shape tables](./DATA_ASSETS.md#2-vector-shapes) in ROM and dynamically generated `VCTR` instructions for things like [laser fire](./ENTITIES.md#2-player-bullet) and explosions.

---

### 1.6. The `DISPLAY` Routine (`ALDISP.MAC`)

The `DISPLAY` routine is the master "compiler" for the vector engine. Its job is to generate a complete, valid display list program every frame.

**Execution Flow:**
1.  **Initialize:** It calls `INIMAT` to set up the Mathbox with the current camera view.
2.  **Select Buffer:** It selects the available (not currently being drawn) `VECRAM` buffer to write to.
3.  **Draw Objects:** It proceeds to call a series of subroutines, one for each type of object in the game:
    -   `DSPCUR`: Displays the player's cursor ("[Blaster](./ENTITIES.md#1-player-the-claw)").
    -   `DSPCHG`: Displays active [player shots](./ENTITIES.md#2-player-bullet).
    -   `DSPINV`: Displays the [enemies](./ENTITIES.md#4-common-enemy-ai-and-behavior) ("Invaders").
    -   `DSPEXP`: Displays explosions.
    -   `DSPNYM`: Displays the [Fuseballs](./ENTITIES.md#8-fuseball) and [Tankers](./ENTITIES.md#6-tanker) ("Nymphs").
    -   `INFO`: Displays scores, credits, and [messages](./DATA_ASSETS.md#1-text-and-language-data).
    -   `DSPWEL`: Displays the [well](./PLAYFIELD.md) itself.
4.  **How Objects are Drawn:** Each subroutine (`DSPCUR`, etc.) iterates through its list of active objects. For each object, it:
    a.  Gets the object's 3D world coordinates.
    b.  Calls [`WORSCR`](#25-integration-and-programming-model) to have the Mathbox transform the coordinates to 2D screen space.
    c.  Uses the `VGYABS` or `VGVTR1` macros to generate instructions to move the beam to the object's starting position.
    d.  Uses the `JSRL` macro to generate an instruction to call the object's [shape subroutine](./DATA_ASSETS.md#2-vector-shapes) from ROM.
5.  **Finalize:** After all objects are processed, it writes a final `HALT` instruction to the end of the display list. The list is now ready for the hardware.

---
---

## 2. Mathbox Co-Processor

This document provides a complete technical specification for the "Mathbox," a custom, high-speed, microcode-driven arithmetic co-processor designed by Atari to perform the complex 16-bit calculations required for *Tempest*'s 3D vector graphics.

---

### 2.1. Purpose and Overview

The 6502 main processor is an 8-bit CPU and is not powerful enough to perform the hundreds of 16-bit multiplications, divisions, and translations required to render a 3D vector scene in real-time. The purpose of the Mathbox is to offload these specific, repetitive calculations from the 6502.

The Mathbox is not a general-purpose CPU. It is a specialized piece of hardware with a small, fixed set of programs (microcode) designed to do one thing: transform 3D "world" coordinates into 2D "screen" coordinates, applying rotation, translation, and perspective projection.

-   **Context of Use:** The Mathbox is used exclusively by the [display list](#12-the-graphics-pipeline) generation logic in `ALDISP.MAC`. At the start of each frame, the `INIMAT` subroutine loads the current camera view matrix into the Mathbox. Then, for every vertex of every object to be drawn, the `WORSCR` subroutine sends the vertex's 3D coordinates to the Mathbox and waits for the 2D result.
-   **Interfaces:** The 6502 interacts with the Mathbox via a 32-byte block of memory-mapped I/O registers. Writing to specific addresses in this block sends data and simultaneously triggers pre-programmed calculation sequences.

---

### 2.2. Architecture and Structure

-   **Core:** The Mathbox is built using four AMD 2901 4-bit "bit-slice" processors, chained together to create a 16-bit ALU (Arithmetic Logic Unit).
-   **Microcode ROM:** It has its own internal 256-word x 24-bit program memory (uCODE ROM). This ROM contains the small, highly optimized programs that execute the mathematical formulas. The binary for this ROM is `MBUCOD.COM`, which is assembled from the source [`MBUCOD.V05`](./MBUCOD.V05.md).
-   **Host Communication:** The 6502 communicates with the Mathbox as a peripheral device. The Mathbox is idle until the 6502 writes to one of its command registers. It then executes its microcode program, and upon completion, sets a "done" flag that the 6502 can poll.

---

### 2.3. 6502 Host Interface (Memory-Mapped Registers)

The 6502 controls the Mathbox via the memory-mapped registers from address `$D100` to `$D11F`. Writing to a register triggers an action.

| Addr | Label   | R/W | Description & Action on Write                                                                                |
| :--- | :------ | :-- | :----------------------------------------------------------------------------------------------------------- |
| D100 | `STAT`  | R   | **Read Status:** D7=1 means busy, D7=0 means done.                                                           |
| D100 | `AL`    | W   | Write low byte of register `A` (used for `cos(angle)`).                                                        |
| D101 | `AH`    | W   | Write high byte of register `A`.                                                                             |
| D102 | `BL`    | W   | Write low byte of register `B` (used for `sin(angle)`).                                                        |
| D103 | `BH`    | W   | Write high byte of register `B`.                                                                             |
| D104 | `EL`    | W   | Write low byte of register `E` (Camera X position).                                                          |
| D105 | `EH`    | W   | Write high byte of register `E`.                                                                             |
| D106 | `FL`    | W   | Write low byte of register `F` (Camera Y position).                                                          |
| D107 | `FH`    | W   | Write high byte of register `F`.                                                                             |
| D108 | `XL`    | W   | Write low byte of register `X` (Input vertex X coordinate).                                                  |
| D109 | `XH`    | W   | Write high byte of register `X`.                                                                             |
| D10A | `YL`    | W   | Write low byte of register `Y` (Input vertex Y coordinate).                                                  |
| D10B | `YHSP`  | W   | **Write high byte of `Y` AND Start Perspective Divide:** The main command to transform a 3D point.             |
| D10C | `NL`    | W   | Write `N` (number of bits for division quotient).                                                            |
| D10D-10| `ZLL-ZHH`| W   | Write the 32-bit dividend for a division operation.                                                        |
| D111 | `YHSC`  | W   | **Write high byte of `Y` AND Start Clipping.**                                                                 |
| ...  | ...     |...  | *(Other registers are used for specific multiply/divide/debug operations as per `MBUDOC.DOC`)*                |
| D115 | `XPL`   | W   | Write low byte of result register `X'`.                                                                     |
| D116 | `XPH`   | W   | Write high byte of result register `X'`.                                                                    |
| D102 | `YLOW`  | R   | Read low byte of the 16-bit calculation result from the ALU output bus.                                      |
| D103 | `YHIGH` | R   | Read high byte of the 16-bit calculation result.                                                             |

---

### 2.4. Core Mathematical Operations

The Mathbox's microcode is pre-programmed to solve the core equations of 3D-to-2D projection.

#### 2.4.1. 2D Rotation and Translation

The primary transformation combines rotation and translation (moving the object relative to the camera). The resulting 2D screen coordinates `(x', y')` are calculated from the 3D world coordinates `(x, y, z)` and the camera's position `(e, f)` and rotation angle (`a=cos(angle)`, `b=sin(angle)`).

-   `x' = (x-e)*a - (y-f)*b`
-   `y' = (x-e)*b + (y-f)*a`

This calculation is triggered by the `SYM` or `YHSM` commands.

#### 2.4.2. Perspective Projection

To create the illusion of depth, the 2D coordinates are scaled based on their distance from the viewer (their Z-coordinate). This is a division operation. The final projected screen coordinate `(X`<sub>`p`</sub>`, Y`<sub>`p`</sub>`)` is calculated:

-   `Z`<sub>`eff`</sub> ` = (z - z_cam) * scale_factor` (Effective Z-depth)
-   `X`<sub>`p`</sub> ` = x' / Z`<sub>`eff`</sub>
-   `Y`<sub>`p`</sub> ` = y' / Z`<sub>`eff`</sub>

The `YHSP` command (`Write Y High and Start Perspective`) executes this entire sequence: it takes `(x,y,z)` as input, performs the rotation/translation to get `(x', y')`, and then performs the perspective divide to get the final on-screen coordinates, which are read back by the 6502.

---

### 2.5. Integration and Programming Model

The interaction between the 6502 and the Mathbox follows a clear, two-stage pattern every frame.

**Stage 1: `INIMAT` (Initialize Matrix)**
This subroutine is called once at the start of each frame's display list generation. It loads the "view matrix" into the Mathbox's registers.
```assembly
INIMAT:
    LDA EYEL      ; Load Eye/Camera Y position (low byte)
    STA EL        ; Store in Mathbox register E (low)
    LDA SYEH      ; Load Eye/Camera Y position (high byte)
    STA EH        ; Store in Mathbox register E (high)
    LDA EZEL      ; Load Eye/Camera Z position (low byte)
    STA FL        ; Store in Mathbox register F (low)
    LDA SZEH      ; Load Eye/Camera Z position (high byte)
    STA FH        ; Store in Mathbox register F (high)
    LDA ANGL      ; Load current view angle
    ASL           ; Multiply by 2 (word index)
    TAY           ; Use as index into sine/cosine tables
    LDA Y,SINTAB  ; Load sin(angle) from a [data table](./DATA_ASSETS.md#text-data)
    STA BL        ; Store in Mathbox register B
    LDA Y,COSTAB  ; Load cos(angle) from a [data table](./DATA_ASSETS.md#text-data)
    STA AL        ; Store in Mathbox register A
    ...
```

**Stage 2: `WORSCR` (World-to-Screen Conversion)**
This subroutine is called for *every single vertex* that needs to be drawn. It sends the 3D point to the Mathbox and retrieves the 2D result.

```assembly
; Input: PXL/H, PYL/H, PZL/H contain the 16-bit X, Y, Z world coordinates
WORSCR:
    LDA PXL       ; Load world X (low)
    STA XL        ; Store in Mathbox X register
    LDA PXH
    STA XH

    LDA PZL       ; Load world Z (low)
    STA YL        ; Store in Mathbox Y register (Note: Z is treated as Y for this command)
    LDA PZH
    STA YHSP      ; Store in Y High AND START THE PERSPECTIVE CALCULATION

    LDA PYL       ; Load world Y (low)
    STA ZLL       ; Store in the Mathbox dividend register
    LDA PYH
    STA ZLH

WAIT:             ; <<< --- BUSY-WAIT LOOP --- >>>
    LDA STAT      ; Read Mathbox status register
    BPL WAIT      ; Loop until D7 is 0 (done)

    LDA YLOW      ; Read result X' (low) from Mathbox output
    STA SXL       ; Store in screen coordinate X (low)
    LDA YHIGH     ; Read result X' (high)
    STA SXH

    JSR DXP       ; Command Mathbox to put Y' on its output bus

    LDA YLOW      ; Read result Y' (low)
    STA SYL       ; Store in screen coordinate Y (low)
    LDA YHIGH
    STA SYH
    RTS
```
The resulting screen coordinates (`SXL/H`, `SYL/H`) are then used to build the `VCTR` commands for the [Vector Generator](#1-vector-graphics-engine).

---
---

## 3. Hardware Abstraction & IRQ Handler (`ALHARD.MAC`)

This document details the hardware interface layer of *Tempest*. Unlike modern systems with a formal Hardware Abstraction Layer (HAL), *Tempest*'s hardware access is centralized within its **Interrupt Request (IRQ) Handler**, defined in `ALHARD.MAC`. This routine is the "heartbeat" of the game, executing at a fixed interval (tied to the VBLANK signal) to perform all time-critical I/O and background processing.

---

### 3.1. Purpose and Overview

The primary purpose of the IRQ handler is to service the 6502's hardware interrupt line. It ensures that critical tasks are performed consistently every frame, independent of the main game logic's execution path. This includes reading inputs, updating hardware timers, driving outputs like lamps, and calling other periodic subsystems.

The system's "abstraction" comes from the use of symbolic labels for hardware registers (defined in `ALCOMN.MAC`), rather than hardcoded memory addresses.

-   **Context of Use:** The IRQ is enabled at startup and runs continuously in the background. It is the sole mechanism for reading player controls and cabinet switches.
-   **Interfaces:** The IRQ handler is not called directly by game code. Instead, the 6502 CPU automatically transfers control to the `IRQ:` entry point when an interrupt occurs. It drives other systems by calling them via `JSR` (Jump to Subroutine), specifically `MODSND` and `MOOLAH`.

---

### 3.2. Architecture and Structure

The IRQ handler is a single, monolithic routine with a linear sequence of tasks.

1.  **Entry & State Preservation:** The routine begins at the `IRQ:` label. It immediately saves the CPU registers (A, X, Y) to the stack.
2.  **System Integrity Check:** It performs checks to see if the system has crashed (stack overflow, watchdog timeout). If a crash is detected, it forces a hardware `RESET`.
3.  **Input Polling & Processing:** It reads all memory-mapped input registers for the rotary controller and cabinet switches. It then performs software debouncing on the switch inputs.
4.  **Output Driving:** It updates the state of output registers controlling coin counters and cabinet lamps.
5.  **Subsystem Calls:** It calls the sound engine update (`MODSND`) and the coin handling logic (`MOOLAH`).
6.  **Timer Updates:** It increments all global frame and second counters.
7.  **Vector Generator Control:** It checks the status of the vector generator and restarts it for the next frame.
8.  **Exit & State Restoration:** It restores the CPU registers from the stack and executes an `RTI` (Return from Interrupt) instruction to pass control back to the main game code.

---

### 3.3. Hardware Register Map (Memory-Mapped I/O)

The following table lists the primary hardware registers accessed by the IRQ handler. These labels are defined in `ALCOMN.MAC`.

| Address | Label     | R/W | Description & Function in IRQ                                                                                                       |
| :------ | :-------- | :-- | :------------------------------------------------------------------------------------------------------------------------------------ |
| `$D000`   | `ALLPOT`  | R   | **Read POKEY 1 Potentiometers.** Used to read the state of the player's rotary controller (spinner).                                 |
| `$D008`   | `ALLPO2`  | R   | **Read POKEY 2 Potentiometers.** Also used for the rotary controller.                                                                 |
| `$D010`   | `IN1`     | R   | **Read Input Port 1.** Reads the state of Coin, Slam, Test, and Player Start switches. Also contains the Vector Generator `HALT` flag. |
| `$D40E`   | `WTCHDG`  | W   | **Write Watchdog Reset.** The IRQ "kicks" the watchdog by writing to this address every frame to prevent a hardware reset.          |
| `$D011`   | `POTGO`   | W   | **Start Potentiometer Scan.** Strobed every frame to begin the analog-to-digital conversion for the rotary controller.             |
| `$C400`   | `OUT0`    | W   | **Write Output Port 0.** Used to control the physical coin counters (`MLCCNT`, `MMCCNT`, `MRCCNT`).                                  |
| `$C401`   | `OUTANK`  | W   | **Write "Tank" Output Port.** Controls the 1-Player (`MLED1`) and 2-Player (`MLED2`) start button lamps.                             |
| `$D500`   | `VGSTOP`  | W   | **Vector Generator Stop.** Writing any value halts the vector generator.                                                              |
| `$D501`   | `VGSTART` | W   | **Vector Generator Start.** Writing any value begins execution of the display list pointed to by `VECADR`.                            |

---

### 3.4. Algorithms and Processes

#### 3.4.1. Switch Debouncing Algorithm

The IRQ implements a robust, bitwise software debounce for the cabinet switches read from `IN1`.

1.  The raw, noisy input from `IN1` is read.
2.  The previous frame's raw input is stored in `DBSW`.
3.  The final, clean state from the previous frame is in `SWSTAT`.
4.  The new clean state is calculated using bitwise logic that approximates a state machine for each bit:
    `new_SWSTAT = ((raw_input AND prev_raw_input) OR prev_SWSTAT) AND (prev_raw_input OR raw_input)`
5.  This ensures a switch must be in the same state for two consecutive frames to be considered stable, effectively filtering out electrical noise.
6.  The final debounced state is used to update `SWFINA`, which contains single-frame trigger flags for game logic.

#### 3.4.2. Rotary Controller (Spinner) Algorithm

1.  `POTGO` is strobed to start the read process on both POKEY chips.
2.  The 4-bit values from `ALLPOT` and `ALLPO2` are read. These represent a Gray code encoding of the spinner's position.
3.  The previous frame's 4-bit value is stored in `OTB`.
4.  The new value is compared to the old value (`SBC OTB`) to get a signed delta, indicating direction and magnitude of rotation.
5.  This delta is added to a velocity accumulator, `TBHD`, which is the final value used by the main game logic to move the player's ship.

#### 3.4.3. Watchdog and System Integrity

-   **Watchdog:** The first instruction in the IRQ after saving registers is `STA WTCHDG`. If the IRQ fails to execute for any reason (e.g., a crash in the main loop), this write will not occur, and the external hardware watchdog timer will expire, forcing a full system reset.
-   **Stack/Timing Check:** Before kicking the watchdog, the IRQ handler checks the stack pointer's depth (`TSX`, `CPX I,0D0`) and a frame timer (`FRTIMR`). If the stack has grown too deep or too much time has passed since the last IRQ, it assumes the CPU is stuck in an invalid state and forces a `RESET` itself. This prevents the game from freezing indefinitely.

---

### 3.5. Interrupt Handling

-   **Vector:** The 6502's hardware IRQ vector at `$FFFE`/`$FFFF` is not pointed directly at the game's IRQ handler. Instead, it's likely pointed to by the OS or hardware default, which then jumps to the address specified in the `.VCTRS` directive at the end of `ALHARD.MAC`.
-   **Vectors Table:** The line `.VCTRS 0DFFA,IRQ,RESET,IRQ` defines the addresses to be placed in the hardware vector table. It specifies that a normal IRQ should jump to the `IRQ:` label, and a `RESET` (or a detected crash) should jump to the `RESET` label.

---

### 3.6. Integration with Other Systems

The IRQ handler is the master clock for several other systems, which it drives via direct subroutine calls.

-   `JSR MOOLAH`: Called every frame to process coin insertion logic. See [Coin and Credit Logic](#4-coin-and-credit-logic-coin65-library).
-   `JSR MODSND`: Called every frame to update and play all active sound effects. See [Sound Engine](#8-sound-engine-alsounmac). This ensures that sounds are generated smoothly and are not affected by varying performance in the main game loop.
-   **Game Timers:** The IRQ increments `FRTIMR`, `INTCT`, `SECOUL`/`M`/`H`, and `SECOPL`/`M`/`H`. The main game logic reads these memory locations to control gameplay events, [Attract Mode](./ATTRACT_MODE.md) sequences, and bookkeeping.
-   **Vector Generator:** The IRQ monitors the `VG DONE` bit from `IN1`. When the VG signals it has finished rendering, the IRQ handler resets it (`VGSTOP`, `VGSTART`), kicking off the rendering process for the *next* frame's [display list](#12-the-graphics-pipeline). This creates a pipelined graphics architecture.

### 3.7. EAROM (Non-Volatile Storage)

The EAROM (Electrically Alterable Read-Only Memory) serves as the game's non-volatile "database." It has a simple, fixed schema:
-   **Block 0:** Initials Table (9 bytes)
-   **Block 1:** High Score Table (11 bytes)
-   **Block 2:** Bookkeeping Statistics (size varies)

Each block is protected by a checksum to ensure data integrity. The `ALEARO.MAC` driver manages all access to this data.

---
---

## 4. Coin and Credit Logic (`COIN65` Library)

This document details the coin acceptance and credit management system of *Tempest*. The logic is not bespoke to the game but is an implementation of Atari's generic **`COIN65`** library, a reusable module for the 6502 processor. The primary entry point to this library, called every frame by the IRQ handler, is the `MOOLAH` subroutine. The code for this library is in `ALCOIN.MAC`.

---

### 4.1. Purpose and Overview

The purpose of the `COIN65` library is to provide a robust, reliable, and cheat-resistant system for processing inputs from physical coin mechanisms and managing the player's credits. It handles switch debouncing, enforces operator-defined pricing, and provides the main game loop with a simple, clean credit count.

-   **Context of Use:** The `MOOLAH` routine is called on every single frame by the [IRQ handler](#3-hardware-abstraction--irq-handler-alhardmac). It continuously monitors the coin slot switches. The main game logic in `ALEXEC.MAC` reads the output of this system (`$$CRDT`) to determine if a new game can be started.
-   **Interfaces:**
    -   **Input:** The library reads the state of the physical coin switches from `$COINA`, which is populated by the IRQ handler. It also reads pricing and bonus settings from other memory locations set by the operator.
    -   **Output:** The library's primary output is the 8-bit master credit count stored in the `$$CRDT` variable. It also drives timers for the physical, electro-mechanical coin counters in the cabinet.

---

### 4.2. Architecture and Structure

The system is architected as a "black box" library that is serviced once per frame.

1.  **[IRQ Handler (`ALHARD.MAC`)](#3-hardware-abstraction--irq-handler-alhardmac):** Reads the raw hardware switch state and calls `MOOLAH`.
2.  **`MOOLAH` Subroutine (`COIN65` Library):** Executes its internal state machine, performing debouncing and credit calculation. It updates the `$$CRDT` variable.
3.  **Game Logic (`ALEXEC.MAC`):** Reads `$$CRDT` when the player presses a Start button.
4.  **Display Logic (`ALSCO2.MAC`):** Reads `$$CRDT` and `$CMODE` to show the player the current credit status.

---

### 4.3. Data Structures and Storage (Zero Page RAM)

The `COIN65` library and the surrounding game logic rely on a set of variables in the 6502's Zero Page for communication and state management.

| Label    | Address | Description                                                                                                                                                             |
| :------- | :------ | :---------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `$$CRDT` | `$0006`   | **Master Credit Count:** The primary output of the system. An 8-bit value holding the number of credits the player has.                                                 |
| `$COINA` | `$0008`   | **Coin Input State:** A bitfield where the IRQ handler places the raw state of the coin switches. The `MOOLAH` routine reads this as its primary input.               |
| `$CMODE` | `$0009`   | **Coin Mode:** An operator-set value (from DIP switches) that determines the pricing. e.g., 0=Free Play, 1=1 Coin/2 Plays, 2=1 Coin/1 Play, 3=2 Coins/1 Play.       |
| `$CNSTT` | `$000E`   | **Coin Status/Timers:** A 3-byte array used by the debouncing state machine. Each byte tracks the stability of a coin switch over several frames.                     |
| `$PSTSL` | `$0011`   | **Post-Slam Timers:** A 3-byte array of timers used by the anti-cheat logic to invalidate a coin if a slam occurs immediately after insertion.                        |
| `$LMTIM` | `$000D`   | **Pre-Slam Lockout Timer:** A master timer that locks out the entire coin system for several seconds if a slam occurs, preventing "slamming for credit."              |
| `$CCTIM` | `$0014`   | **Counter Timers:** A 3-byte array of timers that control the duration of the pulse sent to the physical, electro-mechanical coin counters inside the cabinet.         |
| `$CNCT`  | `$0018`   | **Coin Count (Units):** A temporary counter used when the game is in "units" pricing mode.                                                                          |
| `$BCCNT` | `$0017`   | **Bonus Coin Counter:** Tracks coins inserted towards earning a bonus credit.                                                                                           |
| `$BC`    | `$0019`   | **Bonus Credits:** The number of bonus credits earned but not yet used.                                                                                                 |

---

### 4.4. Algorithms and Processes (The `MOOLAH` Routine)

Based on the `COIN65.MAC` source documentation, the `MOOLAH` routine executes the following logic every frame for each coin mechanism:

1.  **Check for Slam Lockout:** It first checks the master slam timer (`$LMTIM`). If this timer is active, the entire routine is skipped, preventing any coin processing.
2.  **Debounce Coin Switch:** It runs a sophisticated debouncing state machine for each coin switch.
    -   A coin switch signal must be continuously present for a specific duration (e.g., >16ms) to be considered potentially valid. This filters out transient electrical noise.
    -   The signal must also not be present for *too* long (e.g., <800ms). This detects a stuck switch and prevents it from awarding infinite credits.
    -   A valid coin event is only registered on the clean transition from "off" to "on" and back to "off".
3.  **Check for Post-Coin Slam:** If a debounce-valid coin is detected, it checks the post-slam timer (`$PSTSL`). If a slam occurred within a few frames of the coin being validated, the coin is ignored, but the system is not locked out.
4.  **Award Credit:** If a coin passes all checks:
    -   The routine reads the pricing mode from `$CMODE`.
    -   It uses a lookup table to determine how many "units" the specific coin mechanism is worth and adds this to the `$CNCT` temporary counter.
    -   It processes bonus logic via `$BCCNT` if applicable.
    -   It then calls the `$CNVRT` (Convert) subroutine.
5.  **Convert Units to Credits (`$CNVRT`):**
    -   This routine compares the number of units in `$CNCT` against the units required for a credit in the current `$CMODE`.
    -   If enough units have been accumulated, it decrements `$CNCT` by the required amount and increments the final `$$CRDT` variable.
    -   It also handles "2 for 1" pricing, where it might increment `$$CRDT` twice.
6.  **Pulse Mechanical Counter:** If configured, it activates a timer in `$CCTIM` to send an electrical pulse to the correct physical coin meter inside the cabinet.

---

### 4.5. Configuration and Pricing Modes

The operator can configure the coin logic via DIP switches, which are read by the game's diagnostic routines and stored in `$CMODE`.

| `$CMODE` Value | Pricing String        | Behavior                                        |
| :------------- | :-------------------- | :---------------------------------------------- |
| `0`            | FREE PLAY             | The game forces `$$CRDT` to 2.                  |
| `1`            | 1 COIN 2 PLAYS        | *Technically supported but not a default option.* |
| `2`            | 1 COIN 1 PLAY         | Standard single-credit mode.                    |
| `3`            | 2 COINS 1 PLAY        | Requires two coin events to award one credit.     |
| *Other*        | *Flexible Units Mode* | Uses the unit counters for complex pricing.       |

---

### 4.6. Integration with Other Systems

-   **Called By ([`ALHARD.MAC`](#3-hardware-abstraction--irq-handler-alhardmac)):** The `MOOLAH` routine is called every frame from the IRQ handler. This is its only entry point.
-   **Read By (`ALEXEC.MAC`):** The main game loop checks `$$CRDT` in the `PROCRE` subroutine (part of the [`CGETINI` state](./GAME_STATE_FLOW.md#10-cgetini---get-player-initials)) when a start button is pressed. If credits are available (`$$CRDT > 0`), it decrements the count and starts a new game.
-   **Displayed By (`ALSCO2.MAC`):** The `DSPCRD` routine reads `$CMODE` to display the pricing rule text, and reads `$$CRDT` to display the number of credits available to the player.

---
---

## 5. Game State Machine

The core logic of Tempest is managed by a classic state machine. This system ensures that the game is always in a single, well-defined state (e.g., Attract Mode, Gameplay, Game Over).

The complete, fine-grained documentation for every state and the transitions between them can be found in the **[Game State Flow](./GAME_STATE_FLOW.md)** document.

---
---

## 6. Procedural Generation Engine

Tempest does not have a single, monolithic "procedural generation engine." Instead, it employs procedural generation as a core design philosophy in several different subsystems to create complex behavior and content from a small amount of data.

-   **Playfield Generation:** The shape of the "well" for each level is generated by selecting and combining pre-defined [vector shapes](./DATA_ASSETS.md#2-vector-shapes) from a master table. This system is documented in detail in the [Playfield documentation](./PLAYFIELD.md#3-procedural-generation-and-level-selection).

-   **Level & Wave Behavior:** The difficulty and enemy composition of each wave are not individually hardcoded. Instead, a powerful table-driven system modifies a set of base parameters according to the current wave number. This allows for a smooth difficulty curve over 99+ levels with very little data. This system is documented in the [Level Data documentation](./LEVEL_DATA.md).

-   **Audio Synthesis:** All sound effects are generated procedurally by the [Sound Engine](#8-sound-engine-alsounmac). Instead of storing large audio samples, the game stores small, procedural "recipes" that the engine uses to create sounds on the fly.

---
---

## 7. Spike System

The "Spike" is a unique environmental hazard created by the [Spiker](./ENTITIES.md#7-spiker) enemy. The system that manages this hazard is not a standalone module but an emergent behavior resulting from the interaction between the Spiker's AI and the playfield's collision data.

The complete technical breakdown of how the Spike is created and how it affects the player can be found in the **[Spike Collision System section of the Playfield documentation](./PLAYFIELD.md#62-the-spike-collision-system)**.

---
---

## 8. Sound Engine (`ALSOUN.MAC`)

This document details the architecture and function of the Tempest sound engine. It is a complete, data-driven, procedural audio subsystem responsible for generating all in-game sound effects.

---

### 8.1. Purpose and Overview

The purpose of the sound engine is to provide a flexible, memory-efficient system for generating complex audio events on the Atari POKEY sound chip. Unlike sample-based audio, this engine synthesizes sounds in real-time by procedurally modulating the POKEY chip's registers frame-by-frame according to predefined data patterns.

-   **Context of Use:** The engine is used throughout the game to generate all audio, from the [player's](./ENTITIES.md#1-player-the-claw) laser fire and explosions to enemy sound effects and UI feedback clicks. The [main game loop](./GAME_STATE_FLOW.md#4-cplay---gameplay) calls the `MODSND` routine every frame to continue sound playback, and various game logic routines call `SNDON` to trigger new sounds.
-   **Interfaces:** The engine exposes three primary subroutines to the rest of the game code:
    -   `INISOU`: Initializes the two POKEY sound chips and all software channel states. Called once at game startup.
    -   `SNDON`: Triggers a new sound effect. The caller loads the desired Sound ID into the accumulator and jumps to this routine.
    -   `MODSND`: The main update routine. It is called once per frame (e.g., from the VBLANK interrupt) to advance the state of all currently playing sounds.

---

### 8.2. Architecture and Structure

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

### 8.3. Data Structures and Storage

The engine relies on two main categories of data: the static sound definitions in ROM and the dynamic channel state variables in RAM.

#### 8.3.1. Sound Definition Format (ROM)

Each sound effect is composed of one or more 4-byte sequences. This format is the core of the engine's data-driven design.

| Offset | Name     | Size  | Type  | Description                                                                                             |
| :----- | :------- | :---- | :---- | :------------------------------------------------------------------------------------------------------ |
| `+0`   | `STVAL`  | 1-byte| u8    | The initial value to write to the POKEY register for this sequence.                                     |
| `+1`   | `FRCNT`  | 1-byte| u8    | The number of frames to wait *before* the next change. A value of `N` holds for `N+1` frames.            |
| `+2`   | `CHANGE` | 1-byte| i8    | A signed 8-bit value to add to the `CURRENT` register value at each step.                               |
| `+3`   | `NUMBER` | 1-byte| u8    | The total number of steps in this sequence. A value of `N` means the sequence has `N+1` different values. |

-   **Sequence Termination:** A sequence is considered finished when its step counter (`COUNT`) expires. The driver then reads the next 4-byte packet. A sound effect is terminated by a special sequence of `0,0`.
-   **Looping:** A sequence of `X, 0` where `X` is non-zero is a loop command. `X` is a 16-bit offset from the `SOUND` base address, divided by 2, pointing to the location to restart the sound from.

#### 8.3.2. Channel State Variables (RAM)

These arrays, located at the start of `ALSOUN.MAC`'s RAM allocation, track the state of the 10 logical channels.

| Label     | Size      | Description                                                                      |
| :-------- | :-------- | :------------------------------------------------------------------------------- |
| `POINT`   | 10 bytes  | For each channel, an offset into the `SOUND` data table. A value of 0 means the channel is inactive. |
| `CURRENT` | 10 bytes  | The current output value for the channel, which will be written to a POKEY register. |
| `FRAMES`  | 10 bytes  | A countdown timer (in frames) until the next change in the channel's `CURRENT` value. |
| `COUNT`   | 10 bytes  | A countdown of the number of steps remaining in the current 4-byte sequence.      |

#### 8.3.3. Sound ID Lookup Table (PNTRS)

This is a master table that maps logical [Sound IDs](./DATA_ASSETS.md#3-sound-data) to the actual sound data. Game code does not reference sound data addresses directly, but uses these IDs.

-   **Structure:** The `PNTRS` table is a series of 6-byte entries. Each entry corresponds to one sound effect. The `OFFSET` macro builds this table.
-   **Entry Format:** Each of the 6 bytes is an offset pointer into the `SOUND` data region. The engine seems to use pairs of channels for each sound (e.g., Frequency and Amplitude/Control), so one 6-byte entry can define up to 3 channel pairs. A zero value indicates the channel is not used for that sound effect.
-   **Sound IDs:** The code defines symbolic names for these IDs, for example:
    -   `SIDDI`: [Player](./ENTITIES.md#1-player-the-claw) Dies
    -   `SIDEX`: [Enemy](./ENTITIES.md#4-common-enemy-ai-and-behavior) Explosion
    -   `SIDLA`: [Player Fires](./ENTITIES.md#2-player-bullet) (Launch)
    -   `SIDPU`: [Pulsar](./ENTITIES.md#9-pulsar) sound

---

### 8.4. Algorithms and Processes

#### 8.4.1. `INISOU` (Sound Initialization)

This routine is called once at startup to prepare the audio system.

1.  Stop all POKEY sound generation by writing to control registers.
2.  Set the master POKEY audio control registers (`AUDCTL`) to their default game values.
3.  Iterate through all 10 logical channels and zero out their state variables in RAM (`POINT`, `CURRENT`, `FRAMES`, `COUNT`). This marks all channels as inactive.

#### 8.4.2. `SNDON` (Start New Sound)

This routine triggers a new sound effect.

1.  **Input:** Expects a [Sound ID](./DATA_ASSETS.md#3-sound-data) in the accumulator (e.g., `LDA I,SIDDI`).
2.  Check `QSTATUS` flag. If in [Attract Mode](./ATTRACT_MODE.md), sound is suppressed and the routine exits.
3.  Use the [Sound ID](./DATA_ASSETS.md#3-sound-data) as an index into the `PNTRS` lookup table.
4.  For each non-zero channel pointer found in the `PNTRS` entry for this sound:
    a.  Set the `POINT` state variable for that channel to the pointer value.
    b.  Initialize `FRAMES` and `COUNT` to 1. This "wakes up" the channel. `MODSND` will handle loading the real data on the next frame.

#### 8.4.3. `MODSND` (Update Playing Sounds)

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

### 8.5. Hardware Integration: POKEY Registers

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

### 8.6. Example Sound Data

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

### 8.7. Known Limitations and Quirks

-   **Logical vs. Physical Channels:** The driver supports 10 logical channels, but the hardware only has 8 physical channels. The main update loop in `MODSND` writes to POKEY registers based on a simple channel index mapping, effectively mapping logical channels 0-7 to physical channels 1-4 on each of the two chips. Logical channels 8 and 9 are not used.
-   **Attract Mode Silence:** The `SNDON` routine has a hardcoded check to suppress all new sound triggers if the game is in [Attract Mode](./ATTRACT_MODE.md) (`QSTATUS` flag is set). Sounds that are already playing when the mode is entered will continue until they finish.
-   **No Sound Prioritization:** The engine has no concept of sound priority. If more sound effects are triggered than there are available channels, the last sound triggered will overwrite the data pointers for the channels it uses, cutting off the previous sound. 