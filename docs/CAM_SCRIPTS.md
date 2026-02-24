# CAM Scripts: Enemy Behavior Instruction Set

The CAM (Computer-Aided Movement) system is the bytecode interpreter that drives all enemy movement behavior in Tempest. Each active invader has a CAM program counter (`INVCAM`) that steps through a script each frame. The dispatcher reads one opcode at a time, executes it via an indirect JSR through the `TABJSR` jump table, and continues until a `VEXIT` instruction sets the exit flag.

**Source:** `ALWELG.MAC`, lines 1505-1630 (dispatcher), lines 2367-2520 (scripts)

---

## Execution Model

Each frame, `MOVINV` iterates over all active invaders (0 to `WINVMX-1`):

1. Set `EXICAM = 1` (continue flag)
2. Load invader's saved `CAMPC` (CAM program counter)
3. Loop: read opcode byte from `CAM[CAMPC]`, dispatch via `JSRCAM`/`TABJSR`, increment `CAMPC`
4. Loop exits when `EXICAM = 0` (set by `VEXIT`)
5. Save updated `CAMPC` back to invader

CAM instructions execute within a single frame. Multiple instructions can execute per frame until `VEXIT` yields control.

---

## Instruction Set Reference

All opcodes are even-numbered (used as 2-byte index into `TABJSR`).

### Control Flow

| Byte | Mnemonic | Operands | Description |
|------|----------|----------|-------------|
| `$00` | `VEXIT` | none | Yield: set `EXICAM=0`, return control to main loop |
| `$02` | `VSLOOP` | imm8 | Set loop counter (`INVLOO`) to immediate byte value |
| `$08` | `VELOOP` | addr8 | Decrement loop counter; if non-zero, branch to `CAM + addr8 + 1` |
| `$06` | `VSETPC` | addr8 | Unconditional branch: set `CAMPC = addr8` (relative to CAM base) |
| `$0A` | `VNOOP` | none | No operation |
| `$04` | `VSKIP0` | none | If `CAMSTA = 0`, skip next 2 CAM bytes |
| `$1A` | `VBR0PC` | addr8 | If `CAMSTA = 0`, branch to `CAM + addr8 + 1` |
| `$10` | `VSLOPB` | addr16 | Set loop counter from memory address (base-page pointer) |

### Movement

| Byte | Mnemonic | Operands | Description |
|------|----------|----------|-------------|
| `$0C` | `VSMOVE` | none | Move invader up/down by one frame (using `WINVIL`/`WINVIN` speed) |
| `$0E` | `VSTRAI` | none | Process trailer logic (spiker special movement) |
| `$12` | `VJUMPS` | none | Start a lane-change jump sequence |
| `$14` | `VJUMPM` | none | Continue jump motion; sets `CAMSTA=0` when jump completes |
| `$16` | `VCHROT` | none | Reverse jump direction (toggle `INVROT` flag) |
| `$24` | `VCHPLA` | none | Set jump direction toward player's current lane |
| `$1E` | `VSFUSE` | none | Process fuseball up/down motion |
| `$22` | `VSPUMO` | none | Pulsar movement (1/8 tube per step before flip) |

### Testing / Conditionals

| Byte | Mnemonic | Operands | Description |
|------|----------|----------|-------------|
| `$18` | `VKITST` | none | Test if both invader legs are on the same lane as cursor (kill check) |
| `$1C` | `VELTST` | none | Test if invader is on an "enemy line"; sets `CAMSTA` accordingly |
| `$20` | `VFUSKI` | none | Fuseball cursor-kill check |
| `$26` | `VCHKPU` | none | Check if pulsar is pulsing within next 4 frames |

### Instruction Formats

- **Single-byte** (most instructions): opcode only
- **Two-byte immediate** (`VSLOOP`): opcode + parameter byte
- **Two-byte address** (`VSETPC`, `VELOOP`, `VBR0PC`): opcode + `(target_label - CAM_base - 1)`
- **Two-byte pointer** (`VSLOPB`): opcode + base-page address

---

## CAM State Variables

