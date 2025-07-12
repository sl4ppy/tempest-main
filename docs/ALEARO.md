# Tempest: ALEARO.MAC - EAROM Driver

**File:** `ALEARO.MAC`
**Purpose:** This module is the low-level hardware driver for the EAROM (Electrically Alterable Read-Only Memory). The EAROM is a form of non-volatile memory used to store persistent data, such as high scores and bookkeeping statistics, even when the machine is powered off.

---

## 1. Driver Architecture: A Non-Blocking State Machine

Interacting with an EAROM is a slow process, especially for writing and erasing. If the game were to simply call a "write" subroutine, the entire program would freeze for a noticeable period, waiting for the hardware operation to complete.

To solve this, the `ALEARO` module is designed as a **non-blocking state machine**. It does not perform an entire read or write operation in one go. Instead, its main update routine, `EAUPD`, is designed to be called on every single game frame. On each call, it performs just one small step of the requested operation (e.g., writing a single byte) and then exits. This spreads the slow hardware operation over many frames, allowing the game to continue running smoothly without any freezes or interruptions.

## 2. Public API and Interface

The rest of the game code does not interact with the EAROM directly. Instead, it uses a simple, high-level API provided by this module to request operations.

### 2.1. Request Registers

The API is based on two "request" variables in RAM:
-   **`EAREQU` (EAROM Request):** A bitmask where each bit corresponds to a specific block of data in the EAROM (e.g., bit 0 for initials, bit 1 for high scores, bit 2 for bookkeeping). To request an operation, the game sets the corresponding bit.
-   **`EARWRQ` (EAROM Write Request):** A parallel bitmask that specifies whether the requested operation is a read (`0`) or a write (`1`).

### 2.2. High-Level Subroutines

A set of simple subroutines is provided to make requesting operations easier:
-   **`WRHIIN` (Write High Scores & Initials):** Called after a player enters their name in the high score table. It sets the appropriate bits in `EAREQU` and `EARWRQ` to request a *write* operation for the high score and initials data blocks.
-   **`WRBOOK` (Write Bookkeeping):** Requests a *write* of the bookkeeping statistics.
-   **`REHIIN` (Read High Scores & Initials):** Requests a *read* of all data from the EAROM. This is typically done on startup.
-   **`EAZERO` (Erase and Zero EAROM):** Requests a full erase and zeroing of all data in the EAROM. This is used by the operator via the test menu.

## 3. `EAUPD` - The State Machine Core

The `EAUPD` subroutine is the heart of the driver. It is called on every frame.

**Execution Flow of `EAUPD`:**
1.  **Check for Requests:** It first checks if any request bits are set in `EAREQU`.
2.  **Initiate Operation:** If a new request is found, it sets up its internal state machine.
    -   It identifies which data block to operate on (e.g., high scores).
    -   It looks up the RAM and EAROM addresses for that block from internal tables (`TEASRL`, `TEAX`).
    -   It sets an internal flag, `EAFLG`, to the appropriate starting state (e.g., `EAREAD` or `EAERAS`).
3.  **Process One Step:** It then executes a single step based on the current `EAFLG` state:
    -   If in a `READ` state, it reads one byte from the EAROM and stores it in the corresponding RAM location.
    -   If in a `WRITE` state, it takes one byte from RAM and writes it to the EAROM.
    -   If in an `ERASE` state, it triggers the hardware's erase cycle.
4.  **Update State:** It advances its internal pointers (`EAX`, `EABC`) and updates its state flag (`EAFLG`) for the next step (e.g., after writing a byte, it may set the state to `ERASE` in preparation for the next byte).
5.  **Checksum:** When writing, it calculates a running checksum of the data. When reading, it verifies this checksum. If the checksum does not match, it sets a flag in the `EABAD` variable to indicate that the stored data is corrupt.
6.  **Complete:** Once all bytes in a data block have been processed, it clears the `EAFLG`, marking the operation as complete, and waits for the next request.

---

This concludes the documentation for `ALEARO.MAC`. 