---
title: "Tempest - Source Code Documentation"
project_number: 28903
programmer: "Dave Theurer"
date: "1981-12-17"
---

# Tempest: Reconstruction-Grade Documentation

This repository contains the original 6502 assembly source code for the iconic 1981 Atari vector-based arcade game, **Tempest**, along with a complete, reconstruction-grade documentation package derived from reverse-engineering the source.

The goal of this project is to provide a comprehensive blueprint that would allow for the full reconstruction of the original game, covering its hardware interactions, software architecture, and game logic.

## Navigating the Documentation

All documentation is located in the `/docs` directory. The information has been consolidated into several key documents to provide a clear and structured overview of the game's design. For newcomers, it is recommended to start with `SYSTEMS.md` for a technical foundation, followed by `GAME_STATE_FLOW.md` for a high-level understanding of the game's logic.

---

### 1. Core Architecture & Systems
These documents describe the fundamental pillars of the Tempest engine, from the hardware up to the main software systems.

| File | Description |
|---|---|
| [**`docs/SYSTEMS.md`**](./docs/SYSTEMS.md) | **The single most important technical document.** It provides a detailed overview of all core hardware and software systems, including the Vector Graphics Engine, the Mathbox co-processor, the IRQ-based hardware abstraction layer, the procedural generation philosophy, and the data-driven Sound Engine. |
| [**`docs/BUILD_AND_HARDWARE.md`**](./docs/BUILD_AND_HARDWARE.md) | Provides instructions and context for building the original source code, along with details on the physical hardware configuration of the Tempest arcade cabinet. |

---

### 2. Gameplay & Logic
These documents explain the rules of the game, the flow of logic, and how the player and enemies interact within the world.

| File | Description |
|---|---|
| [**`docs/GAME_STATE_FLOW.md`**](./docs/GAME_STATE_FLOW.md) | **The definitive guide to game logic.** Provides an exhaustive, state-by-state breakdown of the entire game, from power-on to attract mode, gameplay, and game over. Includes a complete state transition diagram. |
| [**`docs/ENTITIES.md`**](./docs/ENTITIES.md) | A comprehensive encyclopedia of every entity in the game. Details the behavior, animation, AI, and state logic for the Player ship, all enemy types (Flippers, Tankers, Spikers, etc.), and projectiles. |
| [**`docs/PLAYFIELD.md`**](./docs/PLAYFIELD.md) | Details the "Well" (the playfield), including its geometry, procedural generation, rendering pipeline, the inter-level "warp" animation, and the dynamic collision system. |
| [**`docs/LEVEL_DATA.md`**](./docs/LEVEL_DATA.md) | Documents the data and logic for level progression and difficulty scaling across the game's 99+ levels. |
| [**`docs/GAME_TUNING.md`**](./docs/GAME_TUNING.md) | A reference for the various constants and tables that control game balance, such as enemy behavior, scoring, and bonus lives. |

---

### 3. Data & Assets
These documents provide a reference for the game's raw data assets.

| File | Description |
|---|---|
| [**`docs/DATA_ASSETS.md`**](./docs/DATA_ASSETS.md) | A master reference for the game's core data assets. Contains the complete tables for all sound effect data, vector shape definitions for all game objects and fonts, and all user-facing text strings in all four languages. |
| [**`docs/DATA_REFERENCE.md`**](./docs/DATA_REFERENCE.md) | Acts as a high-level index for where to find different types of data within the source code and documentation. |

---

### 4. UI & Source Code Analysis
These documents cover the user interface and provide direct analysis of specific source files.

| File | Description |
|---|---|
| [**`docs/UI.md`**](./docs/UI.md) | An exhaustive reference for every screen and UI state, including the Title Screen, High Score Entry, Skill Level Select, In-Game HUD, and the full Attract Mode sequence. |
| [**`docs/ATTRACT_MODE.md`**](./docs/ATTRACT_MODE.md) | A detailed, sequential breakdown of the entire "attract mode" or demo sequence that plays when no one is playing the game. |
| [**`docs/ADDITIONAL_DATA.md`**](./docs/ADDITIONAL_DATA.md) | Contains miscellaneous data and analysis that doesn't fit into the other categories, including details on the EAROM (non-volatile storage) map. |
| [**`docs/LEGACY_FILES.md`**](./docs/LEGACY_FILES.md) | A brief description of original Atari documentation files that were included with the source but are now superseded by this documentation set. |

## System Architecture

The game runs on custom Atari arcade hardware, driven by a **6502 microprocessor**. The architecture is defined by the interplay between the central processor, Read-Only Memory (ROMs), and specialized co-processors for vector graphics generation and mathematics.

```mermaid
graph TD
    subgraph "Hardware"
        A[6502 Main CPU]
        B[Vector Generator]
        C[POKEY Sound Chip]
        D[Mathbox Co-Processor]
        E[Input Hardware]
        F[Vector Display]
    end

    subgraph "Software & Data"
        G[Game Logic<br/>(ALEXEC.MAC / ALWELG.MAC)]
        H[Display List Compiler<br/>(ALDISP.MAC)]
        I[Sound Engine<br/>(ALSOUN.MAC)]
        J[IRQ Handler<br/>(ALHARD.MAC)]
        K[Mathbox Driver<br/>(ALDISP.MAC)]
        L[Vector Shape Data<br/>(ALVROM.MAC)]
    end

    G -- Drives --> H
    G -- Drives --> I
    G -- Receives Input From --> J
    J -- Polls --> E

    H -- Sends Display List To --> B
    B -- Reads Shape Data From --> L
    B -- Draws To --> F

    I -- Sends Commands To --> C

    G -- Calls --> K
    K -- Offloads Calculations To --> D
```

## Source File Overview

The original source code is located in the root directory of this project. The key files are:

| File | Description | See Also |
|---|---|---|
| `ALEXEC.MAC` | The main game executive. Contains the top-level game state machine and main loop. | `docs/GAME_STATE_FLOW.md` |
| `ALDISP.MAC` | The core of the Vector Graphics engine. Compiles the display list for the VG hardware and contains the driver for the Mathbox. | `docs/SYSTEMS.md#1-vector-graphics-engine` |
| `ALWELG.MAC` | Contains the core gameplay logic, enemy AI, and player control routines. | `docs/ENTITIES.md` |
| `ALVROM.MAC` | "Vector ROM". Contains the raw data definitions for all vector shapes in the game. | `docs/DATA_ASSETS.md#2-vector-shape-data` |
| `ALSOUN.MAC` | The sound engine and sound effect data. | `docs/SYSTEMS.md#8-sound-engine-alsounmac` |
| `ALHARD.MAC` | The main IRQ handler. The "heartbeat" of the game. | `docs/SYSTEMS.md#3-hardware-abstraction--irq-handler-alhardmac` |
| `ALCOIN.MAC` | Contains the logic for coin handling and credits. | `docs/SYSTEMS.md#4-coin-and-credit-logic-coin65-library` |
| `ALTES2.MAC` | Logic for the operator Test Mode. | `docs/GAME_STATE_FLOW.md#49-state-csystm-system-test-mode` |
| `ALLANG.MAC` | "Language". Contains all the string data for the UI. | `docs/DATA_ASSETS.md#3-text-and-language-data` |
| `ALSCO2.MAC` | "Score". Handles score display and the high score table logic. | `docs/UI.md` | 