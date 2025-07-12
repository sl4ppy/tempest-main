# Tempest: Core Systems Documentation

## 4. Mathbox Co-Processor

This document provides a complete technical specification for the "Mathbox," a custom, high-speed, microcode-driven arithmetic co-processor designed by Atari to perform the complex 16-bit calculations required for *Tempest*'s 3D vector graphics.

---

### 4.1. Purpose and Overview

The 6502 main processor is an 8-bit CPU and is not powerful enough to perform the hundreds of 16-bit multiplications, divisions, and translations required to render a 3D vector scene in real-time. The purpose of the Mathbox is to offload these specific, repetitive calculations from the 6502.

The Mathbox is not a general-purpose CPU. It is a specialized piece of hardware with a small, fixed set of programs (microcode) designed to do one thing: transform 3D "world" coordinates into 2D "screen" coordinates, applying rotation, translation, and perspective projection.

-   **Context of Use:** The Mathbox is used exclusively by the display list generation logic in `ALDISP.MAC`. At the start of each frame, the `INIMAT` subroutine loads the current camera view matrix into the Mathbox. Then, for every vertex of every object to be drawn, the `WORSCR` subroutine sends the vertex's 3D coordinates to the Mathbox and waits for the 2D result.
-   **Interfaces:** The 6502 interacts with the Mathbox via a 32-byte block of memory-mapped I/O registers. Writing to specific addresses in this block sends data and simultaneously triggers pre-programmed calculation sequences.

---

### 4.2. Architecture and Structure

-   **Core:** The Mathbox is built using four AMD 2901 4-bit "bit-slice" processors, chained together to create a 16-bit ALU (Arithmetic Logic Unit).
-   **Microcode ROM:** It has its own internal 256-word x 24-bit program memory (uCODE ROM). This ROM contains the small, highly optimized programs that execute the mathematical formulas. The binary for this ROM is `MBUCOD.COM`, which is assembled from the source `MBUCOD.V05`.
-   **Host Communication:** The 6502 communicates with the Mathbox as a peripheral device. The Mathbox is idle until the 6502 writes to one of its command registers. It then executes its microcode program, and upon completion, sets a "done" flag that the 6502 can poll.

---

### 4.3. 6502 Host Interface (Memory-Mapped Registers)

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

### 4.4. Core Mathematical Operations

The Mathbox's microcode is pre-programmed to solve the core equations of 3D-to-2D projection.

#### 4.4.1. 2D Rotation and Translation

The primary transformation combines rotation and translation (moving the object relative to the camera). The resulting 2D screen coordinates `(x', y')` are calculated from the 3D world coordinates `(x, y, z)` and the camera's position `(e, f)` and rotation angle (`a=cos(angle)`, `b=sin(angle)`).

-   `x' = (x-e)*a - (y-f)*b`
-   `y' = (x-e)*b + (y-f)*a`

This calculation is triggered by the `SYM` or `YHSM` commands.

#### 4.4.2. Perspective Projection

To create the illusion of depth, the 2D coordinates are scaled based on their distance from the viewer (their Z-coordinate). This is a division operation. The final projected screen coordinate `(X`<sub>`p`</sub>`, Y`<sub>`p`</sub>`)` is calculated:

-   `Z`<sub>`eff`</sub> ` = (z - z_cam) * scale_factor` (Effective Z-depth)
-   `X`<sub>`p`</sub> ` = x' / Z`<sub>`eff`</sub>
-   `Y`<sub>`p`</sub> ` = y' / Z`<sub>`eff`</sub>

The `YHSP` command (`Write Y High and Start Perspective`) executes this entire sequence: it takes `(x,y,z)` as input, performs the rotation/translation to get `(x', y')`, and then performs the perspective divide to get the final on-screen coordinates, which are read back by the 6502.

---

### 4.5. Integration and Programming Model

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
    LDA Y,SINTAB  ; Load sin(angle)
    STA BL        ; Store in Mathbox register B
    LDA Y,COSTAB  ; Load cos(angle)
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
The resulting screen coordinates (`SXL/H`, `SYL/H`) are then used to build the `VCTR` commands for the Vector Generator. 