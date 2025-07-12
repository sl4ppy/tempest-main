# Tempest: ALDIS2.MAC - Display List Manager

**File:** `ALDIS2.MAC`
**Purpose:** This is the master display and rendering module for Tempest. It is responsible for creating the entire vector display list that the hardware uses to draw graphics on the screen. Its main entry point, `DISPLA`, is called once per frame to translate the current game state into a series of drawing commands.

---

## 1. Graphics Rendering Architecture

Tempest's rendering engine is built on two key concepts: a display-state machine and a sophisticated, multi-part double-buffering system.

### 1.1. Display-State Machine (`DSTATE` and `DROUTAD`)

Similar to how `ALEXEC` manages game states, `ALDIS2` manages *display* states. The Zero Page variable `QDSTATE` determines what screen to draw. The `DSTATE` subroutine acts as a dispatcher, looking up the appropriate rendering routine in the `DROUTAD` (Display Routine Address Table) and executing it.

This allows the game to switch between completely different screens, such as:
-   The main gameplay display (`DENORM`).
-   The high score table (`LDRDSP`).
-   The Atari logo (`LOGPRO`).
-   The system configuration screen (`DSPSYS`).

### 1.2. Double-Buffering System

To prevent flickering and screen tearing, the display list is not built directly in the memory the Vector Generator is reading from. Instead, it uses a double-buffering approach.

The display is broken into several logical layers (player, shots, enemies, well, stars, info text), and each layer has its own pair of buffers (Buffer A and Buffer B).

The process, managed by `SBCLOG` and `SBCSWI`, is as follows:
1.  **`SBCLOG` (Sub-Buffer Control Log):** Before drawing a layer (e.g., the enemies), this routine is called. It checks which of the two enemy buffers is currently active (being drawn by the hardware) and sets up the `VGLIST` pointer to point to the *inactive* one.
2.  **Drawing:** The enemy drawing routine (e.g., `DSPINV`) writes its vector commands into this safe, inactive buffer.
3.  **`SBCSWI` (Sub-Buffer Control Switch):** After drawing is complete, this routine is called. It modifies a master pointer table, telling the Vector Generator to start reading from the newly filled buffer on the next frame. It also flips a flag to mark the other buffer as inactive, making it available for the next frame's drawing cycle.

The final display list that the hardware executes is simply a short list of `JSRL` (Jump to Subroutine, Long) instructions, with each `JSRL` pointing to the currently active buffer for each layer. This is an elegant and efficient way to build a complex, layered 2.5D scene with vector graphics.

---

## 2. Main Gameplay Rendering (`DENORM`)

When the game is in its main play state (`QDSTATE` = `CDPLAY`), the `DENORM` subroutine is executed. This routine orchestrates the drawing of every single element of the game screen in a specific back-to-front order to ensure correct layering.

**Execution Flow of `DENORM`:**
1.  **`DSPCUR`:** Draws the player's ship ("Blaster" or "Cursor").
2.  **`DSPCHG`:** Draws player and enemy shots.
3.  **`DSPINV`:** Draws enemies that are on the rim of the well.
4.  **`DSPEXP`:** Draws all active explosions.
5.  **`DSPNYM`:** Draws "Nymph" type enemies.
6.  **`INFO`:** Calls the master informational display routine (from `ALSCO2.MAC`) to draw scores, lives, and text messages.
7.  **`DSPWEL`:** Draws the geometric well/tube itself.
8.  **`DSPENL`:** Draws enemies that are currently inside the well.
9.  **`DSTARF`:** Draws the moving starfield in the background.

After all layers are drawn to their respective back-buffers, the routine updates the master display list to point to these newly created buffers, making them visible on the next screen refresh.

---

*(More subroutines, detailing the drawing logic for each specific game element, will be documented as the rest of the file is analyzed.)*

---

## 3. Component Drawing Subroutines

These are the individual routines called by `DENORM` to draw each layer of the game world.

### 3.1. `DSPWEL` (Display Well)

This is one of the most complex and clever routines in the rendering engine. It is responsible for drawing the geometric "well" and managing its dynamic colors.

**Functionality:**
1.  **Optimized Rebuilding:** It first checks the `ROTDIS` flag. The well's vector geometry is only rebuilt (by calling `BLDWEL`) if the level changes. For all other frames, it reuses the existing geometry, which is a major optimization.
2.  **Color Poking:** The routine's main job is to dynamically set the color of each line segment of the pre-built well. It does this by iterating through each "spoke" of the well and "poking" new color values directly into that spoke's display list in RAM.
3.  **"Flashlight" Effect:** It checks the player's position (`CURSL1`, `CURSL2`) and sets the color of the spokes under and adjacent to the player's ship to `CURCOL` (yellow). This creates the iconic "flashlight" effect that illuminates the part of the well in front of the player.
4.  **Enemy Pulsing:** It checks the `SPOKST` array to see if an enemy is on a given spoke. If so, it pulses the color of that spoke's lines, signaling danger to the player.
5.  **Bonus Flash:** It also handles the `BOFLASH` effect, which makes the well flash different colors when a bonus life is awarded.

This method of separating geometry generation from color management allows for highly dynamic visual effects without the overhead of re-calculating hundreds of vectors every frame.

### 3.2. `DSPNYM` (Display Nymphs)

This routine draws the enemies that are currently inside the well (Tankers and Spikers are classified as "Nymphs" when in this state).

