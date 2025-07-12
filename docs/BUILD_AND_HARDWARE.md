# Tempest: Build, Dependencies, and Hardware API

This document details the necessary tools, build procedures, and hardware interactions required to reconstruct the Tempest arcade game from this source code.

---

## 1. Dependencies and Environment (Stage 4)

This project was developed using a specific toolchain from the late 1970s / early 1980s. Rebuilding the original binaries from source would require sourcing or recreating these original tools.

-   **Assembler:** `MAC65`
    -   This is a macro assembler for the 6502 family of microprocessors.
    -   No specific version number is documented.
-   **Linker:** `LINKM`
    -   This tool links the object files produced by the assembler into a single, final executable binary.
    -   No specific version number is documented.
-   **ROM Verification Data:**
    -   The `.DAT` files (`002X1.DAT`, `002X2.DAT`, `MABOX.DAT`) are data files used by a ROM reader/verifier utility to confirm that the data burned onto the physical EPROM/ROM chips is correct.

There are no external software libraries or environment variables in the modern sense. The entire environment is defined by the assembler, the linker, and the target hardware itself.

---

## 2. Build and Setup Instructions (Stage 5)

### 2.1. Build Command

The original `TEMPST.DOC` file specifies the exact linker command used to build the final game binary (`ALEXEC.LDA`). The command is:

`ALEXEC/L,ALEXEC/A=ALWELG,ALSCO2,ALDIS2,ALEXEC,ALSOUN,ALVROM,ALCOIN,ALLANG,ALHAR2,ALTES2,ALEARO,ALVGUT`

**Explanation:**
-   This command instructs the `LINKM` linker to...
-   Create a list file named `ALEXEC.L`...
-   And an absolute binary file named `ALEXEC.A` (which should be renamed to `ALEXEC.LDA`)...
-   By assembling and linking the following source files in order:
    1.  `ALWELG.MAC` (Attract Mode)
    2.  `ALSCO2.MAC` (Scoring & Gameplay)
    3.  `ALDIS2.MAC` (Display Manager)
    4.  `ALEXEC.MAC` (Main Executive)
    5.  `ALSOUN.MAC` (Sound Engine)
    6.  `ALVROM.MAC` (Vector ROM Data)
    7.  `ALCOIN.MAC` (Coin Logic)
    8.  `ALLANG.MAC` (Language Data)
    9.  `ALHAR2.MAC` (IRQ Handler)
    10. `ALTES2.MAC` (Test Mode)
    11. `ALEARO.MAC` (EAROM Driver)
    12. `ALVGUT.MAC` (Vector Graphics Utilities)

### 2.2. Post-Build Process

The output of the linker is a single binary file (e.g., `ALEXEC.LDA`). This file would then be split into multiple smaller chunks, corresponding to the size of the physical ROM chips on the arcade PCB. A ROM programmer would then be used to "burn" each chunk onto a separate chip. The `TEMPST.DOC` file contains a table mapping part numbers to memory addresses for this purpose.

---

## 3. Hardware API and Data Contracts (Stage 6)

The software's "API" is the set of memory-mapped hardware registers it uses to interact with the physical components of the arcade machine. A comprehensive list of these registers is documented in `docs/ALCOMN.md`.

### 3.1. Key Hardware Registers

| Address | Name | R/W | Description |
|---|---|---|---|
| `$0800` | `COLPORT` | W | **Color Port:** Sets the color for the vector generator. |
| `$0C00` | `IN1` | R | **Input Port 1:** Reads the state of the cabinet switches (Coin, Test, Slam). |
| `$0D00` | `INOP0` | R | **DIP Switch Bank 0:** Reads the operator settings for pricing. |
| `$0E00` | `INOP1` | R | **DIP Switch Bank 1:** Reads operator settings for lives, bonus, and language. |
| `$2000` | `VECRAM`| W | **Vector RAM:** The start of the 2KB RAM where the CPU builds the display list. |
| `$4800` | `VGSTART` | W | **Vector Generator Start:** Writing here tells the VG to start drawing. |
| `$5000` | `WTCHDG` | W | **Watchdog Reset:** Must be written to periodically to prevent a hardware reset. |
| `$5800` | `VGSTOP` | W | **Vector Generator Stop:** Halts the VG processor. |
| `$6000` | `HARDWA` | R/W| **Base address for the Auxiliary "Mathbox" board.** All EAROM, POKEY, and Mathbox registers are offsets from this address. |
| `$60C0` | `POKEY` | R/W | Base address for the first POKEY sound chip. |
| `$60D0` | `POKEY2`| R/W | Base address for the second POKEY sound chip (also used for rotary dial input). |

### 3.2. EAROM Data Schema

The EAROM serves as the game's non-volatile "database." It has a simple, fixed schema:
-   **Block 0:** Initials Table (9 bytes)
-   **Block 1:** High Score Table (11 bytes)
-   **Block 2:** Bookkeeping Statistics (size varies)

Each block is protected by a checksum to ensure data integrity. The `ALEARO.MAC` driver manages all access to this data.

---

## 4. Additional Details (Stage 7)

### Key Design Decisions
-   **Hardware Acceleration:** The design offloads critical, performance-intensive tasks to dedicated hardware, a common technique in high-performance arcade games of the era. The **Vector Generator** handles all graphics rendering, and the **Mathbox** handles all 16-bit multiplication for 3D math. This frees up the main 6502 CPU to focus entirely on game logic.
-   **Data-Driven Design:** Many of the subsystems (sound, text, vector shapes) are designed to be "data-driven." The logic is generic, and the specific outcomes are controlled by structured tables of data. This is a sophisticated design that makes the code more flexible and easier to modify.
-   **Robustness:** The use of a hardware watchdog timer, a software stack check in the IRQ, and checksums on the EAROM data demonstrates a focus on creating a highly robust and reliable system that can run continuously in an arcade environment.

---

This concludes the documentation of the build process and hardware interface. 