# Tempest: STATE2.MAC - Vector Generator State Machine Microcode

**File:** `STATE2.MAC`
**Purpose:** This file is not 6502 assembly code. It is a data table used to program a Programmable Logic Array (PLA) chip that acts as the core state machine of the Vector Generator (VG) hardware. This file is effectively the "microcode" that defines the VG's behavior, dictating how it fetches and executes the vector drawing commands from the VG ROM.

---

## 1. State Machine Architecture

The comments in the file describe the inputs to the state machine chip. An 8-bit address is formed from these inputs, and the 4-bit value at that address in this table determines the machine's *next* state.

**Hardware Inputs (Address bits `A7-A0`):**
-   `A7`: **Run Flag.** A master control bit (`NOT HALT & NOT GO`).
-   `A6-A4`: **Opcode (`OP2-OP0`).** The top 3 bits of the VG instruction being executed (e.g., from a `VCTR` or `JSRL` command).
-   `A3-A0`: **Current State (`STATE3-STATE0`).** The 4-bit current state of the machine, which is fed back from the output.

The file is a large lookup table. Given a current state and a VG instruction opcode, this table provides the next state, thus driving the hardware through the multi-step process required to execute a single drawing command.

## 2. State Transition Table

The core of the file is a table of state transitions for each of the 8 possible 3-bit opcodes. Each line in the table defines the 16 possible "next states" for a given opcode, depending on the current state.

**Example: `VCTR` Instruction (Opcode `000`)**

The line for the `VCTR` (long vector draw) instruction is:
`.BYTE 9,X,X,X,X,X,X,X,B,8,C,A,D,F,X,0`

This defines the following sequence of micro-steps the hardware takes:
-   If the current state is `0` (idle), the next state is `9`.
-   If the current state is `9`, the next state is `B`.
-   If the current state is `B`, the next state is `8`.
-   ...and so on.

This sequence (`0 -> 9 -> B -> 8 -> C -> A -> D -> F -> 0`) corresponds to the physical operations required to draw a long vector, such as:
1.  Fetch the first word of the instruction (`dy`).
2.  Fetch the second word (`dx`, `intensity`).
3.  Load these values into the hardware's math registers.
4.  Trigger the vector draw operation.
5.  Wait for the draw to complete.
6.  Return to an idle state (`0`).

### Opcode Mapping

By cross-referencing with `VGMC.MAC`, we can map the 3-bit hardware opcodes to the instructions:

| Opcode | Instruction | Description |
|---|---|---|
| `000` | `VCTR` | Long Vector Draw |
| `001` | `HALT` | Halt the VG Processor |
| `010` | `SVEC` | Short Vector Draw |
| `011` | `STAT` | Set Status (Intensity/Clipping) |
| `100` | `WAIT`| (Likely an internal wait state) |
| `101` | `JSRL` | Jump to Subroutine |
| `110` | `RTSL` | Return from Subroutine |
| `111` | `JMPL` | Jump |

---

This concludes the documentation for `STATE2.MAC`. 