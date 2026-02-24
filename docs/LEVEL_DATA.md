# Tempest: Level, Wave, and Enemy Data

This document details the data structures that control the behavior and difficulty of each level in Tempest. The game features 99 distinct levels (though the level counter can go higher), with behavior defined by a set of tables that modify base parameters for different ranges of waves. This data is sourced from `ALWELG.MAC`.

The system is highly flexible, allowing for fine-tuned difficulty curves. Instead of a simple grid of properties for each level, the game uses a system of modifications. A set of base parameters is established, and then a master table (`WTABLE`) is processed at the start of each wave to apply level-specific changes to those parameters.

## Level Tuning Parameters

These are the core variables in RAM that are modified by the level data tables. They control the characteristics of the current wave.

| Variable | Description |
| :--- | :--- |
| `WCHARFR` | Frames until an enemy can fire. |
| `WCHAMX` | Maximum number of [enemy shots](./ENTITIES.md#11-enemy-bullet) on screen at once (minus 1). |
| `WFLMIN` | Minimum number of [Flipper](./ENTITIES.md#5-flipper) enemies to spawn. |
| `WFLMAX` | Maximum number of [Flipper](./ENTITIES.md#5-flipper) enemies to spawn. |
| `WPUMIN` | Minimum number of [Pulsar](./ENTITIES.md#9-pulsar) enemies. |
| `WPUMAX` | Maximum number of [Pulsar](./ENTITIES.md#9-pulsar) enemies. |
| `WTAMIN` | Minimum number of [Tanker](./ENTITIES.md#6-tanker) enemies. |
| `WTAMAX` | Maximum number of [Tanker](./ENTITIES.md#6-tanker) enemies. |
| `WSPMIN` | Minimum number of [Spiker](./ENTITIES.md#7-spiker) enemies. |
| `WSPMAX` | Maximum number of [Spiker](./ENTITIES.md#7-spiker) enemies. |
| `WFUMIN` | Minimum number of [Fuseball](./ENTITIES.md#8-fuseball) enemies. |
| `WFUMAX` | Maximum number of [Fuseball](./ENTITIES.md#8-fuseball) enemies. |
| `PULPOT` | [Pulsar](./ENTITIES.md#9-pulsar) "potency" height. |
| `PULTIM` | [Pulsar](./ENTITIES.md#9-pulsar) timer increment. |
| `WTACAR+2` | CAM pattern for one type of enemy. |
| `WTACAR+3` | CAM pattern for another type of enemy. |
| `WINVMX` | Maximum number of "invaders" (general enemies). |
| `NWNYMC` | Number of "nymphs" (the small particles that form [Spikers](./ENTITIES.md#7-spiker)). |
| `NWTELI` | Initial height of the enemy lines at the top of the well. |
| `PUCHDE` | [Pulsar](./ENTITIES.md#9-pulsar) chase delay. |
| `WFLICAM` | The CAM (movement) pattern used by [Flippers](./ENTITIES.md#5-flipper) for the current level. |
| `WPULFI` | Controls whether [Pulsars](./ENTITIES.md#9-pulsar) can fire. |
| `WTTFRA` | The rate at which [Flippers](./ENTITIES.md#5-flipper) change their animation frame. |
| `WINVIL` | Fractional part of the base speed for "invader" type enemies. |
| `WINVIN` | Integer part of the base speed for "invader" type enemies. |
| `TSPIIN` | Speed of [Spikers](./ENTITIES.md#7-spiker). |
| `TCHARIN` | Speed of [enemy shots](./ENTITIES.md#11-enemy-bullet). |

## `WTABLE` Data Structure

`WTABLE` is the master table that orchestrates all level modifications. It is a list of pointers. Each entry in `WTABLE` points to a sub-table and the memory address of the tuning parameter that the sub-table modifies.

Each sub-table (e.g., `TCHARFR`, `TINVIN`) consists of one or more records with the following structure:

- **Byte 1: Encoding Type**: A code that determines how the data is interpreted.
- **Byte 2: Start Wave**: The first wave this record applies to.
- **Byte 3: End Wave**: The last wave this record applies to.
- **Byte 4+: Data**: The actual parameter values, interpreted based on the encoding type.

### Encoding Types

| Code | Name | Description |
| :--- | :--- | :--- |
| `T1` | One Value | A single byte of data is applied to the target parameter for all waves in the specified range. |
| `TZ` | Z-Table | The data is an array of bytes, one for each wave in the range. The game applies the value corresponding to `(current_wave - start_wave)`. |
| `TA` | Add | Takes two data bytes. For each wave in the range, the value is `data1 + (current_wave - start_wave) * data2`. |
| `TB` | Add to Base | Takes one data byte. Adds it to a base value (`WINVIN`). Used for speeds. |
| `TR` | Alternate | Takes two data bytes. Alternates between them for each wave in the range. |
| `TZANDF` | Z-Table with AND | The current wave number is first bitwise ANDed with `0x0F`. The result is then used as an index into a Z-Table. This creates repeating 16-wave patterns. |
| `TE` | End of Table | Marks the end of the sub-table. |

---

## Decoded Level Data Tables

Here are the decoded contents of the sub-tables pointed to by `WTABLE`.

### `TCHARFR` (modifies `WCHARFR` - Invader Fire Rate)
This table controls how frequently enemies can fire. The value is the number of frames between shots.

| Type | Wave Range | Data | Effect |
| :--- | :--- | :--- | :--- |
| `TA` | 1-20 | `80, -3` | Starts at 80 frames, decreasing by 3 each wave. |
| `T1` | 21-64 | `20` | Fixed at 20 frames. |
| `T1` | 65-99 | `10` | Fixed at 10 frames. |

### `TCHAMX` (modifies `WCHAMX` - Max Enemy Shots)
The maximum number of [enemy shots](./ENTITIES.md#11-enemy-bullet) allowed on screen at once is `value + 1`.

| Type | Wave Range | Data | Effect |
| :--- | :--- | :--- | :--- |
| `TZ` | 1-9 | `1,1,1,2,3,2,2,3,3` | Max shots increases from 2 to 4. |
| `T1` | 10-64 | `2` | Max shots is 3. |
| `T1` | 65-99 | `3` | Max shots is 4. |

### `TINVIN` (modifies `WINVIN` - Base Invader Speed)
This is the integer part of the base speed for [Flippers](./ENTITIES.md#5-flipper) and [Tankers](./ENTITIES.md#6-tanker). Higher negative numbers mean faster speed.

| Type | Wave Range | Data | Effect |
| :--- | :--- | :--- | :--- |
| `TA` | 1-8 | `-44, -5` | Speed increases linearly. |
| `TZ` | 9-16 | `-81, -84, ...` | Speed increases wave-by-wave. |
| `TA` | 17-25 | `-81, -3` | Speed increases linearly. |
| `TA` | 26-32 | `-99, -3` | Speed increases linearly. |
| ... | ... | ... | ... |

### `TFLIMI` / `WFLIMX` (modifies `WFLMIN` / `WFLMAX` - Flipper Count)
These tables control the min/max number of [Flippers](./ENTITIES.md#5-flipper) that spawn.

| Table | Type | Wave Range | Data | Effect |
| :--- | :--- | :--- | :--- | :--- |
| `WFLIMI` | `T1` | 1-4 | `1` | Min 1 flipper. |
| `WFLIMI` | `T1` | 5-99 | `0` | Min 0 flippers. |
| `WFLIMX` | `T1` | 1-4 | `4` | Max 4 flippers. |
| `WFLIMX` | `T1` | 5-16 | `5` | Max 5 flippers. |
| `WFLIMX` | `T1` | 17-19 | `3` | Max 3 flippers ([Pulsars](./ENTITIES.md#9-pulsar) appear). |
| ... | ... | ... | ... | ... |

### `WTANMI` / `WTANMX` (modifies `WTAMIN` / `WTAMAX` - Tanker Count)

| Table | Type | Wave Range | Data | Effect |
| :--- | :--- | :--- | :--- | :--- |
| `WTANMI` | `TZ` | 1-4 | `0,0,1,0` | Min [Tankers](./ENTITIES.md#6-tanker) appear on wave 3. |
| `WTANMI` | `T1` | 5-16 | `1` | Min 1 tanker. |
| `WTANMX` | `TZ` | 1-5 | `0,0,1,0,1`| Max [Tankers](./ENTITIES.md#6-tanker) appear. |
| `WTANMX` | `T1` | 6-16 | `2` | Max 2 tankers. |
| ... | ... | ... | ... | ... |

### `CAMWAV` (modifies `WFLICAM` - Flipper Movement CAM)
This table uses the `TZANDF` encoding to create a repeating 16-wave pattern for the [Flipper's](./ENTITIES.md#5-flipper) movement AI.

| Type | Wave Range | Data | Effect |
| :--- | :--- | :--- | :--- |
| `TZANDF` | 1-99 | `[NOJUMP, MOVJMP, ...]` | A repeating sequence of 16 different CAM pointers. Determines if [Flippers](./ENTITIES.md#5-flipper) jump, spiral, etc. |

### `TELIHI` (modifies `NWTELI` - Enemy Line Height)
This also uses `TZANDF` to set the starting height of the enemies at the top of the well, creating different visual patterns for the wells.

| Type | Wave Range | Data | Effect |
| :--- | :--- | :--- | :--- |
| `TZANDF` | 1-99 | `[E0, D8, D4, ...]` | A repeating sequence of 16 different height values. |

---

## Complete Decoded Level Data Tables

All values below are extracted from `ALWELG.MAC`. See [Encoding Types](#encoding-types) for how each type (`T1`, `TZ`, `TA`, `TB`, `TR`, `TZANDF`) works.

### Complete `TCHARFR` (Invader Fire Delay, in frames)

| Waves | Type | Raw Data | Decoded Values |
| :--- | :--- | :--- | :--- |
| 1-20 | `TA` | `80, -3` | 80, 77, 74, 71, 68, 65, 62, 59, 56, 53, 50, 47, 44, 41, 38, 35, 32, 29, 26, 23 |
| 21-64 | `T1` | `20` | All 20 |
| 65-99 | `T1` | `10` | All 10 |

### Complete `TCHAMX` (Max Enemy Shots minus 1)

| Waves | Type | Raw Data | Decoded Values |
| :--- | :--- | :--- | :--- |
| 1-9 | `TZ` | per-wave | 1, 1, 1, 2, 3, 2, 2, 3, 3 |
| 10-64 | `T1` | `2` | All 2 |
| 65-99 | `T1` | `3` | All 3 |

### Complete `TINVIN` (Base Invader Speed, signed)

| Waves | Type | Raw Data | Decoded Values |
| :--- | :--- | :--- | :--- |
| 1-8 | `TA` | `-44, -5` | -44, -49, -54, -59, -64, -69, -74, -79 |
| 9-16 | `TZ` | per-wave | -81, -84, -84, -84, -88, -92, -96, -96 |
| 17-25 | `TA` | `-81, -3` | -81, -84 (note: -81 base, step by -3 per additional wave) |
| 26-32 | `TA` | `-99, -3` | -99, -102, -105, -108, -111, -114, -117 |
| 33-39 | `TA` | `-108, -3` | -108, -111, -114, -117, -120, -123, -126 |
| 40-48 | `TA` | `-110, -1` | -110, -111, -112, ..., -118 |
| 49-64 | `TA` | `-120, -1` | -120, -121, -122, ..., -135 |
| 65-99 | `TR` | `-160, -191` | Alternates: -160, -191, -160, -191, ... |

### Complete `TSPIIN` (Spinner Speed Increment)

| Waves | Type | Raw Data | Decoded Values |
| :--- | :--- | :--- | :--- |
| 1-20 | `TB` | `0` | Add 0 to `WINVIL` (no extra speed) |
| 21-32 | `TB` | `-48` | Add -48 to `WINVIL` |
| 33-48 | `TB` | `-40` | Add -40 to `WINVIL` |
| 49-99 | `TB` | `-48` | Add -48 to `WINVIL` |

### Complete `TCHARIN` (Charge Shot Length)

| Waves | Type | Raw Data | Decoded Values |
| :--- | :--- | :--- | :--- |
| 1-99 | `TB` | `-64` | Add -64 to `WCHARL` |

### Complete Flipper Count (`WFLIMI` / `WFLIMX`)

| Waves | Min | Max |
| :--- | :--- | :--- |
| 1-4 | 1 | 4 |
| 5-16 | 0 | 5 |
| 17-19 | 0 | 3 |
| 20-25 | 0 | 4 |
| 26-99 | 0 | 5 |

### Complete Tanker Count (`WTANMI` / `WTANMX`)

| Waves | Min | Max |
| :--- | :--- | :--- |
| 1 | 0 | 0 |
| 2 | 0 | 0 |
| 3 | 1 | 1 |
| 4 | 0 | 0 |
| 5 | 1 | 1 |
| 6-16 | 1 | 2 |
| 17-26 | 1 | 1 |
| 27-32 | 1 | 1 |
| 33-44 | 1 | 2 |
| 45-99 | 1 | 3 |

### Complete Spinner Count (`WSPIMI` / `WSPIMX`)

| Waves | Min | Max |
| :--- | :--- | :--- |
| 1-3 | 0 | 0 |
| 4 | 1 | 2 |
| 5 | 2 | 3 |
| 6 | 2 | 4 |
| 7-10 | 2 | 4 |
| 11-16 | 2 | 3 |
| 17-19 | 0 | (not set) |
| 20-25 | 1 | 2 |
| 26 | 1 | 1 |
| 27-28 | 1 | 2 |
| 29-30 | 1 | 2 |
| 31 | 1 | 1 |
| 32 | 1 | 2 |
| 35-39 | 1 | 1 |
| 43-99 | 1 | 1 |

### Complete Fuse Count (`WFUSMI` / `WFUSMX`)

| Waves | Min | Max |
| :--- | :--- | :--- |
| 1-10 | 0 | 0 |
| 11-16 | 1 | 1 |
| 17-21 | 0 | 0 |
| 22-25 | 1 | 1 |
| 26 | 0 | 0 |
| 27-32 | 1 | 1 |
| 33-39 | 1 | 4 |
| 40-99 | 1 | 3 |

### Complete Pulsar Parameters

**`WPULPOT` (Pulsar Potency Height):**

| Waves | Value |
| :--- | :--- |
| 1-64 | $A0 |
| 65-99 | $C0 |

**`WPULTIM` (Pulsar Timer Increment):**

| Waves | Value |
| :--- | :--- |
| 1-48 | 4 frames |
| 49-64 | 6 frames |
| 65-99 | 8 frames |

**`TPUCHDE` (Pulsar Chase Delay):**

| Waves | Value |
| :--- | :--- |
| 17 | 40 (PN) |
| 18 | 20 (PC) |
| 19-32 | Alternates 20, 40 |
| 33-39 | 20, 19, 18, 17, 16, 15, 14 |
| 40-99 | Alternates 20, 10 |

### Complete `TWTTFRA` (Flipper Top Flip Rate)

| Waves | Value |
| :--- | :--- |
| 1-32 | 2 frames |
| 33-99 | 3 frames |

### Complete `TINVMX` (Max Active Invaders)

| Waves | Value |
| :--- | :--- |
| 1-99 | 6 |

### Complete `TNYMMX` (Nymph Count / Total Enemies)

| Waves | Values |
| :--- | :--- |
| 1-16 | 10, 12, 15, 17, 20, 22, 20, 24, 27, 29, 27, 24, 26, 28, 30, 27 |
| 17-26 | 20, 21, 22, 23, 24, 25, 26, 27, 28, 29 |
| 27-39 | All 27 |
| 40-48 | 29, 30, 31, 32, 33, 34, 35, 36, 37 |
| 49-64 | 31 + (wave - 49) |
| 65-80 | 35 + (wave - 65) |
| 81-99 | 43 + (wave - 81) |

### Complete `TELIHI` (Enemy Line Height, TZANDF mod 16)

| Mod 16 Index | Value (hex) |
| :--- | :--- |
| 0 | $00 |
| 1 | $00 |
| 2 | $00 |
| 3 | $E0 |
| 4 | $D8 |
| 5 | $D4 |
| 6 | $D0 |
| 7 | $C8 |
| 8 | $C0 |
| 9 | $B8 |
| A | $B0 |
| B | $A8 |
| C | $A0 |
| D | $A0 |
| E | $A0 |
| F | $A8 |

### Fuse Parameters

**`TWFUSC` (Fuse Scan Range):**

| Waves | Value |
| :--- | :--- |
| 17-32 | Alternates 0, 40 |
| 33-48 | Alternates 40, $C0 |
| 49-99 | $C0 |

**`TFUFRQ` (Fuse Frequency):**

| Waves | Value |
| :--- | :--- |
| 1-16 | 220 |
| 17-39 | 192 |
| 40-64 | 192 + (wave - 40) |
| 65-99 | 230 |

### Bonus Point Thresholds (`BONPTM`)

| Level | Points |
| :--- | :--- |
| 0 | 0 |
| 1 | 60 |
| 2 | 160 |
| 3 | 320 |
| 4 | 540 |
| 5 | 740 |
| 6 | 940 |
| 7 | 1140 |
| 8 | 1340 |

### Skill Level Selection Table (`LEVEL`)

Maps skill level selection index to starting wave number:

```
0, 2, 4, 6, 8, 10, 12, 14, 16, 19, 33, 23, 37, 27, 48, 50,
35, 39, 43, 70, 72, 81, 85, 89, 98, 100, 114, 128
```

End marker: `$FF`

--- 