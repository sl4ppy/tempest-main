# Hardware Registers & Memory Map

Complete reference for the Tempest arcade hardware I/O registers, extracted from `ALCOMN.MAC`, `ALHAR2.MAC`, `ALSOUN.MAC`, and `VGMC.MAC`.

**Base address:** `HARDWA = $6000`

---

## 1. Memory Map Overview

| Address Range | Size | Description |
|---------------|------|-------------|
| `$0000-$07FF` | 2 KB | Game RAM |
| `$0800` | 1 | Color/Intensity Port (`COLPORT`) |
| `$0C00` | 1 | Input Port 1 (`IN1`) |
| `$0D00` | 1 | DIP Switch Bank 0 (`INOP0`) |
| `$0E00` | 1 | DIP Switch Bank 1 (`INOP1`) |
| `$2000-$27FF` | 2 KB | Vector RAM (`VECRAM`) — display list buffer |
| `$3000-$3FFF` | 4 KB | Vector ROM |
| `$4000` | 1 | Output Port 0 (`OUT0`) |
| `$4800` | 1 | Vector Generator Start (`VGSTART`) |
| `$5000` | 1 | Watchdog / Interrupt Acknowledge (`WTCHDG`/`INTACK`) |
| `$5800` | 1 | Vector Generator Stop/Reset (`VGSTOP`) |
| `$6000-$6010` | | EAROM registers |
| `$6040` | 1 | Mathbox Status (`MSTAT`) |
| `$6050` | 1 | EAROM Read (`EAIN`) |
| `$6060` | 1 | Mathbox Result Low (`MYLOW`) |
| `$6070` | 1 | Mathbox Result High (`MYHIGH`) |
| `$6080-$6098` | 25 | Mathbox Parameter Registers (`MBSTAR`+) |
| `$60C0-$60CF` | 16 | POKEY Chip 1 |
| `$60D0-$60DF` | 16 | POKEY Chip 2 |
| `$60E0` | 1 | Output Port 1 (`OUTANK`) |
| `$9000-$FFFF` | 28 KB | Program ROM |

---

## 2. Input Ports

### IN1 ($0C00) - Read

| Bit | Mask | Name | Description |
|-----|------|------|-------------|
| 0 | `$01` | `MCOINR` | Right Coin Mechanism |
| 1 | `$02` | `MCOINC` | Center Coin Mechanism |
| 2 | `$04` | `MCOINL` | Left Coin Mechanism |
| 3 | `$08` | `SLAM` | Slam Switch |
| 4 | `$10` | `MTEST` | Self-Test Switch |
| 5 | `$20` | `MDITES` | Diagnostic Test Switch |
| 6 | `$40` | `MHALT` | VG Halt Status (1 = VG idle) |
| 7 | `$80` | `M3KHTI` | 3 KHz Timer Signal |

### POKEY1 Input ($60C0 read)

| Bit | Mask | Description |
|-----|------|-------------|
| 0-3 | `$0F` | Potentiometer Read (spinner ADC) |
| 4 | `$10` | `COCKTA` - Cocktail Mode Flag |
| 5 | `$20` | `MOPTI4` - Special Option 4 |

### POKEY2 Input ($60D0 read)

| Bit | Mask | Description |
|-----|------|-------------|
| 0-2 | `$07` | `MOPT13` - Special Options |
| 3 | `$08` | `MSUZA` - Superzapper Button |
| 4 | `$10` | `MFIRE` - Fire Button |
| 5 | `$20` | `MSTRT1` - Player 1 Start |
| 6 | `$40` | `MSTRT2` - Player 2 Start |
| 7 | `$80` | `MFAKE` - Fake Input (test) |

### DIP Switches

- `INOP0` (`$0D00`): DIP Switch Bank 0
- `INOP1` (`$0E00`): DIP Switch Bank 1

---

## 3. Output Ports

### OUT0 ($4000) - Write

