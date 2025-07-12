# Legacy and Superseded Files

This document provides a brief overview of files in the codebase that appear to be older, superseded versions of the primary modules. While they are part of the source history, they are not used in the final build of the documented version of the game. A full, detailed analysis is not provided as their functionality is covered by the documentation for their newer counterparts.

---

## `ALDISP.MAC`

-   **Superseded by:** `ALDIS2.MAC`
-   **Purpose:** [Display List Manager](./SYSTEMS.md#1-vector-engine).
-   **Analysis:** This is an earlier version of the main rendering engine. The file structure, key subroutines (`DISPLAY`, `DENORM`), double-buffering architecture, and display state machine are all functionally identical to those found in `ALDIS2.MAC`.

## `ALSCOR.MAC`

-   **Superseded by:** `ALSCO2.MAC`
-   **Purpose:** [Scoring & Gameplay Module](./GAME_STATE_FLOW.md).
-   **Analysis:** An earlier version of the core gameplay and UI text module.

## `ALHARD.MAC`

-   **Superseded by:** `ALHAR2.MAC`
-   **Purpose:** [Interrupt Request (IRQ) Handler](./SYSTEMS.md).
-   **Analysis:** An earlier version of the main hardware interrupt service routine.

## `ALTEST.MAC`

-   **Superseded by:** `ALTES2.MAC`
-   **Purpose:** [Self-Test and Diagnostics](./GAME_STATE_FLOW.md#12-csystm---system-test-mode).
-   **Analysis:** An earlier version of the operator's test mode.

## `ALDIAG.MAC`

-   **Purpose:** Diagnostics.
-   **Analysis:** This file contains additional hardware diagnostic routines. It is not included in the main game's linker command and was likely used as a standalone tool for hardware debugging during development or manufacturing. 