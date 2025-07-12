# Tempest: MBUCOD.V05 - Mathbox Microcode

**File:** `MBUCOD.V05`
**Purpose:** This file contains the microcode (firmware) for the "Mathbox," a custom auxiliary processor built from AMD 2901 bit-slice components. This is not 6502 assembly code, but the native instruction set for this custom CPU. The Mathbox acts as a hardware accelerator, performing the complex 16-bit multiplication and division operations required for the game's 3D graphics, offloading the work from the main 6502 processor.

---

## 1. Mathbox Architecture

The Mathbox is a 16-bit processor with its own set of internal registers (A, B, Q), a dedicated ALU (Arithmetic Logic Unit), and its own microcode program ROM. The main 6502 CPU communicates with it through a set of memory-mapped registers (defined in `ALCOMN.MAC`) to pass operands and read back results.

The microcode itself consists of very wide instructions (24 bits), where different groups of bits control the ALU function, the source of operands, the destination of the result, and program flow (jumps and branches). The source code uses a series of complex macros to make programming this wide instruction word manageable.

## 2. Instruction Set (ALU and Source)

The instruction set is defined by a series of macros that set specific bits in the 24-bit micro-instruction word.

### 2.1. ALU Functions

These macros define the core arithmetic or logical operation to be performed in a given instruction cycle.

| Macro | Function |
|---|---|
| `ADD` | Adds two source operands. |
| `SUBR`| **Subtract Reverse:** (S - R). Subtracts the second operand from the first. |
| `SUBS`| **Subtract Standard:** (R - S). Subtracts the first operand from the second. |
| `OR`  | Performs a bitwise OR operation. |
| `AND` | Performs a bitwise AND operation. |
| `BIC` | **Bit Clear (AND NOT):** Performs a bitwise `(NOT R) AND S`. |
| `EXOR`| Performs a bitwise Exclusive OR operation. |
| `DEC` | **Decrement:** A specialized version of `SUBR` used to decrement a value by one. |

### 2.2. Source Operand Modes

These macros, primarily `SOC` and `EVSRC`, determine which registers or data paths provide the two inputs (the "R" and "S" operands) for the ALU function.

-   **Internal Registers:** The source can be one of the internal registers (A or B).
-   **Q Register:** A special register used for multiplication and division.
-   **External Data Bus (`D`):** The Mathbox can take an operand directly from the data bus, which is how the 6502 provides it with numbers to work on.
-   **Zero (`Z`):** A constant value of zero can be used as a source operand.

These are combined into modes like `ZA` (sources are Zero and Register A), `DA` (sources are the Data Bus and Register A), or `AB` (sources are Register A and Register B).

### 2.3. Destination Control

These macros control where the result of the ALU operation is stored.

| Macro | Destination / Action |
|---|---|
| `QREG` | Store the result in the internal **Q Register**. |
| `RAMF` | Store the result in one of the 16 internal RAM registers. |
| `NOP` | **No Operation:** The result is discarded. |
| `RAMQU`/`RAMU` | **RAM Shift Up:** Store the result in a RAM register, and also shift the Q register **up** (left) by one bit. |
| `RAMQD`/`RAMD` | **RAM Shift Down:** Store the result in a RAM register, and also shift the Q register **down** (right) by one bit. |

The shifting operations are the key to implementing multiplication and division efficiently in hardware.

### 2.4. Program Flow

These macros control the flow of the microcode program itself.
-   **`JMP`**: An unconditional jump to a new address in the microcode ROM.
-   **`JPL`**: A conditional jump. It "Jumps on PLus" or "Jumps on PLeasing," but its technical function is to jump if the ALU's "Sign" output is set (i.e., the result of the last operation was negative). This is the primary mechanism for conditional logic.

---

## 3. Core Algorithms

The Mathbox firmware is a simple loop that waits for a command from the 6502, performs one of several hard-coded mathematical operations, and then waits for the next command.

### 3.1. Multiplication (`MULXP`, `MULYP`)

The multiplication routine implements a standard **16x16 bit shift-and-add algorithm**.
1.  It loads the two 16-bit operands from the main data bus into its internal registers.
2.  It enters a loop that runs 16 times (once for each bit of the multiplier).
3.  Inside the loop, it performs a conditional add (`CADD`) and a right-shift (`RAMQD`).
4.  After 16 iterations, the final 32-bit result is stored across two internal registers, ready for the 6502 to read back.

### 3.2. Division (`DIVID`)

The division routine implements a **32/16 bit non-restoring shift-and-subtract algorithm**.
1.  It loads the 32-bit dividend and 16-bit divisor.
2.  It enters a 16-iteration loop.
3.  Inside the loop, it performs a subtraction (`SUBR`) and a left-shift (`RAMQU`).
4.  After 16 iterations, the 16-bit quotient and 16-bit remainder are stored in internal registers.

### 3.3. Diagnostics (`DIAG`)

The firmware begins with a power-on self-test routine. It performs a series of operations on its own internal registers and ALU. If any test fails, the processor enters an infinite `STALL` loop, effectively halting. This prevents a faulty Mathbox from corrupting the main game.

---

This concludes the documentation for `MBUCOD.V05`. 