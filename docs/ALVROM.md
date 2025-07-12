# Tempest: ALVROM.MAC - Vector ROM Data

**File:** `ALVROM.MAC`
**Purpose:** This file is not 6502 assembly code. It is a data file that contains the raw graphical information for every object in the game. This data is assembled into a binary ROM which is read directly by the dedicated **Vector Generator (VG)** hardware.

---

## 1. Vector Data Format

The file is a sequence of commands and coordinates that the VG hardware interprets to draw lines on the screen. The data stream is generated using a set of low-level macros defined in `VGMC.MAC`. The primary commands are:

| Command | Parameters | Description |
|---|---|---|
| **`VCTR`** | `dx, dy, intensity` | **Draw Vector.** This is the most fundamental command. It instructs the VG to draw a line from the current beam position to a new position, offset by `dx` and `dy`. The `intensity` parameter controls the brightness of the line. An intensity of `0` means the beam moves without drawing, which is used for positioning. |
| **`JSRL`** | `label` | **Jump to Subroutine, Long.** This powerful command tells the VG to jump to another location (`label`) in the ROM, draw the shape defined there, and then return. This allows for complex objects to be constructed from smaller, reusable graphical components (e.g., drawing a letter of the alphabet). |
| **`RTSL`** | - | **Return from Subroutine, Long.** This command marks the end of a graphical subroutine, telling the VG to return to the location from which the `JSRL` was called. |
| **`SCAL`** | `scale_factor` | **Set Scale.** Sets a global scale multiplier for all subsequent `VCTR` commands. This allows the same shape data to be drawn at different sizes (e.g., for enemies that are far away versus close up). |
| **`CNTR`** | - | **Center Beam.** Immediately moves the drawing beam to the center of the screen (0, 0). |
| **`CSTAT`** | `color_code` | **Set Color.** This is a custom macro in this file that generates the command to change the drawing color for all subsequent vectors. |

## 2. High-Level Drawing Macros

To simplify the process of creating complex geometric shapes, the file defines its own set of higher-level macros that expand into a series of `VCTR` commands.

-   **`CVEC newx, newy, brit`**: A "coordinate-based" vector command. Instead of relative deltas, it takes absolute coordinates. The macro calculates the required `dx` and `dy` relative to the last position.
-   **`CIRC16`**: A macro that generates all the `VCTR` commands needed to draw a 16-sided approximation of a circle.
-   **`SPOK16`**: A macro that generates the commands to draw a 16-spoke wheel shape.

## 3. Shape Data Inventory (Partial)

This file contains the labeled data blocks for every graphical element in the game.

### 3.1. Alphanumerics and Symbols

The file begins with definitions for non-standard text characters.
-   **`DASH`**: The shape for a hyphen or dash.
-   **`COPYR`**: The shape for the © copyright symbol.
-   **`HALF`**: The shape for the "1/2" graphic used in the credit display.
-   The file also includes `ANVGAN.MAC`, which contains the vector data for the standard A-Z alphabet and 0-9 numerals.

### 3.2. Game Objects

-   **`LIFE1`**: The shape for the player's "life" icon, which is displayed at the top of the screen. The master pointer `LSYMBL` points to this routine.
-   **`LIFE0`**: A special "null" shape. When a life needs to be hidden, the game calls this routine, which simply moves the beam but doesn't draw anything.

### 3.3. Test Patterns

-   **`INTEST`**: A complex picture used in the self-test mode to draw grids of different colors and intensities, allowing an arcade operator to verify the monitor's function.

---

## 4. Shape Data Inventory (Complete)

This section provides an index of the labels for the most important graphical objects defined in this file.

### 4.1. Player and Shots
-   **`PTCURS`**: The player's iconic U-shaped ship, the "Blaster".
-   **`PTSPLA`**: The player's shot, a simple vertical line.
-   **`DIARA2`**: The "Superzapper" charge graphic that appears around the player's ship.

### 4.2. Enemies
-   **`PTFLIP`**: The "Flipper" enemy, which walks along the rim of the well.
-   **`PTTANK`**: The "Tanker" enemy, which splits into two Flippers when shot.
-   **`PTSPIK`**: The "Spiker", a long, thin enemy that lays "spikes" down the well. The file contains separate data for the Spiker's body and its growing tail.
-   **`PTFUSE`**: The "Fuseball" enemy.
-   **`PTPULS`**: The "Pulsar" enemy.
-   **`PTESHO`**: The enemy shot graphic.

### 4.3. Explosions
The game uses a sequence of shapes to create explosion animations.
-   **`EXPL1`, `EXPL2`, `EXPL3`, `EXPL4`**: A sequence of expanding starburst shapes used for generic enemy explosions.
-   **`SPLAT1` through `SPLAT6`**: The more complex, multi-frame animation used for the player's death explosion.

### 4.4. Starfield
The background starfield is created from several static patterns of dots. The game creates the illusion of movement by drawing these patterns and moving their origin point on each frame.
-   **`MSTAR1`, `MSTAR2`, `MSTAR3`, `MSTAR4`**: Macros that define four different, dense patterns of dots used for the starfield.

### 4.5. Alphanumerics (A-Z, 0-9)
Starting at the label `CHARA`, the file contains the vector definitions for every letter of the alphabet (`CHARA`, `CHARB`, etc.) and every digit (`CHAR.0`, `CHAR.1`, etc.). These are used by the message display engine to render all text.

---

This concludes the documentation for `ALVROM.MAC`. 