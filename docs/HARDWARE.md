# Tempest: Core Systems Documentation

## 2. Hardware Abstraction & IRQ Handler (`ALHARD.MAC`)

This document details the hardware interface layer of *Tempest*. Unlike modern systems with a formal Hardware Abstraction Layer (HAL), *Tempest*'s hardware access is centralized within its **Interrupt Request (IRQ) Handler**, defined in `ALHARD.MAC`. This routine is the "heartbeat" of the game, executing at a fixed interval (tied to the VBLANK signal) to perform all time-critical I/O and background processing.

---

### 2.1. Purpose and Overview

The primary purpose of the IRQ handler is to service the 6502's hardware interrupt line. It ensures that critical tasks are performed consistently every frame, independent of the main game logic's execution path. This includes reading inputs, updating hardware timers, driving outputs like lamps, and calling other periodic subsystems.

The system's "abstraction" comes from the use of symbolic labels for hardware registers (defined in `ALCOMN.MAC`), rather than hardcoded memory addresses.

-   **Context of Use:** The IRQ is enabled at startup and runs continuously in the background. It is the sole mechanism for reading player controls and cabinet switches.
-   **Interfaces:** The IRQ handler is not called directly by game code. Instead, the 6502 CPU automatically transfers control to the `IRQ:` entry point when an interrupt occurs. It drives other systems by calling them via `JSR` (Jump to Subroutine), specifically `MODSND` and `MOOLAH`.

---

### 2.2. Architecture and Structure

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

### 2.3. Hardware Register Map (Memory-Mapped I/O)

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

### 2.4. Algorithms and Processes

#### 2.4.1. Switch Debouncing Algorithm

The IRQ implements a robust, bitwise software debounce for the cabinet switches read from `IN1`.

1.  The raw, noisy input from `IN1` is read.
2.  The previous frame's raw input is stored in `DBSW`.
3.  The final, clean state from the previous frame is in `SWSTAT`.
4.  The new clean state is calculated using bitwise logic that approximates a state machine for each bit:
    `new_SWSTAT = ((raw_input AND prev_raw_input) OR prev_SWSTAT) AND (prev_raw_input OR raw_input)`
5.  This ensures a switch must be in the same state for two consecutive frames to be considered stable, effectively filtering out electrical noise.
6.  The final debounced state is used to update `SWFINA`, which contains single-frame trigger flags for game logic.

#### 2.4.2. Rotary Controller (Spinner) Algorithm

1.  `POTGO` is strobed to start the read process on both POKEY chips.
2.  The 4-bit values from `ALLPOT` and `ALLPO2` are read. These represent a Gray code encoding of the spinner's position.
3.  The previous frame's 4-bit value is stored in `OTB`.
4.  The new value is compared to the old value (`SBC OTB`) to get a signed delta, indicating direction and magnitude of rotation.
5.  This delta is added to a velocity accumulator, `TBHD`, which is the final value used by the main game logic to move the player's ship.

#### 2.4.3. Watchdog and System Integrity

-   **Watchdog:** The first instruction in the IRQ after saving registers is `STA WTCHDG`. If the IRQ fails to execute for any reason (e.g., a crash in the main loop), this write will not occur, and the external hardware watchdog timer will expire, forcing a full system reset.
-   **Stack/Timing Check:** Before kicking the watchdog, the IRQ handler checks the stack pointer's depth (`TSX`, `CPX I,0D0`) and a frame timer (`FRTIMR`). If the stack has grown too deep or too much time has passed since the last IRQ, it assumes the CPU is stuck in an invalid state and forces a `RESET` itself. This prevents the game from freezing indefinitely.

---

### 2.5. Interrupt Handling

-   **Vector:** The 6502's hardware IRQ vector at `$FFFE`/`$FFFF` is not pointed directly at the game's IRQ handler. Instead, it's likely pointed to by the OS or hardware default, which then jumps to the address specified in the `.VCTRS` directive at the end of `ALHARD.MAC`.
-   **Vectors Table:** The line `.VCTRS 0DFFA,IRQ,RESET,IRQ` defines the addresses to be placed in the hardware vector table. It specifies that a normal IRQ should jump to the `IRQ:` label, and a `RESET` (or a detected crash) should jump to the `RESET` label.

---

### 2.6. Integration with Other Systems

The IRQ handler is the master clock for several other systems, which it drives via direct subroutine calls.

-   `JSR MOOLAH`: Called every frame to process coin insertion logic.
-   `JSR MODSND`: Called every frame to update and play all active sound effects. This ensures that sounds are generated smoothly and are not affected by varying performance in the main game loop.
-   **Game Timers:** The IRQ increments `FRTIMR`, `INTCT`, `SECOUL`/`M`/`H`, and `SECOPL`/`M`/`H`. The main game logic reads these memory locations to control gameplay events, demo mode sequences, and bookkeeping.
-   **Vector Generator:** The IRQ monitors the `VG DONE` bit from `IN1`. When the VG signals it has finished rendering, the IRQ handler resets it (`VGSTOP`, `VGSTART`), kicking off the rendering process for the *next* frame's display list. This creates a pipelined graphics architecture. 