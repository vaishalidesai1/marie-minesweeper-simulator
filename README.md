# Minesweeper in MARIE.js — Overall Summary

## Report

Overleaf project (full write-up and documentation):  
https://www.overleaf.com/6586667898kvfbzpdncpzv#6dea7d

## Architecture

Two parallel flat arrays in memory, both indexed by:

```text
Idx = fila * 16 + col
```

- `tablero` at `0x400` — logical ground truth  
  - `-1` = mine  
  - `0–8` = adjacent mine counts  
  - `+100` = revealed variants  
  - `+200` = flagged variants

- `display` at `0xF00` — visual state  
  - `Gris` = hidden  
  - `Amarillo` = flagged  
  - `GrisC` = revealed blank candidate (used in flood fill)  
  - `C1–C8` = numbered colors  
  - `Negro` = mine revealed

Both arrays are preceded by padding to force alignment since MARIE has no `ORG` directive.  
The board itself is precomputed and baked as `DEC` constants.

---

## Game Flow

### Initialization

- `display` filled with `Gris` using a 256-iteration do-while from `DispBase`
- `CntSeg` initialized to 216:

```text
256 cells − 40 mines = 216 safe cells
```

---

## Each Turn (`Turno`)

1. Print prompts and read:
   - `FilaIn`
   - `ColIn`
   - `Accion`

2. Convert user input from 1-indexed to 0-indexed:

```text
Subt Uno
```

3. Compute:

```text
Idx = FilaIn * 16 + ColIn
```

(using 16 unrolled additions)

4. Compute:

- `TablPtr`
- `DispPtr`

5. Read:

```text
ValCelda = tablero[Idx]
```

6. Dispatch based on visual state:

| display[Idx] | Path |
|-------------|------|
| Gris | `ChkAccion` |
| Amarillo | `ChkBandera` / `ChkTogBand` |
| anything else | ignore move, return to `Turno` |

---

## Actions

### Reveal (`Accion = 1`)

- Check for mine  
- Dispatch to color routine  
- Mark revealed:

```text
tablero += 100
CntSeg--
```

- If:

```text
ValCelda == 0
```

start flood fill.

---

### Flag (`Accion = 2`)

Toggle:

```text
tablero += 200 or -= 200
display: Amarillo <-> Gris
```

---

### Reveal on Flagged Cell

Blocked.

Move is not counted.

---

## Flood Fill

Starts from:

```text
FloodOrig
```

### Phase 1 — `SubirLoop`

Walk upward through the column:

```text
-16 per step
```

Reveal until:

- boundary
- mine
- numbered cell

---

### Phase 2 — `BajarLoop`

Walk downward:

```text
+16 per step
```

Same stopping conditions.

---

### Phase 3 — Wave Propagation (`FFWLOOP`)

Column-major scan of entire board.

Every `GrisC` cell propagates to all 8 neighbors via:

```text
FFRVNGH
```

Repeat until:

```text
FFChg == 0
```

(no new changes in a full pass)

---

Each revealed cell does:

```text
PintaColor
tablero += 100
CntSeg--
```

---

## Terminal Conditions

| Condition | Path |
|----------|------|
| `ValCelda == -1` | Reveal all mines, print "Perdiste", `Halt` |
| `CntSeg == 0` | Print "Ganaste", output `Movimientos`, `Halt` |

---

## MARIE Design Constraints

| Constraint | Solution |
|-----------|----------|
| No multiply | `×16` done as 16 unrolled `Add FilaIn` instructions |
| No stack | `JnS` / `JumpI` with return address stored at subroutine label |
| No `ORG` directive | Explicit `DEC 0` padding aligns arrays to `0x400` and `0xF00` |
| No post-increment | Explicit pointer increment after every indirect access |
| No jump table | Linear subtraction chains for color dispatch (`0–8`) |
| AC is only register | Frequent `Store`/`Load` of intermediates (`ValCelda`, `WVal`, `ValTemp`) |

---

## Summary

This implementation models full Minesweeper behavior in pure MARIE assembly:

- Precomputed 16×16 board with 40 mines  
- Reveal and flag actions  
- Three-phase flood fill for zero regions  
- Win/loss detection  
- Separate logical and visual memory models  
- All implemented under MARIE’s single-accumulator architecture constraints