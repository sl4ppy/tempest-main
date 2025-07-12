# Tempest: ANVGAN.MAC - Alphanumeric Vector Data

**File:** `ANVGAN.MAC`
**Purpose:** This file is a data library containing the vector graphics information for the standard alphanumeric character set (A-Z and 0-9). It is included by `ALVROM.MAC` to form part of the complete Vector Generator ROM data.

---

## 1. File Structure and Purpose

This file contains no executable 6502 code. It consists entirely of a series of graphical subroutines, one for each letter and number. Each subroutine is a sequence of `VCTR` (Vector) and `RTSL` (Return from Subroutine) commands, as defined in `VGMC.MAC`.

When the game needs to display text, the `MSGS` routine in `ALSCO2.MAC` generates a series of `JSRL` (Jump to Subroutine, Long) instructions. These `JSRL`s point to the labels within this file (`CHAR.A`, `CHAR.B`, etc.) to draw the appropriate characters on the screen.

## 2. Example: The Letter 'A'

The format for each character is a simple sequence of relative vector draws. For example, the entry for the letter 'A' is:

```assembly
CHAR.A:	VCTR 0,16,.BRITE	; Draw line up
	VCTR 8,8,.BRITE		; Draw line up and right (top point)
	VCTR 8,-8,.BRITE	; Draw line down and right
	VCTR 0,-16,.BRITE	; Draw line down
	VCTR -16,8,0		; Move beam left and up without drawing
	VCTR 16,0,.BRITE	; Draw the horizontal crossbar
	VCTR 8,-8,0		; Move beam away from the character
	RTSL			; Return from subroutine
```
This sequence of 8 commands, when executed by the Vector Generator, draws the shape of the letter 'A'. Every other character in the file is defined in a similar manner.

---

This concludes the documentation for `ANVGAN.MAC`. 