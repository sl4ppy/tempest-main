# Tempest: VGMC.MAC - Vector Generator Macros

**File:** `VGMC.MAC`
**Purpose:** This file is the "Instruction Set Architecture" (ISA) manual for Atari's custom Vector Generator (VG) hardware. It contains a set of low-level macros that assemble directly into the 16-bit opcodes that the VG processor executes. This is the lowest level of the graphics system.

---

## 1. Vector Generator Instruction Set

Each macro in this file corresponds to a single instruction that the VG hardware can execute. All instructions are 16-bit words, where the most significant bits define the operation and the lower bits define the parameters.

### `VCTR dx, dy, intensity`
**Opcode:** `4` (Short Vector) or `0` (Long Vector)

This is the most fundamental instruction. It draws a line (a vector) from the current beam position to a new position.

-   **Short Vector (16-bit):** If the `dx` and `dy` values are small enough to fit within 5 bits, the macro generates a highly efficient 16-bit instruction. The opcode, deltas, and intensity are all packed into a single word.
-   **Long Vector (32-bit):** If the deltas are too large, the macro generates a 32-bit, two-word instruction. This allows for drawing lines of any length across the screen.
    -   Word 1: Contains the `dy` component.
    -   Word 2: Contains the opcode, `dx` component, and the `intensity` (0=off, 7=brightest).

### `JSRL label`
**Opcode:** `A`

**Jump to Subroutine, Long.** This is the cornerstone of creating complex graphics. It functions like a subroutine call in a CPU.
- The hardware pushes its current program counter onto its internal stack.
- It then jumps to the 12-bit word `label` address within the 8KB Vector ROM space.
- The VG begins executing the instructions at the new address until it encounters an `RTSL` instruction.

### `JMPL label`
**Opcode:** `E`

**Jump, Long.** This is an unconditional jump. The VG immediately begins executing instructions at the specified `label`. It does not push the current address and does not return.

### `RTSL`
**Opcode:** `C`

**Return from Subroutine, Long.** This instruction pops an address from the VG's internal stack and returns execution to the instruction following the last `JSRL` call. The VG has a hardware stack that can handle five levels of subroutine nesting.

### `SCAL binary_scale, linear_scale`
**Opcode:** `7`

**Set Scale.** This instruction sets the drawing scale for all subsequent `VCTR` commands. It has two components:
-   **`binary_scale`**: A 3-bit value (0-7) that scales vectors by powers of two (e.g., 0=full size, 1=1/2 size, 2=1/4 size).
-   **`linear_scale`**: An 8-bit value for fine-grained scaling.

### `CNTR`
**Opcode:** `8`

**Center.** This instruction immediately moves the drawing beam to the center of the screen (0, 0) without drawing a line.

### `HALT`
**Opcode:** `2`

**Halt.** This instruction stops the VG processor. The main 6502 CPU can then load a new display list pointer into the hardware and restart the VG.

### `STAT z, hi, in`
**Opcode:** `6`

**Set Status.** This is an older instruction used in previous Atari vector games to set intensity and a clipping window. In Tempest, it appears to be superseded by the dedicated color hardware and is not generally used.

---

## 2. Helper Macros

### `ALPHA string`
This is a utility macro that simplifies drawing text. It takes a string as input and expands it into a series of `JSRL` calls, one for each character. For example, `ALPHA CAT` becomes:
```assembly
JSRL CHAR.C
JSRL CHAR.A
JSRL CHAR.T
```

---

This concludes the documentation for `VGMC.MAC`. 