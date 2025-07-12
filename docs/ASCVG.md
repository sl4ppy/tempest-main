# Tempest: ASCVG.MAC - ASCII to Vector Graphics Macro

**File:** `ASCVG.MAC`
**Purpose:** This file provides a single, powerful macro named `.ASCVG`. This macro does not contain vector data itself; instead, it acts as a "string compiler." It takes a human-readable ASCII string and converts it into the compressed, byte-array data format that the game's message display engine (`MSGS` in `ALSCO2.MAC`) uses to render text.

---

## 1. The `.ASCVG` Macro

This macro is a piece of developer tooling used within the assembly source code to define all the text messages in the game.

-   **Input:** A standard string of ASCII characters. For example: `.ASCVG "HIGH SCORES"`.
-   **Output:** A sequence of bytes (`.BYTE` directives) representing the compiled string.

### 1.1. Compilation Process

The macro iterates through the input string character by character and performs the following conversion:

1.  **Character to Index:** It converts the ASCII value of the character into a numeric index.
    -   For letters (A-Z), the formula is `(ASCII - $36)`.
    -   For numbers (0-9), the formula is `(ASCII - $2F)`.
    -   A space character becomes `0`.
2.  **Index to Offset:** It multiplies the resulting index by 2. This final value is an offset into the `VGMSGA` table, which is the master list of `JSRL` commands for drawing each character shape.
3.  **End-of-String Marker:** For the very last character in the string, it sets the most significant bit of the final byte (i.e., it performs a bitwise OR with `$80`). This marker tells the `MSGS` rendering engine when to stop drawing the string.

### 1.2. Example

Consider the macro call `.ASCVG "HI"`.

1.  **'H'**:
    -   ASCII value is `$48`.
    -   Index = `$48 - $36 = $12` (18 decimal).
    -   Offset = `18 * 2 = 36`.
    -   The macro generates: `.BYTE 36`

2.  **'I'**:
    -   ASCII value is `$49`.
    -   Index = `$49 - $36 = $13` (19 decimal).
    -   Offset = `19 * 2 = 38`.
    -   This is the last character, so the end-of-string marker is added: `38 | $80 = $A6` (166 decimal).
    -   The macro generates: `.BYTE 166`

The final output in the ROM for the string "HI" would be the byte sequence `36, 166`. This data is then read by the `MSGS` routine at runtime to draw the message. This compilation step makes the source code more readable and saves memory in the final ROM.

---

This concludes the documentation for `ASCVG.MAC`. 