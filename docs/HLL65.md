# Tempest: HLL65.MAC - High-Level Language Macros

**File:** `HLL65.MAC`
**Purpose:** This file is a macro library for the `MAC65` assembler. It provides a set of macros that emulate high-level structured programming constructs, most notably `IF/THEN/ELSE/ENDIF` blocks and `BEGIN...END` loops. This allows for more readable and maintainable code than would be possible with traditional `JMP` and `Bxx` branch instructions alone.

---

## 1. System Overview

The HLL65 system is built upon a set of internal macros (`DEFIF`, `DEFEND`, `LOC`, `FND`) that dynamically create the user-facing macros (e.g., `IFEQ`, `NEEND`). It uses an assembler variable (`...X`) as a stack pointer to keep track of jump locations, which allows the control structures to be nested.

When an `IFxx` or `BEGIN` macro is used, it pushes the current location onto this internal stack. The corresponding `ENDIF` or `xxEND` macro then uses this stored location to resolve the jump, patching in the correct relative offset for the branch instruction.

## 2. Conditional (`IF...`) Macros

These macros create conditional blocks of code that are executed only if the specified CPU status flag is set or clear. They are direct analogs to `if` statements in high-level languages.

**Usage:**
```assembly
    CMP #$0A      ; Compare accumulator with 10
    IFEQ          ; If equal (Z flag is set)...
        ;... code to run if equal
    ELSE
        ;... code to run if not equal
    ENDIF
```

| Macro | Condition for Execution | Underlying 6502 Branch |
|---|---|---|
| `IFCC`| **If Carry Clear** (C=0) | `BCS` (Branches if C=1) |
| `IFCS`| **If Carry Set** (C=1) | `BCC` (Branches if C=0) |
| `IFEQ`| **If Equal** (Z=1) | `BNE` (Branches if Z=0) |
| `IFNE`| **If Not Equal** (Z=0) | `BEQ` (Branches if Z=1) |
| `IFMI`| **If Minus** (N=1) | `BPL` (Branches if N=0) |
| `IFPL`| **If Plus** (N=0) | `BMI` (Branches if N=1) |
| `IFVC`| **If Overflow Clear** (V=0)| `BVS` (Branches if V=1) |
| `IFVS`| **If Overflow Set** (V=1) | `BVC` (Branches if V=0) |
| `THEN`| Marks the end of the conditional branch instruction. Required after the `IFxx` block. | - |
| `ELSE`| Provides an alternative block of code to be executed if the `IFxx` condition is false. This is optional. | - |
| `ENDIF`| Marks the end of the entire `IF...ELSE` block and resolves the jump logic. This is required. | - |

*Note: The underlying branch logic seems inverted (e.g., `IFCC` uses `BCS`) because the macro makes the assembler branch **over** the `THEN` block if the condition is false.*

## 3. Loop (`BEGIN...END`) Macros

These macros create loops that repeat until a certain condition is met. They are analogous to `do...while` or `repeat...until` loops.

**Usage:**
```assembly
    LDX #10
    BEGIN         ; Marks the start of the loop
        DEX
        CPX #0
    NEEND         ; Loop back to BEGIN if Not Equal (Z=0)
```

| Macro | Condition to Loop | Underlying 6502 Branches |
|---|---|---|
| `BEGIN` | Marks the top of the loop. | - |
| `CCEND` | **Loop if Carry Clear** (C=0) | `BCS` (long jmp), `BCC` (short jmp) |
| `CSEND` | **Loop if Carry Set** (C=1) | `BCC` (long jmp), `BCS` (short jmp) |
| `EQEND` | **Loop if Equal** (Z=1) | `BNE` (long jmp), `BEQ` (short jmp) |
| `NEEND` | **Loop if Not Equal** (Z=0) | `BEQ` (long jmp), `BNE` (short jmp) |
| `MIEND` | **Loop if Minus** (N=1) | `BPL` (long jmp), `BMI` (short jmp) |
| `PLEND` | **Loop if Plus** (N=0) | `BMI` (long jmp), `BPL` (short jmp) |
| `VCEND` | **Loop if Overflow Clear** (V=0)| `BVS` (long jmp), `BVC` (short jmp) |
| `VSEND` | **Loop if Overflow Set** (V=1) | `BVC` (long jmp), `BVS` (short jmp) |

*Note: The macros are smart enough to use a short relative branch if the loop is small, or a longer `JMP` if it's out of range of a standard branch.*

## 4. Custom Data Loading Macros

These macros appear to be custom, non-standard ways of loading data. Their usage is likely a specific trick for this assembler.

| Macro | Description |
|---|---|
| `LDAL ...1` | Emits an `LDA #imm` opcode (`$A9`) followed by the full 16-bit word provided in parameter `...1`. It then moves the assembler's location counter back by one byte. This is a highly unusual construction. The 6502 would execute `LDA` with the low byte of `...1`, and the program counter would then point to the high byte, which would be interpreted as the next instruction's opcode. This is a form of interleaved code and data. |
| `LDAH ...1` | Similar to `LDAL`, but it wraps the 16-bit word in `.ENABL M68` / `.DSABL M68` directives, suggesting it may be related to interfacing with the auxiliary Mathbox board or is intended to be interpreted as 6800-family data. | 