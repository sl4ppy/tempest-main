# Tempest: Game Tuning and Constants

This document lists the global constants and parameters that define the core tuning and feel of Tempest. These values, primarily defined in `ALCOMN.MAC`, control aspects like scoring, player abilities, and object physics. This data is distinct from the per-level data, as these constants apply globally unless overridden by level-specific parameters.

## Scoring

| Constant | Value | Source File | Description |
| :--- | :--- | :--- | :--- |
| `TUPSCM` | `[...` | `ALEXEC.MAC` | Table of BCD score values for shooting various enemies. Ranges from 50 to 750 points. |
| `BONPTM` | `[...` | `ALWELG.MAC` | Table of BCD bonus point values awarded at the end of a wave. |

## Player and Controls

| Constant | Value | Source File | Description |
| :--- | :--- | :--- | :--- |
| `ILIVES` | 4 | `ALCOMN.MAC` | Initial number of lives given to the player. |
| `PCVELO` | 13 | `ALCOMN.MAC` | The player's maximum speed along the rim of the well. |
| `PCHVEL` | 80 | `ALCOMN.MAC` | The velocity of the player's shots. |
| `PCHRDL` | 4 | `ALCOMN.MAC` | Player shot refire delay. A 4-frame delay between shots. |
| `MAXPCH` | 8 | `ALCOMN.MAC` | Maximum number of player shots allowed on screen at once. |
| `BOOMT` | 180 | `ALCOMN.MAC` | Duration of the Superzapper "boom" effect, in frames (3 seconds). |
| `NZAPCH` | 3 | `ALCOMN.MAC` | Number of Superzappers per life. |

## Object Counts and Limits

| Constant | Value | Source File | Description |
| :--- | :--- | :--- | :--- |
| `NHISCO` | 8 | `ALCOMN.MAC` | Number of high scores kept in the table. |
| `NRANKS` | 99 | `ALCOMN.MAC` | The highest rank a player can achieve. |
| `NEXPLO` | 10 | `ALCOMN.MAC` | Maximum number of simultaneous explosions. |
| `NENESH` | 4 | `ALCOMN.MAC` | Maximum number of enemy shots. |
| `NNYMPH` | 32 | `ALCOMN.MAC` | Maximum number of "nymph" particles for Spikers. |
| `NPARTI` | 10 | `ALCOMN.MAC` | Number of particles in the Superzapper explosion. |

## Timers and Durations

| Constant | Value | Source File | Description |
| :--- | :--- | :--- | :--- |
| `SECOND` | 20 (dec) | `ALCOMN.MAC` | Number of game frames per second (Note: this seems low, likely a game logic tick rate). |
| `ITIMEXP` | 1 | `ALCOMN.MAC` | Initial value for the explosion update timer. |
| `ITIMXP` | 5 | `ALCOMN.MAC` | Number of frames for each "bang" (frame of animation) in an explosion. |
| `TIMAVD` | 120 | `ALCOMN.MAC` | Timer for the "AVOID SPIKES" message (2 seconds). |
| `TIMSUP` | 120 | `ALCOMN.MAC` | Timer for the "NEW SUPERZAPPER" message (2 seconds). |

## Physics and Initial Values

| Constant | Value | Source File | Description |
| :--- | :--- | :--- | :--- |
| `IEYL` | 32 | `ALCOMN.MAC` | Initial Y-coordinate of the camera "eye". |
| `IEZL` | 128 | `ALCOMN.MAC` | Initial Z-coordinate of the camera "eye". |
| `PARLXA` | 32 | `ALCOMN.MAC` | Fractional deceleration of Superzapper particles on the X-axis. |
| `PARLYA` | 32 | `ALCOMN.MAC` | Fractional deceleration of Superzapper particles on the Y-axis. |
| `PARLZA` | 32 | `ALCOMN.MAC` | Fractional deceleration of Superzapper particles on the Z-axis. |

---

## Color Palette

The game uses a fixed palette of 16 colors, with constants defined for clear identification.

| Constant | Value | Description |
| :--- | :--- | :--- |
| `BLUE` | 6 | Standard Blue |
| `GREEN` | 5 | Standard Green |
| `RED` | 3 | Standard Red |
| `YELLOW` | 1 | Standard Yellow |
| `WHITE` | 0 | Standard White |
| `PURPLE` | 2 | Standard Purple |
| `TURQOI` | 4 | Turquoise |
| `BLULET` | 7 | A slightly different shade of blue, often for letters. |
| `PSHCTR` | 8 | Color of the player's shot center. |
| `PDIWHI` | 9 | White color used in the player death explosion. |
| `PDIYEL`| 10 | Yellow color used in the player death explosion. |
| `PDIRED`| 11 | Red color used in the player death explosion. |
| `NYMCOL` | 12 | Color of the "nymph" particles. |
| `FLASH` | 15 | A flashing color value. | 