| Variable | Size | Description |
|----------|------|-------------|
| `INVCAM` | 1 byte/invader | Saved CAM program counter |
| `CAMPC` | 1 byte | Current CAM PC during execution |
| `CAMSTA` | 1 byte | Status flag for conditional instructions |
| `EXICAM` | 1 byte | Exit flag (0=yield, 1=continue) |
| `INVLOO` | 1 byte/invader | Loop counter for `VSLOOP`/`VELOOP` |
| `INVROT` | 1 byte/invader | Jump direction flag |

---

## Script Definitions

### NOJUMP - Straight Up, No Lane Changes
Used by: Tankers, early-wave Flippers
```
NOJUMP:
    VSMOVE          ; Move up one frame
    VEXIT           ; Yield
    VSETPC NOJUMP   ; Loop
```

### TRALUP - Trailer (Spiker) Moving Up
```
TRALUP:
    VSMOVE          ; Move up
    VSTRAI          ; Process trailer special logic
    VBR0PC NOJUMP   ; If condition met, convert to carrier behavior
    VEXIT           ; Yield
    VSETPC TRALUP   ; Loop
```

### MOVJMP - Move N Frames Then Jump
Used by: Square and Triangle wells
```
MOVJMP:
    VSLOOP 8        ; Loop 8 times
MJLOP1:
    VSMOVE          ; Move up
    VEXIT           ; Yield
    VELOOP MJLOP1   ; Repeat 8 times
    VJUMPS          ; Start lane-change jump
MJLOP5:
    VEXIT           ; Yield
    VJUMPM          ; Continue jump motion
    VSKIP0          ; Skip next 2 if jump done (CAMSTA=0)
    VSETPC MJLOP5   ; Jump not done, keep going
    VSETPC MOVJMP   ; Jump done, restart pattern
```

### SPIRAL - Smooth Upward Spiral
Used by: Cross and Heart wells
```
SPIRAL:
    VSMOVE          ; Move up
    VEXIT           ; Yield
    VJUMPS          ; Start jump
SPILOP:
    VEXIT           ; Yield
    VJUMPM          ; Continue jump
    VSMOVE          ; Move up while jumping
    VSKIP0          ; Skip if jump done
    VSETPC SPILOP   ; Continue jump
    VSETPC SPIRAL   ; Restart
```

### SPIRCH - Spiral With Direction Changes
Used by: Peanut, Clover, 8-shape wells
```
SPIRCH:
    VSMOVE          ; Move up
    VEXIT           ; Yield
    VSLOOP 2        ; Do 2 jumps
SPRLP1:
    VJUMPS          ; Start jump
SPRLP2:
    VEXIT
    VJUMPM          ; Continue jump
    VSMOVE          ; Move while jumping
    VSKIP0          ; Skip if done
    VSETPC SPRLP2   ; Continue
    VEXIT
    VELOOP SPRLP1   ; Next jump
    VCHROT          ; Reverse direction
    VSLOOP 3        ; Do 3 jumps
SPRLP3:
    VJUMPS
SPRLP4:
    VEXIT
    VJUMPM
    VSMOVE
    VSKIP0
    VSETPC SPRLP4
    VEXIT
    VELOOP SPRLP3
    VCHROT          ; Reverse again
    VSETPC SPIRCH   ; Restart pattern
```

### TOPPER - Chase Player Around Rim
Used by: Flippers that reach the top
```
TOPPER:
    VSLOOP 4        ; Crouch for 4 frames
KICHEK:
    VKITST          ; Check for cursor kill
    VEXIT           ; Yield
    VELOOP KICHEK   ; Wait 4 frames
    VJUMPS          ; Start jump toward player
KJULP1:
    VEXIT           ; Yield
    VSLOPB WTTFRA   ; Load double-speed flip rate
KJULP2:
    VJUMPM          ; Process jump at double speed
    VBR0PC TOPPER   ; Jump done? Branch to restart
    VELOOP KJULP2   ; Continue jump
    VSETPC KJULP1   ; Keep going
```