| Bit | Mask | Name | Description |
|-----|------|------|-------------|
| 0 | `$01` | `MRCCNT` | Right Coin Counter Pulse |
| 1 | `$02` | `MMCCNT` | Center Coin Counter Pulse |
| 2 | `$04` | `MLCCNT` | Left Coin Counter Pulse |
| 3 | `$08` | `MVINVX` | Video Invert X (horizontal flip) |
| 4 | `$10` | `MVINVY` | Video Invert Y (vertical flip) |

### OUTANK ($60E0) - Write

| Bit | Mask | Name | Description |
|-----|------|------|-------------|
| 0 | `$01` | `MLED2` | Player 2 Start LED |
| 1 | `$02` | `MLED1` | Player 1 Start LED |
| 2 | `$04` | `MFLIP` | Screen Flip (cocktail mode) |

---

## 4. POKEY Sound Chip Registers

Two identical POKEY chips at base addresses `$60C0` (Chip 1) and `$60D0` (Chip 2).

### Register Map (offset from chip base)

| Offset | Write Name | Read Name | Description |
|--------|-----------|-----------|-------------|
| `+$00` | `AUDF1` | — | Audio Frequency Channel 1 |
| `+$01` | `AUDC1` | — | Audio Control Channel 1 (volume/distortion) |
| `+$02` | `AUDF2` | — | Audio Frequency Channel 2 |
| `+$03` | `AUDC2` | — | Audio Control Channel 2 |
| `+$04` | `AUDF3` | — | Audio Frequency Channel 3 |
| `+$05` | `AUDC3` | — | Audio Control Channel 3 |
| `+$06` | `AUDF4` | — | Audio Frequency Channel 4 |
| `+$07` | `AUDC4` | — | Audio Control Channel 4 |
| `+$08` | `AUDCTL` | `ALLPOT` | Audio Master Control / Read All Pots |
| `+$0A` | — | `RANDOM` | Random Number Generator (read) |
| `+$0B` | `POTGO` | — | Start Potentiometer Scan (write strobe) |
| `+$0F` | `SKCTL` | — | Serial/Keyboard Control |

### AUDCTL Bit Definitions

| Bit | Description |
|-----|-------------|
| 0 | Poly4/Poly9 select for channel 4 |
| 1 | Poly4/Poly9 select for channel 2 |
| 2 | Channel 1 uses 1.79 MHz clock |
| 3 | Channel 3 uses 1.79 MHz clock |
| 4 | Channel 1+2 clocked together (16-bit) |
| 5 | Channel 3+4 clocked together (16-bit) |
| 6 | High-pass filter channel 1 by channel 3 |
| 7 | High-pass filter channel 2 by channel 4 |

Default init value: `$00` (`AUDCV`, `AUDCV2`)

### Logical-to-Physical Channel Mapping

The sound engine manages 10 logical channels (indices 0-9). The `MODSND` routine maps them:

```
Channels 0-7: Write to POKEY1 (AUDF1 + channel_index)
  Channel 0 -> POKEY1 AUDF1  ($60C0)
  Channel 1 -> POKEY1 AUDC1  ($60C1)
  Channel 2 -> POKEY1 AUDF2  ($60C2)
  Channel 3 -> POKEY1 AUDC2  ($60C3)
  Channel 4 -> POKEY1 AUDF3  ($60C4)
  Channel 5 -> POKEY1 AUDC3  ($60C5)
  Channel 6 -> POKEY1 AUDF4  ($60C6)
  Channel 7 -> POKEY1 AUDC4  ($60C7)

Channels 8-9: Write to POKEY2 (AUDF2 + (channel_index - 8))
  Channel 8 -> POKEY2 AUDF1  ($60D0)
  Channel 9 -> POKEY2 AUDC1  ($60D1)
```

Channels are written in pairs: even = frequency register, odd = control register. Each sound effect activates up to 6 channels via the `PNTRS` master table.

### Sound Data Format

Each sound sequence is 4 bytes:
| Byte | Name | Description |
|------|------|-------------|
| 0 | `STVAL` | Starting value written to POKEY register |
| 1 | `FRCNT` | Frames between each change |
| 2 | `CHANGE` | Signed delta added per update (wraps) |
| 3 | `NUMBER` | Total number of changes (`$FF` = infinite loop) |

