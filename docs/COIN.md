# Tempest: Core Systems Documentation

## 3. Coin and Credit Logic (`COIN65` Library)

This document details the coin acceptance and credit management system of *Tempest*. The logic is not bespoke to the game but is an implementation of Atari's generic **`COIN65`** library, a reusable module for the 6502 processor. The primary entry point to this library, called every frame by the IRQ handler, is the `MOOLAH` subroutine. The binary for this library is likely `ALCOIN.MAC`.

---

### 3.1. Purpose and Overview

The purpose of the `COIN65` library is to provide a robust, reliable, and cheat-resistant system for processing inputs from physical coin mechanisms and managing the player's credits. It handles switch debouncing, enforces operator-defined pricing, and provides the main game loop with a simple, clean credit count.

-   **Context of Use:** The `MOOLAH` routine is called on every single frame by the IRQ handler (`ALHARD.MAC`). It continuously monitors the coin slot switches. The main game logic in `ALEXEC.MAC` reads the output of this system (`$$CRDT`) to determine if a new game can be started.
-   **Interfaces:**
    -   **Input:** The library reads the state of the physical coin switches from `$COINA`, which is populated by the IRQ handler. It also reads pricing and bonus settings from other memory locations set by the operator.
    -   **Output:** The library's primary output is the 8-bit master credit count stored in the `$$CRDT` variable. It also drives timers for the physical, electro-mechanical coin counters in the cabinet.

---

### 3.2. Architecture and Structure

The system is architected as a "black box" library that is serviced once per frame.

1.  **IRQ Handler (`ALHARD.MAC`):** Reads the raw hardware switch state and calls `MOOLAH`.
2.  **`MOOLAH` Subroutine (`COIN65` Library):** Executes its internal state machine, performing debouncing and credit calculation. It updates the `$$CRDT` variable.
3.  **Game Logic (`ALEXEC.MAC`):** Reads `$$CRDT` when the player presses a Start button.
4.  **Display Logic (`ALSCO2.MAC`):** Reads `$$CRDT` and `$CMODE` to show the player the current credit status.

---

### 3.3. Data Structures and Storage (Zero Page RAM)

The `COIN65` library and the surrounding game logic rely on a set of variables in the 6502's Zero Page for communication and state management.

| Label    | Address | Description                                                                                                                                                             |
| :------- | :------ | :---------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `$$CRDT` | `$0006`   | **Master Credit Count:** The primary output of the system. An 8-bit value holding the number of credits the player has.                                                 |
| `$COINA` | `$0008`   | **Coin Input State:** A bitfield where the IRQ handler places the raw state of the coin switches. The `MOOLAH` routine reads this as its primary input.               |
| `$CMODE` | `$0009`   | **Coin Mode:** An operator-set value (from DIP switches) that determines the pricing. e.g., 0=Free Play, 1=1 Coin/2 Plays, 2=1 Coin/1 Play, 3=2 Coins/1 Play.       |
| `$CNSTT` | `$000E`   | **Coin Status/Timers:** A 3-byte array used by the debouncing state machine. Each byte tracks the stability of a coin switch over several frames.                     |
| `$PSTSL` | `$0011`   | **Post-Slam Timers:** A 3-byte array of timers used by the anti-cheat logic to invalidate a coin if a slam occurs immediately after insertion.                        |
| `$LMTIM` | `$000D`   | **Pre-Slam Lockout Timer:** A master timer that locks out the entire coin system for several seconds if a slam occurs, preventing "slamming for credit."              |
| `$CCTIM` | `$0014`   | **Counter Timers:** A 3-byte array of timers that control the duration of the pulse sent to the physical, electro-mechanical coin counters inside the cabinet.         |
| `$CNCT`  | `$0018`   | **Coin Count (Units):** A temporary counter used when the game is in "units" pricing mode.                                                                          |
| `$BCCNT` | `$0017`   | **Bonus Coin Counter:** Tracks coins inserted towards earning a bonus credit.                                                                                           |
| `$BC`    | `$0019`   | **Bonus Credits:** The number of bonus credits earned but not yet used.                                                                                                 |

---

### 3.4. Algorithms and Processes (The `MOOLAH` Routine)

Based on the `COIN65.MAC` source documentation, the `MOOLAH` routine executes the following logic every frame for each coin mechanism:

1.  **Check for Slam Lockout:** It first checks the master slam timer (`$LMTIM`). If this timer is active, the entire routine is skipped, preventing any coin processing.
2.  **Debounce Coin Switch:** It runs a sophisticated debouncing state machine for each coin switch.
    -   A coin switch signal must be continuously present for a specific duration (e.g., >16ms) to be considered potentially valid. This filters out transient electrical noise.
    -   The signal must also not be present for *too* long (e.g., <800ms). This detects a stuck switch and prevents it from awarding infinite credits.
    -   A valid coin event is only registered on the clean transition from "off" to "on" and back to "off".
3.  **Check for Post-Coin Slam:** If a debounce-valid coin is detected, it checks the post-slam timer (`$PSTSL`). If a slam occurred within a few frames of the coin being validated, the coin is ignored, but the system is not locked out.
4.  **Award Credit:** If a coin passes all checks:
    -   The routine reads the pricing mode from `$CMODE`.
    -   It uses a lookup table to determine how many "units" the specific coin mechanism is worth and adds this to the `$CNCT` temporary counter.
    -   It processes bonus logic via `$BCCNT` if applicable.
    -   It then calls the `$CNVRT` (Convert) subroutine.
5.  **Convert Units to Credits (`$CNVRT`):**
    -   This routine compares the number of units in `$CNCT` against the units required for a credit in the current `$CMODE`.
    -   If enough units have been accumulated, it decrements `$CNCT` by the required amount and increments the final `$$CRDT` variable.
    -   It also handles "2 for 1" pricing, where it might increment `$$CRDT` twice.
6.  **Pulse Mechanical Counter:** If configured, it activates a timer in `$CCTIM` to send an electrical pulse to the correct physical coin meter inside the cabinet.

---

### 3.5. Configuration and Pricing Modes

The operator can configure the coin logic via DIP switches, which are read by the game's diagnostic routines and stored in `$CMODE`.

| `$CMODE` Value | Pricing String        | Behavior                                        |
| :------------- | :-------------------- | :---------------------------------------------- |
| `0`            | FREE PLAY             | The game forces `$$CRDT` to 2.                  |
| `1`            | 1 COIN 2 PLAYS        | *Technically supported but not a default option.* |
| `2`            | 1 COIN 1 PLAY         | Standard single-credit mode.                    |
| `3`            | 2 COINS 1 PLAY        | Requires two coin events to award one credit.     |
| *Other*        | *Flexible Units Mode* | Uses the unit counters for complex pricing.       |

---

### 3.6. Integration with Other Systems

-   **Called By (`ALHARD.MAC`):** The `MOOLAH` routine is called every frame from the IRQ handler. This is its only entry point.
-   **Read By (`ALEXEC.MAC`):** The main game loop checks `$$CRDT` in the `PROCRE` subroutine when a start button is pressed. If credits are available (`$$CRDT > 0`), it decrements the count and starts a new game.
-   **Displayed By (`ALSCO2.MAC`):** The `DSPCRD` routine reads `$CMODE` to display the pricing rule text, and reads `$$CRDT` to display the number of credits available to the player. 