# `src/lib/relic/contracts.ts` — Integration Contracts

**Tags:** `#interfaces` `#dependency-injection` `#phase-1`  
**See also:** [[relic-engine]] · [[relic-latency]] · [[relic-router]]

---

## Purpose

Defines the **interface contracts** for the two teammate-owned modules: the **geometry/mapping module** (`GeometryProvider`) and the **encoding/decoding module** (`Codec`). The engine depends only on these interfaces — never on concrete teammate implementations.

> [!IMPORTANT]
> Swap the stubs in `engine.ts` for real implementations at merge time with **zero changes** to routing or latency code. This is the key extension point.

---

## `GeometryProvider`

Owner: **Mapping teammate**

| Method                                 | Description                                                  |
| -------------------------------------- | ------------------------------------------------------------ |
| `towerPositions(planet)`               | Absolute tower positions on the equatorial ring              |
| `centerDistanceKm(a, b)`               | Scaled center-to-center distance `S`                         |
| `voidDistanceKm(a, b)`                 | Laser void distance `L = S - (R1+h1) - (R2+h2)`              |
| `closestTowerPair(a, b)`               | Tower pair minimizing line-of-sight gap                      |
| `segmentsBetween(planet, entry, exit)` | Ring segments `s` between two towers (→ 0 when entry = exit) |

**Formula constraints documented in JSDoc:**

- `S = scale × √((x2-x1)² + (y2-y1)²)`
- `L = S - (R1+h1) - (R2+h2)`

### `TowerPosition`

`{ index, x_km, y_km, angle_deg }` — absolute world coordinates.

### `TowerPair`

`{ origin_tower, destination_tower, separation_km }` — used by the router to choose entry/exit towers.

---

## `Codec`

Owner: **Encoding/decoding teammate**

| Method                                | Description                                      |
| ------------------------------------- | ------------------------------------------------ |
| `toAscii(payload)`                    | String → ASCII byte array                        |
| `fromAscii(bytes)`                    | ASCII byte array → string                        |
| `encodeToCodex(ascii, base)`          | ASCII → codex digits (e.g. base 5: `72 → "242"`) |
| `decodeFromCodex(encoded)`            | Codex digits → ASCII                             |
| `serializeToBinary(encoded)`          | Codex digits → flat 8-bit binary stream          |
| `deserializeFromBinary(stream, base)` | Binary stream → codex digits                     |

### `EncodedPayload`

`{ base: number, digits: string[] }` — per-character digit strings.

---

## Design Notes

- **Void Distance Simplification**: `voidDistanceKm` is center-based. Tower angular position does NOT change `L`; only tower choice changes internal fiber routing.
- The production codec ([`RelicCodec`](relic-codec)) implements this interface; a stub codec lives in `stubs/codec.stub.ts` for development without the encoding teammate.

---

Back to [[00 - Index]]
