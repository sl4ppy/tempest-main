# Tempest: ALVGUT.MAC - Vector Graphics Utilities

**File:** `ALVGUT.MAC`
**Purpose:** This file is a library of low-level utility subroutines for the Vector Generator (VG). These routines are the fundamental building blocks used by higher-level modules like `ALDIS2.MAC` to dynamically construct the display list in RAM. They provide a clean API for common operations, abstracting away the raw VG opcodes and byte ordering.

**Dependencies:** All routines in this file rely on a set of Zero Page RAM locations for their state:
-   **`VGLIST`**: A 16-bit pointer to the current address in the display list buffer where the next command should be written.
-   **`VGBRIT`**: An 8-bit value holding the current drawing brightness.
-   **`XCOMP`**: A 4-byte buffer for vector calculations.

---

## 1. Low-Level VG Command Subroutines

These subroutines are thin wrappers that add a single, specific VG instruction to the display list buffer pointed to by `VGLIST`.

| Subroutine | VG Instruction Added | Description |
|---|---|---|
| **`VGCNTR`** | `CNTR` (`$8040`) | **Center Beam:** Adds the command to move the VG beam to the center of the screen. |
| **`VGRTSL`** | `RTSL` (`$C000`) | **Return from Subroutine:** Adds the command to return from a VG subroutine. |
| **`VGHALT`** | `HALT` (`$2000`) | **Halt VG:** Adds the command to stop the VG processor. |

## 2. Parameterized VG Command Subroutines

These routines take parameters in CPU registers and use them to build and add a more complex VG instruction to the display list.

### `VGJSRL`
**Function:** Adds a `JSRL` (Jump to Subroutine, Long) instruction to the display list. This is the core routine for drawing any pre-defined shape from the VG ROM.
-   **Inputs:**
    -   Accumulator: High byte of the target address in VG ROM.
    -   X-Register: Low byte of the target address in VG ROM.
-   **Action:** Assembles the address into the proper `JSRL` opcode format and writes the 16-bit instruction to the `VGLIST` buffer.

### `VGSTAT` / `VGSTA1`
**Function:** Adds a `STAT` instruction to the display list to set the drawing brightness.
-   **Inputs:**
    -   Accumulator: Flag bits.
    -   Y-Register (or `VGBRIT`): The desired brightness value (0-7).
-   **Action:** Assembles the bits into a `STAT` opcode and writes it to the buffer.

### `VGSCA1` / `VGSCAL`
**Function:** Adds a `SCAL` instruction to the display list to set the drawing scale.
-   **Input:** The desired binary and linear scale factors.
-   **Action:** Assembles the values into a `SCAL` opcode and writes it to the buffer.

### `VGVCTR`
**Function:** Adds a `VCTR` (Vector) instruction to the display list. This is the core routine for drawing a line.
-   **Input:** The `XCOMP` RAM variable holds the 16-bit signed `dx` and `dy` components of the vector.
-   **Action:** Assembles the components into the appropriate 16-bit (short vector) or 32-bit (long vector) `VCTR` instruction format and writes it to the buffer.

---

## 3. Number Drawing Utilities

### `VGHEX` and `VGHEXZ`
These routines draw a single hexadecimal digit (0-F) on the screen. They are used by all the score and credit-drawing routines.
-   **Input:** The accumulator holds the 4-bit value of the digit to draw.
-   **Action:** It looks up the address of the corresponding character shape (from `ANVGAN.MAC`) and calls `VGJSRL` to add the command to draw it.
-   **`VGHEXZ`** includes logic for **Zero-Suppression**: if the carry flag is set and the digit is 0, it draws nothing. This is used to prevent scores like `001500` from being displayed.

---

This concludes the documentation for `ALVGUT.MAC`. 