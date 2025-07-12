# Tempest: COIN65.MAC - Universal Coin Handler

**File:** `COIN65.MAC`
**Purpose:** This file is a "universal" coin-handling library for Atari 6502-based arcade games. It is a highly configurable, reusable module designed to manage all aspects of coin insertion, credit management, pricing schemes, and anti-cheating (slam) logic.

---

## 1. Library Architecture and Usage

This file is not a standalone program but a library intended to be included in a larger project. It is driven by a single main entry point, `MOOLAH`, which is expected to be called repeatedly by the host game's main interrupt service routine (IRQ).

The library's behavior is customized through a large set of assembler flags that must be defined *before* the file is included. This allows a game developer to tailor the coin logic to the specific needs of their game's hardware and pricing structure.

## 2. Configuration Flags

These flags control the conditional assembly of the module.

| Flag | Values | Description |
|---|---|---|
| `MECHS` | 1, 2, or 3 | Defines the number of physical coin mechanisms the cabinet has. |
| `MODES` | 0 or 4 | Toggles between two major pricing systems. `4` enables standard modes like "1 Coin, 1 Play", "2 Coins, 1 Play", etc. `0` enables a more flexible "units" system where different coins can be worth different numbers of units. |
| `BONADD`| 0 or 1 | Enables (`1`) or disables (`0`) the "bonus adder" functionality, where the game awards an extra credit after a certain number of coins have been inserted. |
| `EMCTRS`| 0, 1, 3 | Configures support for physical electro-mechanical coin counters. |
| `MULTS` | 0 or 1 | Enables (`1`) the "Delman standard" multipliers for the unit-based pricing system. |
| `SLAM` | 0 or 1 | Defines the polarity of the "slam switch" input (a sensor that detects if the machine is being shaken or tilted). |
| `COIN` | 0 or 1 | Defines the polarity of the coin switch inputs. |

*(Tempest uses the default configuration for most of these settings).*

## 3. Required RAM Interface

For the library to function, the host game (`ALCOMN.MAC` in Tempest's case) must allocate a specific set of variables in the 6502's Zero Page RAM. The library uses these locations to store its state.

**Key Required Variables:**
-   **`$$CRDT`**: The master credit counter. This is the main output of the library.
-   **`$CMODE`**: A byte where the pricing mode, selected by the operator via DIP switches, is stored. The library reads this to determine how many credits to award for a coin.
-   **`$COINA`**: The memory-mapped address of the coin switch hardware inputs.
-   **`$LAM`**: The address of the slam switch input.
-   **`$LMTIM`**: A timer used by the slam switch logic to ignore coin inputs for a short period after a slam event is detected, preventing cheating.
-   **`$CNSTT`**: A set of timers used to debounce the physical coin switches, ensuring one coin drop adds only one credit.
-   **`$BCCNT` / `$BC`**: Counters for the bonus adder feature.

---

*(The internal logic of the main `MOOLAH` processing routine will be documented as the rest of the file is analyzed.)*

---

## 4. `MOOLAH` Processing Routine

The `MOOLAH` subroutine is the heart of the library. It is called on every interrupt and performs the entire sequence of checking for, validating, and awarding credit for a coin insertion.

**Execution Flow of `MOOLAH`:**

1.  **Iterate Mechs:** The routine begins by looping through each configured coin mechanism (from right to left).

2.  **Debounce Coin Switch:** For each mechanism, it executes a sophisticated software debounce algorithm. This state machine uses a dedicated byte in RAM (`$CNSTT`) to track the state of the switch over time. A coin is only considered "valid" if the switch signal is present for a specific duration (more than ~16ms but less than ~800ms), and is preceded and followed by a clean "coin absent" signal. This makes the system highly resilient to electrical noise and physical switch glitches.

3.  **Check Slam Switch (Anti-Cheat):** During the entire process, the routine constantly checks the state of the cabinet's slam/tilt switch.
    -   If a slam is detected shortly *before* a coin is fully registered, a master timer (`$LMTIM`) locks out the entire coin system for several seconds to prevent "slamming for credit."
    -   If a slam is detected shortly *after* a coin is validated, a per-mechanism timer (`$PSTSL`) invalidates that specific coin, but does not lock out the system.

4.  **Calculate Credit:** If a coin passes all the debounce and slam checks, the routine proceeds to award credit.
    -   It first reads the pricing mode from the `$CMODE` variable (which holds the operator's DIP switch settings).
    -   **Unit Mode (`MODES=0`):** If using the flexible "units" system, it checks which mechanism the coin came from and adds the appropriate number of units (e.g., right slot = 4 units, center slot = 2 units) to a temporary counter, `$CNCT`.
    -   **Standard Mode (`MODES=4`):** If using the standard pricing modes, it uses the mode to determine how to update the final credit count directly. For example, in "2 Coins per Play" mode, it will only increment the final credit after two valid coin events.
    -   **Bonus Adder (`BONADD=1`):** If the bonus adder is enabled, it also increments a separate counter (`$BCCNT`). When this counter reaches a pre-set threshold, the game awards a bonus credit by incrementing `$BC`.

5.  **Update Final Credits:** In the final step, the routine converts the units in `$CNCT` and any bonus credits in `$BC` into final game credits, which are stored in `$$CRDT`. This is the variable the main game reads to see if a player can start a game.

6.  **Update Physical Counters:** If configured (`EMCTRS > 0`), the routine also manages the timers (`$CCTIM`) that send electrical pulses to the physical, electro-mechanical counters inside the cabinet.

---

This concludes the documentation for `COIN65.MAC`. 