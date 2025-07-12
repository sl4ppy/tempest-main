# Tempest: ALHAR2.MAC - Interrupt Request (IRQ) Handler

**File:** `ALHAR2.MAC`
**Purpose:** This file contains the master Interrupt ReQuest (IRQ) handler for the game. This code does not run as part of the main game loop; instead, it is executed automatically at a fixed frequency (typically 60 times per second) whenever the hardware generates an interrupt. It is responsible for all low-level, time-critical tasks like reading player controls, debouncing switches, and updating system timers.

---

## 1. IRQ Execution Flow

The `IRQ` label marks the entry point for the interrupt service routine (ISR). The routine is carefully structured to be as fast and efficient as possible.

**Execution Flow of the `IRQ` handler:**
1.  **Save State:** It immediately saves the CPU registers (A, X, Y) to the stack so the main program's state is not disturbed.
2.  **System Watchdog:** It performs a stability check. It verifies that the stack pointer is within a valid range. If the stack has grown too deep (a common sign of a software crash), the routine intentionally triggers a hardware `RESET`, rebooting the game. This makes the system extremely robust. It also resets the hardware watchdog timer (`WTCHDG`) to prevent a reset under normal conditions.
3.  **Input Processing:** It reads the state of all physical controls on the machine.
4.  **Output Processing:** It updates the state of all hardware outputs, like coin counters and start button lights.
5.  **Timer Updates:** It increments all global system and gameplay timers.
6.  **VG Management:** It checks if the Vector Generator has finished drawing and restarts it if needed.
7.  **Restore State:** It restores the CPU registers from the stack and executes an `RTI` (Return from Interrupt) instruction to pass control back to the main program.

---

## 2. Input Processing Subsystem

This is the most critical part of the IRQ handler. It reads raw data from the hardware ports and translates it into clean, game-ready state variables in RAM.

### 2.1. Rotary Dial (`Spinner`)

The IRQ reads the player's rotary dial input.
- It reads the current 4-bit potentiometer value from the POKEY sound chip's `ALLPOT` register.
- It compares this new value to the value from the previous interrupt (`OTB` - Old Trackball).
- By subtracting the old value from the new one, it calculates a delta (a signed number representing direction and speed of rotation).
- This delta is added to the `TBHD` (Trackball Head) variable. The main game logic in `ALSCO2.MAC` then reads `TBHD` to determine how far to move the player's ship.

### 2.2. Buttons and Switches

The IRQ reads all other buttons and cabinet switches from the `IN1` and POKEY I/O ports. This includes the fire button, Superzapper button, player start buttons, and the test/diagnostic switches.

A sophisticated **software debounce** routine is then performed. Because physical switches can "bounce" (rapidly open and close multiple times on a single press), this logic ensures that one press registers as exactly one event. It uses the current input, the previous input (`DBSW`), and the last known stable state (`SWSTAT`) to filter out the noise. The final, clean, latched button press state is stored in `SWFINA` for the main game loop to use.

---

## 3. Output and Timer Subsystems

### 3.1. Hardware Outputs

The IRQ is responsible for directly controlling lights and other outputs.
- **Coin Counters:** It checks timers (`$CCTIM`) and sends pulses to the mechanical coin counters in the machine via the `OUT0` port.
- **Start Lights:** It updates the "Player 1/2 Start" button LEDs. It contains logic to keep them steadily lit during gameplay or flash them during attract mode when credits are available. The patterns are stored in the `LITSON` table.

### 3.2. Timers

Since the IRQ runs at a fixed frequency, it is the perfect place to update all game timers. On every interrupt, it increments:
- **`FRTIMR`:** The master frame timer, used by `ALEXEC` to sync the main loop to the display refresh.
- **`$INTCT`:** A general-purpose interrupt counter.
- **`SECOUL/M/H`:** A master up-timer, counting the total seconds the machine has been on.
- **`SECOPL/M/H`:** A gameplay timer, counting the seconds a game has been in progress.

---

This concludes the documentation for `ALHAR2.MAC`. 