# Display List Compiler (ALDIS2.MAC)

This document describes the display list compiler — the system that builds the Vector Generator display list each frame. Source: `ALDIS2.MAC`.

---

## Entry Point and Frame Flow

The `DISPLAY` routine is called once per frame from the main loop:

1. `JSR INIMAT` — initialize Mathbox registers for 3D projection
2. Check game display state (`QDSTATE`) to determine what to draw
3. For gameplay: jump to `DENORM` which draws all game objects

---

## Double-Buffering

The compiler uses a double-buffer scheme so the VG hardware can execute one buffer while the CPU builds the next.

### Buffer Management

- **`BUFACT[n]`**: Per-group flag tracking which buffer (A or B) is active
- **`BUFASL/BUFASH`**: Start address (lo/hi) for Buffer A
- **`BUFBSL/BUFBSH`**: Start address (lo/hi) for Buffer B
- **`BUFSWL/BUFSWH`**: Address of the JMPL switch instruction in VECRAM
- **`VGLIST`**: 2-byte pointer to current write position in the inactive buffer
- **`VGY`**: Y-register index into VGLIST (reset to 0 on each buffer setup)

### SBCLOG — Select Buffer for Writing

```
Input: ACC = sub-buffer group index (0-15)
1. Check BUFACT[group] to find which buffer is active
2. If BUFACT=0: select Buffer B addresses (BUFBSL/BSH)
   If BUFACT≠0: select Buffer A addresses (BUFASL/ASH)
3. Store address in VGLIST, set VGY=0
```

### SBCSWI — Switch Active Buffer

```
Input: ACC = sub-buffer group index
1. Write RTSL instruction at end of newly built buffer
2. Toggle BUFACT[group] (EOR with $01)
3. Load JMPL target address from new buffer's JMPALO/JMPBLO table
4. Write the new JMPL instruction to the switch location (BUFSWL/H)
```

The VG hardware follows JMPL chains through VECRAM. By updating a single JMPL instruction, the compiler atomically switches which buffer the hardware reads.

---

## Drawing Order (DENORM routine)

Objects are drawn in this order (bottom to top in display priority):

| Order | Routine | Buffer Group | Content |
|-------|---------|-------------|---------|
| 1 | `DSPCUR` | `BCCURS` | Player cursor |
| 2 | `DSPCHG` | `BCSHOT` | Player and enemy shots |
| 3 | `DSPINV` | `BCINVA` | Invaders (enemies) |
| 4 | `DSPEXP` | `BCEXPL` | Explosions |
| 5 | `DSPNYM` | `BCNYMP` | Nymphs (spiker particles) |
| 6 | INFO | `BCINFO` | Score, lives, UI text |
| 7 | `DSPWEL` | `BCWELL` | Well structure |
| 8 | `DSPENL` | `BCENEL` | Enemy lines (spikes) |
| 9 | `DSTARF` | `BCSTAR` | Star field background |

After all objects are drawn, the master JMPL at the start of VECRAM is updated to point to the new frame.

---

## Invader Display Dispatch

Each invader type has its own rendering routine, dispatched via the `INVPIC` table:

| Type Index | Routine | Entity |
|------------|---------|--------|
| 0 | `FLIPIC` | Flipper |
| 1 | `PULPIC` | Pulsar |
| 2 | `TANPIC` | Tanker |
| 3 | `TRAPIC` | Trailer (Spiker) |
| 4 | `FUSPIC` | Fuseball |

---

## 3D-to-2D Projection (WORSCR)

Each entity's world-space position is projected to screen coordinates using the Mathbox. See [HARDWARE_REGISTERS.md](./HARDWARE_REGISTERS.md#worscr-protocol-3d-to-2d-projection) for the full register write sequence.

The projection uses `INVERSE` and `INVEXP` lookup tables indexed by the Y-depth delta (`PYL - EYL`) rather than traditional sine/cosine tables. This performs perspective division:

```
Screen_X = (World_X - Eye_X) * INVERSE[World_Y - Eye_Y]
Screen_Y = (World_Z - Eye_Z) * INVERSE[World_Y - Eye_Y] + ZADJL
```

The `INVEXP` table provides the bit-shift exponent to normalize the result.

---

## VGLIST Overflow Protection

The maximum `VGY` index before overflow is `$F0` (240 bytes). If the buffer fills during drawing, remaining objects are skipped for that frame.