### COWJMP / COWJM2 - Flip Only on Open Lines
Used by: Key, Stairs, Star wells
```
COWJM2:
    VEXIT           ; Yield
COWJMP:
    VSMOVE          ; Move up
    VELTST          ; On enemy line?
    VBR0PC COWJM2   ; Yes: stay on this line, continue up
    VJUMPS          ; No: start jump to adjacent lane
    VEXIT
    VSMOVE          ; Move up during jump
COWJM3:
    VJUMPM          ; Continue jump
    VBR0PC COWJM2   ; Jump done?
    VEXIT
    VSETPC COWJM3   ; Keep jumping
```

### FUSEUP - Fuseball Vertical Motion
```
FUSEUP:
    VSFUSE          ; Process fuse up/down
    VFUSKI          ; Check cursor kill
    VEXIT           ; Yield
    VSETPC FUSEUP   ; Loop
```

### FUSELR - Fuseball Lateral Motion
```
FUSELR:
    VEXIT           ; Yield
    VSLOOP 3        ; Slow: 3 frames between lateral moves
FUSLOP:
    VFUSKI          ; Check kill
    VEXIT
    VELOOP FUSLOP
    VJUMPM          ; Move laterally
    VBR0PC FUSEUP   ; Jump done? Switch to vertical
    VSETPC FUSELR   ; Continue lateral
```

### PULSCH - Pulsar Chaser (Chases Player)
```
PULSCP:
    VSLOPB PUCHDE   ; Load chase delay from PUCHDE
PULSC1:
    VSPUMO          ; Move 1/8 tube
    VEXIT
    VELOOP PULSC1   ; Wait chase delay frames
PULSC2:
    VCHKPU          ; Currently pulsing?
    VBR0PC PULSC3   ; No: go flip
    VSPUMO          ; Yes: keep moving during pulse
    VEXIT
    VSETPC PULSC2   ; Recheck
PULSC3:
    VCHPLA          ; Set flip direction toward player
    VJUMPS          ; Start flip
PULSCJ:
    VEXIT
    VJUMPM          ; Continue flip
    VBR0PC PULSCP   ; Done? Restart
    VSETPC PULSCJ   ; Keep flipping
```

### AVOIDR - Avoidance Flipper (Flips Away From Player)
Used by: U-shape and Topo wells (late game)
```
AVOIDR:
    VCHPLA          ; Set direction toward player
    VCHROT          ; Reverse it (flee)
    VJUMPS          ; Start flip away
AVOID1:
    VEXIT
    VSMOVE          ; Move up during flip
    VJUMPM          ; Continue flip
    VSKIP0          ; Skip if done
    VSETPC AVOID1   ; Keep flipping
    VSLOOP 4        ; Post-flip: move straight for 4 frames
AVOID2:
    VEXIT
    VSMOVE          ; Move up
    VELOOP AVOID2   ; Wait 4 frames
    VSETPC AVOIDR   ; Restart avoidance
```

---

## CAM Assignment by Wave

The `CAMWAV` table uses `TZANDF` encoding (wave number AND `$0F`) to cycle through 16 well shapes:

| Wave mod 16 | Well Shape | Flipper CAM Script |
|-------------|------------|-------------------|
| 0 | Circle | `NOJUMP` |
| 1 | Square | `MOVJMP` |
| 2 | Cross | `SPIRAL` |
| 3 | Peanut | `SPIRCH` |
| 4 | Key | `COWJM2` |
| 5 | Triangle | `MOVJMP` |
| 6 | Clover | `SPIRCH` |
| 7 | V-shape | `SPIRAL` |
| 8 | Stairs | `COWJM2` |
| 9 | U-shape | `AVOIDR` |
| 10 | Flat | `SPIRCH` |
| 11 | Heart | `SPIRAL` |
| 12 | Star | `COWJM2` |
| 13 | Waves | `NOJUMP` |
| 14 | Topo | `AVOIDR` |
| 15 | 8-shape | `SPIRCH` |

## Default CAM by Enemy Type

From `TNEWCAM` (new enemy initialization):

| Index | Enemy Type | Default CAM |
|-------|-----------|-------------|
| 0 | Flipper | `NOJUMP` (overridden by `CAMWAV` per wave) |
| 1 | Pulsar | `PULSCH` |
| 2 | Tanker | `NOJUMP` |
| 3 | Trailer (Spiker) | `TRALUP` |
| 4 | Fuseball | `FUSEUP` |