**Functionality:**
1.  It iterates through the `NNYMPH` object table.
2.  For each active Nymph, it performs a **"fake" 3D projection**. Instead of performing expensive 3D rotation and projection calculations, it uses the Nymph's Y-coordinate (its depth in the well) to directly calculate a scale factor. This is a powerful optimization for objects that are constrained to move along a predictable Z-axis path.
3.  It determines which spoke of the well the Nymph is on.
4.  It calculates the Nymph's final X/Z screen coordinates based on its spoke and the "fake" projection scale.
5.  It calls the low-level vector drawing routines to render the Nymph's shape (a simple diamond) at that position.

### 3.3. `DSPEXP` (Display Explosions)

This routine is responsible for drawing all standard explosions, such as when a shot hits an enemy or when the player's ship is destroyed. The explosion effect is a complex, table-driven animation.

**Functionality:**
1.  It iterates through the `NEXPLO` explosion object table.
2.  For each active explosion, it performs a full 3D-to-2D projection of its coordinates using the `WORSCR` subroutine.
3.  It then determines the type of explosion (e.g., standard hit, player death) and looks up the current animation frame from a series of tables:
    -   `TSPTIM`: Defines the duration (number of frames) for each picture in the animation sequence.
    -   `TSPCOD`: Contains a "special function" code for each frame.
    -   `PICPNTR`: A table of pointers to the actual vector shape data for each animation frame (e.g., `SPLAT1`, `SPLAT2`, etc.).
4.  **Special Functions:** If the `TSPCOD` table specifies a special function for the current frame, it calls the `SPECIAL` subroutine. This dispatcher can then call other routines to create unique effects:
    -   `ROTCOL`: Rotates the color palette for the explosion, creating the cycling color effect seen when the player dies.
    -   `SETSHR`: Initializes the "shrapnel" effect for certain explosions.
5.  Finally, it adds a `JSRL` to the display list, pointing to the correct vector picture for the current animation frame, drawn at the projected coordinates.

### 3.4. `DSBOOM` (Display Boom)

This routine draws the dramatic "Superzapper" effect. It does not draw game objects, but rather a dynamic particle field that radiates from the center of the screen.

**Functionality:**
1.  It first blanks the screen and resets the viewpoint to create a clean canvas.
2.  It iterates through the `NPARTI` (10) particles in the particle object table.
3.  For each active particle, it gets its 3D coordinates and projects them to the screen.
4.  It then draws a vector from the center of the screen to the particle's projected position.
5.  It also draws a bright dot at the end of each vector.
6.  The brightness and color of each line and dot are calculated dynamically, creating a vibrant, energetic explosion that appears to fill the screen.

---

## 4. Geometric Data Tables

This file also contains the raw, hard-coded coordinate data that defines the shapes of the various levels (wells). Tables like `NEWLIX` and `LCIRCL` contain the X/Y coordinates for the vertices on the rim of the circular, square, and other well shapes. This data is read by the `BLDWEL` routine when a new level begins to construct the well's vector geometry.

---
## 5. Core 3D Projection (`WORSCR`)

The `WORSCR` (World to Screen) subroutine is the heart of Tempest's 3D engine. It is responsible for taking a point in 3D "world space" and calculating where it should appear on the 2D screen.

**Inputs:**
-   `PXL`, `PYL`, `PZL`: The X, Y, Z world coordinates of the point to be projected.
-   `EXL`, `EYL`, `EZL`: The X, Y, Z world coordinates of the camera or "eye."

**Functionality:**
The routine implements a standard perspective projection transformation.
1.  **Calculate Deltas:** It finds the difference between the point's coordinates and the eye's coordinates.
2.  **Calculate Depth Factor:** It calculates a depth factor based on the distance of the point from the eye along the Y-axis (`Factor / (PYL - EYL)`). This is the key step that makes objects farther away appear smaller.
3.  **Offload to Math Box:** It uses the dedicated **Math Box** hardware accelerator to perform the multiplications required for the projection. It loads the operands into the Math Box registers and waits for the `MSTAT` flag to indicate the calculation is complete. This offloading is a critical optimization that frees up the main 6502 processor.
4.  **Calculate Screen Coordinates:** It retrieves the results from the Math Box and calculates the final screen X and Z coordinates.

**Output:**
-   `SXL`/`SXH`: The final 16-bit X coordinate on the 2D screen.
-   `SZL`/`SZH`: The final 16-bit Z coordinate on the 2D screen (which corresponds to the vertical axis of the monitor).

---
## 6. Component Drawing Subroutines (Continued)

These are the remaining routines that draw the primary game objects. They all follow a similar pattern: iterate through an object table, project the object's 3D coordinates to 2D screen space using `WORSCR`, and then call a generic drawing routine (`DRAPIC`) to render the object's shape.

### 6.1. `DSPCUR` (Display Cursor)
Draws the player's ship (the "Blaster"). It projects the player's single set of coordinates and calls `DRAPIC` with the pointer to the player's shape data (`PTCURS`).

### 6.2. `DSPCHG` (Display Charges)
Draws all active player and enemy shots by iterating through the `NSHOTS` object table.

### 6.3. `DSPINV` (Display Invaders)
Draws the enemies that are on the rim of the well (Flippers and Tankers). It iterates through the `NINVAD` object table and draws the correct picture based on the enemy's type and animation frame.

### 6.4. `DSTARF` (Display Starfield)
Draws the background starfield. It iterates through the `NSTARS` table, updates each star's position to create movement, projects its new 3D coordinates, and draws a single dot on the screen for each star.

### 6.5. `BLDWEL` (Build Well)
This routine is called only when a new level begins. It is responsible for constructing the static vector geometry of the well itself. It reads the raw vertex data for the current level shape from the geometric data tables and generates a complete display list of `VGVCTR` (Vector) commands to draw all the "spokes" and "rungs" of the well. This list is then saved and reused by `DSPWEL` on every subsequent frame, which only modifies its colors.

---

This concludes the documentation for `ALDIS2.MAC`. 