### Sound Effect IDs

| ID | Constant | Description |
|----|----------|-------------|
| 0 | `SIDLO` | Cursor move |
| 1 | `SIDEX` | Enemy explosion |
| 2 | `SIDLA` | Player fire |
| 3 | `SIDPU` | Pulsation |
| 4 | `SIDWP` | Special score |
| 5 | `SIDDI` | Player dies |
| 6 | `SIDT2` | Thrust in tube |
| 7 | `SIDT3` | Thrust in space |
| 8 | `SIDES` | Enemy shot |
| 9 | `SIDEL` | Enemy line destruction |
| 10 | `SIDSL` | Slam |
| 11 | `SIDS3` | 3-second warning |
| 12 | `SIDPO` | Pulsar off |

---

## 5. Mathbox Registers

### Parameter Write Registers (base `MBSTAR = $6080`)

| Address | Offset | Name | Description |
|---------|--------|------|-------------|
| `$6080` | +0 | `MAL` | Multiplier A Low |
| `$6081` | +1 | `MAH` | Multiplier A High |
| `$6082` | +2 | `MBL` | Multiplier B Low |
| `$6083` | +3 | `MBH` | Multiplier B High |
| `$6084` | +4 | `MEL` | Operand E Low |
| `$6085` | +5 | `MEH` | Operand E High |
| `$6086` | +6 | `MFL` | Operand F Low |
| `$6087` | +7 | `MFH` | Operand F High |
| `$6088` | +8 | `MXL` | Operand X Low |
| `$6089` | +9 | `MXH` | Operand X High |
| `$608C` | +C | `MNL` | Operand N Low |
| `$608D` | +D | `MZLL` | Z Low Low |
| `$608E` | +E | `MZLH` | Z Low High |
| `$608F` | +F | `MZHL` | Z High Low |
| `$6090` | +10 | `MZHH` | Z High High |
| `$6092` | +12 | `SYM` | Symbol / Operation Trigger |
| `$6094` | +14 | `MSZXD` | Size X Delta |
| `$6095` | +15 | `MXPL` | X Position Low |
| `$6096` | +16 | `MXPH` | X Position High |
| `$6098` | +18 | `MDYPL` | Delta Y Position Low |

### Result Read Registers

| Address | Name | Description |
|---------|------|-------------|
| `$6060` | `MYLOW` | Result Low Byte |
| `$6070` | `MYHIGH` | Result High Byte |
| `$6040` | `MSTAT` | Status (bit 7 = 1 when operation complete) |

### WORSCR Protocol (3D-to-2D Projection)

The `WORSCR` routine in `ALDISP.MAC` projects a world-space point to screen coordinates:

**Inputs:** `PXL`, `PYL`, `PZL` (world point), `EXL`, `EYL` (eye/camera position)

**Sequence:**
1. Calculate Y delta: `DeltaY = PYL - EYL`
2. Look up `INVERSE[DeltaY]` and `INVEXP[DeltaY]` tables for reciprocal/exponent
3. Write `MBL = INVERSE[DeltaY]` (multiplier)
4. Calculate `DeltaX = ABS(PXL - EXL)`, note sign
5. Write `MXL = DeltaX` (multiplicand)
6. Write `SYM = 0` (trigger multiply)
7. Poll `MSTAT` bit 7 until set (operation complete)
8. Read `MYLOW` -> `SXL`, `MYHIGH` -> `SXH` (screen X)
9. Apply exponent shift from `INVEXP`
10. Repeat steps 4-9 for Z axis to get screen Y
11. Add `ZADJL` vanishing point adjustment to screen Y

**Formula:** `Screen_X = (PXL - EXL) * INVERSE[PYL - EYL]`, shifted by `INVEXP` exponent

---

## 6. EAROM (Electrically Alterable ROM)

| Address | Name | Description |
|---------|------|-------------|
| `$6000` | `EADAL` | Write: set data address |
| `$6040` | `EACTL` | Control register |
| `$6050` | `EAIN` | Read: data out |

