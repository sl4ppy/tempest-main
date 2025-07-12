# Tempest: ALTES2.MAC - Self-Test and Diagnostics

**File:** `ALTES2.MAC`
**Purpose:** This module contains the complete logic for the operator's self-test and system configuration mode. This mode is entered when the physical "Test" switch inside the arcade cabinet is activated. It allows an operator to verify hardware functionality, view bookkeeping statistics, and configure game settings.

---

## 1. Test Mode Architecture

The test mode is driven by the `SYSTEM` subroutine, which is called by the main executive when the test switch is detected. `SYSTEM` sets the display state to `CDSYST`, which in turn calls the `DSPSYS` routine to draw all the information on the screen. The mode is interactive, using the rotary dial and fire button as inputs.

## 2. `DSPSYS` - The Main Information Display

The `DSPSYS` subroutine is responsible for drawing all the configuration and bookkeeping information on the screen. It acts as a master display routine that calls numerous helpers:

-   **Coin & Credit Info:** It calls `DSPCRD` to display the configured coin mode (e.g., "FREE PLAY") and other credit information.
-   **DIP Switch State:** It calls `DOPSWI` to show the raw binary state of the two DIP switch banks.
-   **Bookkeeping:** It calls `DBOOKE` to display the machine's statistics, which are read from the EAROM (e.g., total coins inserted, total playtime in hours).
-   **Game Settings:** It displays the currently configured number of lives per game and the difficulty level (Easy/Medium/Hard).

## 3. Interactive Operator Menu

The test mode presents an interactive menu to the operator.
-   **Input:** The operator uses the game's rotary dial to move a cursor and select an option. The fire button is used to activate the selected option.
-   **Options:**
    1.  **`SELF TEST`**: When selected, this option jumps to the `RESET` routine to begin a full, destructive hardware test (RAM and ROM).
    2.  **`ZERO HI SCORES`**: This calls the `EAZHIS` routine (from `ALEARO.MAC`) to erase only the high score and initials portion of the EAROM.
    3.  **`ZERO TIMES`**: This calls the `EAZBOO` routine (from `ALEARO.MAC`) to erase only the bookkeeping (playtime and coin count) portion of the EAROM.

## 4. Hardware Self-Test (`RESET` / `SFTEST`)

When the "SELF TEST" option is chosen, the `RESET` routine is executed. This is a cold start that bypasses the normal game boot and performs a thorough hardware check.

-   It disables interrupts (`SEI`).
-   It performs a destructive read/write test of all system RAM, page by page.
-   If a RAM error is detected, it jumps to the `BRAMREP` routine.
-   After testing RAM, it proceeds to perform a checksum test on all of the game's ROM chips to verify their integrity.
-   Finally, it displays the results of these tests on the screen.

### 4.1. Audio Error Reporting (`BRAMREP`)

This is a fascinating piece of diagnostic design. If the `SFTEST` detects a bad RAM chip, it calls `BRAMREP` (Bad RAM Reporter). This routine does not rely on the screen (which may not be working). Instead, it communicates the error via sound and light.

It generates a specific sequence of high- and low-pitched tones on the POKEY sound chip and flashes the cabinet's LEDs. The exact pattern of tones and flashes indicates precisely which RAM chip failed and even which data bit is bad. This allowed a trained technician to diagnose a hardware fault on the main circuit board without needing an oscilloscope or other advanced tools.

---

This concludes the documentation for `ALTES2.MAC`. 