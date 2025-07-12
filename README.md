---
title: "Tempest - Source Code Documentation"
project_number: 28903
programmer: "Dave Theurer"
date: "1981-12-17"
---

# Tempest: Reconstruction-Grade Documentation

This repository contains the original 6502 assembly source code for the iconic 1981 Atari vector-based arcade game, **Tempest**, along with a complete, reconstruction-grade documentation package derived from reverse-engineering the source.

The goal of this project is to provide a comprehensive blueprint that would allow for the full reconstruction of the original game, covering its hardware interactions, software architecture, and game logic.

## Documentation

The documentation is structured to guide a developer from a high-level overview of the system down to the specifics of each component. All documentation is located in the `/docs` directory.

### Core Systems

These documents describe the fundamental pillars of the Tempest engine.

| File | Description |
|---|---|
| [**`docs/HARDWARE.md`**](./docs/HARDWARE.md) | Details the hardware abstraction, focusing on the main IRQ handler which drives all time-sensitive I/O, including input polling and calling other subsystems. |
| [**`docs/MATHBOX.md`**](./docs/MATHBOX.md) | Describes the custom "Mathbox" co-processor (built from AMD 2901 bit-slice ALUs) responsible for offloading heavy 3D vector calculations from the 6502. |
| [**`docs/VECTOR_ENGINE.md`**](./docs/VECTOR_ENGINE.md) | Explains the final rendering pipeline stage, where the CPU builds a display list that the independent Vector Generator hardware executes to draw lines on the screen. |
| [**`docs/SOUND_ENGINE.md`**](./docs/SOUND_ENGINE.md) | Covers the procedural, data-driven sound engine that synthesizes all audio effects in real-time using the POKEY sound chip. |
| [**`docs/COIN.md`**](./docs/COIN.md) | Documents the coin validation, anti-cheat, and credit management system, which is based on Atari's generic `COIN65` library. |

### Gameplay and UI

These documents cover the game's structure, rules, and player experience.

| File | Description |
|---|---|
| [**`docs/UI.md`**](./docs/UI.md) | An exhaustive reference for every screen and UI state, including the Title Screen, High Score Entry, Skill Level Select, In-Game HUD, and operator-facing diagnostic menus. |
| [**`docs/PLAYFIELD.md`**](./docs/PLAYFIELD.md) | Details the "Well" (the playfield), including its geometry, procedural generation, rendering pipeline, the inter-level "warp" animation, and the dynamic collision system. |
| [**`docs/ENTITIES.md`**](./docs/ENTITIES.md) | Provides a detailed breakdown of every enemy and game object, from the player's ship to the various enemies (Flippers, Tankers, Spikers, etc.), documenting their behavior, animation, and interaction with the game world. |

### Source Code & Data Reference

| File | Description |
|---|---|
| [**`docs/`**](./docs/) | The `docs` directory contains markdown documentation for nearly every `.MAC` source file, providing a line-by-line analysis of the code. |
| [**`docs/VECTOR_SHAPES.md`**](./docs/VECTOR_SHAPES.md) | A reference for the vector shape data used to draw all game objects. |
| [**`docs/LEVEL_DATA.md`**](./docs/LEVEL_DATA.md) | Documents the data and logic for level progression and difficulty scaling. |

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
        G[Game Logic<br/>(ALEXEC.MAC)]
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
| `ALEXEC.MAC` | The main game executive. Contains the top-level game state machine and main loop. | `docs/ALEXEC.md` |
| `ALDISP.MAC` | The core of the Vector Graphics engine. Compiles the display list for the VG hardware and contains the driver for the Mathbox. | `docs/VECTOR_ENGINE.md` |
| `ALWELG.MAC` | "Well Generator". Contains the logic for procedurally generating the playfield for each level. | `docs/PLAYFIELD.md` |
| `ALVROM.MAC` | "Vector ROM". Contains the raw data definitions for all vector shapes in the game. | `docs/VECTOR_SHAPES.md` |
| `ALSOUN.MAC` | The sound engine and sound effect data. | `docs/SOUND_ENGINE.md` |
| `ALHARD.MAC` | The main IRQ handler. The "heartbeat" of the game. | `docs/HARDWARE.md` |
| `ALCOIN.MAC` | A binary blob for the coin handling logic. The source is likely `COIN65.MAC`. | `docs/COIN.md` |
| `COIN65.MAC` | The likely source for Atari's generic coin handling library. | `docs/COIN65.md` |
| `ALTES2.MAC` | Logic for the operator Test Mode. | `docs/ALTES2.md` |
| `ALLANG.MAC` | "Language". Contains all the string data for the UI. | `docs/TEXT_DATA.md` |
| `ALSCO2.MAC` | "Score". Handles score display and the high score table logic. | `docs/ALSCO2.md` | 