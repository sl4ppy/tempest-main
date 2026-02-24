# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is a **reconstruction-grade reverse-engineering** of Atari's 1981 Tempest arcade game. It contains:
1. **Original 6502 assembly source code** in `/source` (`.MAC` files)
2. **Comprehensive documentation** in `/docs` (~37 files, ~11K lines) derived from reverse-engineering the source
3. **A modern recreation plan** (`TEMPEST_DEVELOPMENT_PLAN.md`) for rebuilding in C++20 on Windows

There is no runnable modern codebase yet — only the original assembly and documentation.

## Documentation Navigation

Start here, in order:
1. `docs/SYSTEMS.md` — **Most important.** Vector graphics engine, Mathbox co-processor, IRQ handler, sound engine, all core systems
2. `docs/GAME_STATE_FLOW.md` — Complete 19-state game state machine with transitions
3. `docs/ENTITIES.md` — All game entities (player, enemies, projectiles) with behavior/AI specs
4. `docs/PLAYFIELD.md` — "Well" geometry, procedural generation, rendering pipeline
5. `docs/DATA_ASSETS.md` — Sound data, vector shapes, text strings (4 languages)
6. `docs/BUILD_AND_HARDWARE.md` — Original build tools (MAC65 assembler, LINKM linker), hardware specs

Domain-specific lookups:
- **Graphics**: `SYSTEMS.md` sections 1-2, `PLAYFIELD.md`
- **Audio**: `SYSTEMS.md` section 8, `ALSOUN.md`
- **Game logic**: `ALEXEC.md`, `ALWELG.md`, `GAME_STATE_FLOW.md`
- **Hardware**: `BUILD_AND_HARDWARE.md`, `SYSTEMS.md` sections 3-7
- **Testing plan**: `TESTING_STRATEGY.md`, `IMPLEMENTATION_DETAILS.md`

## Source Code Structure (`/source`)

Key assembly files (6502, `.MAC` extension):
| File | Role |
|---|---|
| `ALEXEC.MAC` | Main game executive — state machine, main loop |
| `ALWELG.MAC` | Core gameplay logic, enemy AI, player controls |
| `ALDIS2.MAC` | Display list compiler (vector graphics) |
| `ALVROM.MAC` | Vector ROM — all shape definitions |
| `ALSOUN.MAC` | Sound engine (data-driven synthesis) |
| `ALHAR2.MAC` | Hardware abstraction & IRQ handler |
| `ALSCO2.MAC` | Score management and high score table |
| `ALCOIN.MAC` | Coin handling and credits |
| `ALLANG.MAC` | Language/text string data |
| `ALTES2.MAC` | Operator test/diagnostic mode |

Supporting files: `ALCOMN.MAC` (common defs/macros), `HLL65.MAC` (high-level macros), `VGMC.MAC` (vector generator macros), `MBUCOD.V05` (Mathbox microcode), `STATE2.MAC` (VG state machine microcode).

## Architecture Concepts

**State machine**: 19 game states dispatched via jump table (`ROUTAD`). States prefixed with `C` (e.g., `CPLAY`, `CBOOM`, `CNEWGA`). See `docs/GAME_STATE_FLOW.md`.

**Vector graphics pipeline**: Game logic -> Mathbox 3D-to-2D transform -> Display list build -> Hardware VG executes asynchronously (double-buffered).

**Sound system**: Fully data-driven. Up to 8 parallel sequences per effect, each controlling one POKEY parameter at 60Hz.

**Naming conventions**: All-caps labels (`QSTATE`, `ROUTAD`). Routines use verb+subject (`MOVCUR`, `FIREPC`). File prefix `AL` = "Atari Locals".

## Modern Recreation Plan

Target stack (from `TEMPEST_DEVELOPMENT_PLAN.md`):
- **C++20** with MSVC toolchain
- **CMake** build system, **vcpkg** package management
- **Direct3D 11/12** for vector rendering
- **WASAPI** for low-latency audio
- **Google Test** + **Catch2** for testing
- **Dear ImGui** for debug UI

Planned source layout: `src/{core,cpu,mathbox,vector,audio,platform,game}`, `tests/{unit,integration,fixtures}`

Key hardware to emulate: 6502 CPU (1.512 MHz), Mathbox (4x AMD 2901), Vector Generator, dual POKEY sound chips, EAROM persistence.
