# Tempest: Core Systems Documentation

## 5. Vector Graphics Engine

This document details the architecture and function of the Atari Vector Generator (VG) hardware and the software systems that drive it. The VG is a specialized co-processor that operates independently of the main CPU. Its sole purpose is to read a "display list" program from RAM and draw the corresponding vectors on the screen.

---

### 5.1. Purpose and Overview

The Vector Graphics Engine is the final stage in the rendering pipeline. After the Mathbox has transformed all 3D world coordinates into 2D screen coordinates, the 6502 assembles a program for the VG to execute. This program, called a display list, contains simple instructions like "move the beam here," "draw a line there," and "jump to a subroutine." The VG reads this program and translates it into the analog signals needed to guide the CRT beam and draw the game's graphics.

-   **Context of Use:** The entire `ALDISP.MAC` module is dedicated to generating the display list. The main `DISPLAY` routine is called once per frame. It fills one of two RAM buffers with VG instructions. The IRQ handler (`ALHARD.MAC`) then "starts" the VG, pointing it to the newly created display list.
-   **Architecture:** The system is a classic pipelined display architecture. While the VG is busy drawing the contents of Buffer A for the current frame, the 6502 CPU and Mathbox are busy creating the display list for the *next* frame in Buffer B. This prevents tearing and ensures a smooth, consistent frame rate.

---

### 5.2. The Graphics Pipeline

1.  **Game Logic (`ALWELG.MAC`):** The state of all game objects (player, enemies, bullets) is updated, determining their 3D positions in the world.
2.  **Mathbox Transformation (`WORSCR`):** For each vertex of each object, the 3D world coordinates are sent to the Mathbox, which applies rotation, translation, and perspective projection, returning 2D screen coordinates.
3.  **Display List Generation (`ALDISP.MAC`):** The main `DISPLAY` routine iterates through all visible objects. It takes the 2D screen coordinates from the Mathbox and uses the macros in `VGMC.MAC` to write a sequence of binary instructions into a block of RAM (`VECRAM`). This block of instructions is the display list.
4.  **Vector Generation (Hardware):** The 6502 commands the VG to start executing the newly created display list. The VG reads the instructions from `VECRAM` and draws the vectors on the monitor, completely independent of the CPU. The CPU is now free to begin calculating the next frame.

---

### 5.3. The Vector Generator Instruction Set

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

### 5.4. Hardware Interaction and Double Buffering

The 6502 controls the VG via a few key hardware registers and a double-buffering scheme to ensure flicker-free animation.

-   **`VGSTART` (`$D501`):** A write-only register. Writing any value to this address commands the VG to begin executing the display list currently pointed to by the internal vector program counter.
-   **`VGSTOP` (`$D500`):** A write-only register. Writing any value here immediately halts the VG.
-   **`MHALT` flag (in `IN1` register):** A read-only flag that the VG sets high when it executes a `HALT` instruction. The IRQ handler polls this bit to know when the VG is idle and ready for the next frame's display list.

-   **Double Buffering:**
    1.  The game allocates two buffers in RAM for display lists, `VECRAM` Buffer A and Buffer B.
    2.  In Frame 1, the `DISPLAY` routine builds the display list in Buffer A.
    3.  At the end of Frame 1, the IRQ points the VG hardware at Buffer A and issues a `VGSTART` command.
    4.  In Frame 2, while the VG is busy reading from Buffer A, the `DISPLAY` routine builds the next display list in Buffer B.
    5.  At the end of Frame 2, the IRQ sees that the VG has halted, points it at Buffer B, issues a `VGSTART`, and the cycle repeats.
    6.  The `SWAPVG` routine is responsible for swapping the buffer pointers.

---

### 5.5. Data Structures

-   **Shape Tables (`ALVROM.MAC`):** The vector data for all game objects (the player's ship, enemies, text characters) is stored as subroutines in the ROM. A subroutine is simply a sequence of `VCTR` commands followed by an `RTSL`.
    -   Example: The player's ship shape is at the label `PTCURS`. To draw the ship, the `DISPLAY` routine just needs to add a single `JSRL PTCURS` instruction to the display list.

-   **Display List Buffers (`VECRAM`):** These are the blocks of RAM where the display list is assembled each frame before being handed off to the hardware. The list is a mix of `JSRL` instructions pointing to the shape tables in ROM and dynamically generated `VCTR` instructions for things like laser fire and explosions.

---

### 5.6. The `DISPLAY` Routine (`ALDISP.MAC`)

The `DISPLAY` routine is the master "compiler" for the vector engine. Its job is to generate a complete, valid display list program every frame.

**Execution Flow:**
1.  **Initialize:** It calls `INIMAT` to set up the Mathbox with the current camera view.
2.  **Select Buffer:** It selects the available (not currently being drawn) `VECRAM` buffer to write to.
3.  **Draw Objects:** It proceeds to call a series of subroutines, one for each type of object in the game:
    -   `DSPCUR`: Displays the player's cursor ("Blaster").
    -   `DSPCHG`: Displays active player shots.
    -   `DSPINV`: Displays the enemies ("Invaders").
    -   `DSPEXP`: Displays explosions.
    -   `DSPNYM`: Displays the Fuseballs and Tankers ("Nymphs").
    -   `INFO`: Displays scores, credits, and messages.
    -   `DSPWEL`: Displays the well itself.
4.  **How Objects are Drawn:** Each subroutine (`DSPCUR`, etc.) iterates through its list of active objects. For each object, it:
    a.  Gets the object's 3D world coordinates.
    b.  Calls `WORSCR` to have the Mathbox transform the coordinates to 2D screen space.
    c.  Uses the `VGYABS` or `VGVTR1` macros to generate instructions to move the beam to the object's starting position.
    d.  Uses the `JSRL` macro to generate an instruction to call the object's shape subroutine from ROM.
5.  **Finalize:** After all objects are processed, it writes a final `HALT` instruction to the end of the display list. The list is now ready for the hardware. 