### EACTL Bit Definitions

| Bit | Name | Description |
|-----|------|-------------|
| 0 | `EACK` | Clock |
| 1 | `EAC2` | C2 control |
| 2 | `EAC1` | C1 control (inverted to chip) |
| 3 | `EACE` | Chip enable |

### Operation Modes

| Mode | C1 | C2 | Description |
|------|----|----|-------------|
| Read | 0 | 0 | Read data at address |
| Write | 1 | 0 | Write data at address |
| Erase | 1 | 1 | Erase data at address |

### EAROM Data Layout

| Offset | Size | Description |
|--------|------|-------------|
| `$00-$08` | 9 bytes | High score initials |
| `$09-$13` | 11 bytes | High scores and options |
| `$14+` | variable | Bookkeeping counters |

---

## 7. Vector Generator Instruction Encoding

All VG instructions are 16-bit words stored in `VECRAM` (`$2000-$27FF`). The VG hardware reads and executes them autonomously.

### Instruction Set

#### VCTR - Draw Vector (Long Format, 32-bit)

Two consecutive 16-bit words:
```
Word 1: [DY_sign:1][DY_abs:12][unused:3]
Word 2: [0:1][0:1][0:1][0:1][Z:3][DX_sign:1][DX_abs:12]
```
- `DY`: 13-bit signed delta Y
- `DX`: 13-bit signed delta X
- `Z`: 3-bit intensity (0 = beam off/move only, 7 = brightest)

#### SVEC - Short Vector (16-bit)

```
[1:1][DY_sign:1][DY:2][Z:3][0:1][DX_sign:1][DX:2][0:3]
```
- `DY`, `DX`: 2-bit magnitude (scaled by current scale factor)
- `Z`: 3-bit intensity

#### SCAL - Set Scale

```
[0:1][1:1][1:1][1:1][BSCALE:3][LINEAR:8][0:1]
```
- Opcode: `$7xxx`
- `BSCALE`: 3-bit binary scale (0=full, 7=1/128)
- `LINEAR`: 8-bit fine scale (0=full, $FF=1/256)

#### CNTR - Center Beam

```
$8040
```
Moves beam to screen center (0,0) without drawing.

#### HALT - Stop VG

```
$2000
```
Stops the Vector Generator. Sets `MHALT` bit in `IN1`.

#### JSRL - Jump to Subroutine

```
[1:1][0:1][1:1][0:1][ADDRESS:12]
```
- Opcode: `$Axxx`
- `ADDRESS`: 12-bit word address (byte address / 2)
- Pushes return address onto 5-level hardware stack

#### JMPL - Jump (Unconditional)

```
[1:1][1:1][1:1][0:1][ADDRESS:12]
```
- Opcode: `$Exxx`
- `ADDRESS`: 12-bit word address

#### RTSL - Return from Subroutine

```
[1:1][1:1][0:1][0:1][unused:12]
```
- Opcode: `$Cxxx`
- Pops from 5-level hardware stack

#### STAT - Set Status/Color

```
[0:1][1:1][1:1][0:1][Z:4][HI:1][IN:1][MODE:2][unused:4]
```
- Opcode: `$6xxx`
- `Z`: 4-bit intensity
- `HI`: corner select
- `IN`: inside/outside clipping

### VG State Machine (from STATE2.MAC)

| Code | Instruction |
|------|-------------|
| `000` | VCTR |
| `001` | HALT |
| `010` | SVEC |
| `011` | STAT |
| `100` | (wait) |
| `101` | JSR |
| `110` | RTS |
| `111` | JMP |

### Addressing

- VG addresses are 12-bit word addresses (covering 8 KB byte-addressable space)
- `VECRAM` at `$2000` and Vector ROM at `$3000` are both in the VG address space
- `JSRL LABEL` encodes as: `$A000 + (LABEL & $1FFF) / 2`
- `JMPL LABEL` encodes as: `$E000 + (LABEL & $1FFF) / 2`
