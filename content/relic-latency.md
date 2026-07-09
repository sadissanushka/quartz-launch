# `src/lib/relic/latency.ts` — Physics Latency Engine

**Tags:** `#physics` `#latency` `#phase-1`  
**See also:** [[relic-router]] · [[relic-contracts]] · [[relic-types]]

---

## Purpose

Implements the **two physical latency primitives** of the Relic Ring Protocol. Every constant is read from `UniverseMetadata`; geometry is delegated to the `GeometryProvider`. This module is **pure computation** — no I/O, no state.

---

## The Two Formulas

### 1. Void Travel Time — `Tv`

Time for a laser packet to cross the void between two planets:

```
Tv = ((h1 × n1) + (h2 × n2) + L) / C
   = atmosphere_ms + void_ms
```

- `h1/h2` — atmosphere thickness of each planet (km)
- `n1/n2` — refraction index of each planet
- `L` — void distance (km), center-based (no tower offset)
- `C` — speed of light (km/s), from `universe_metadata`

**Function:** `computeVoidLatency(origin, destination, geometry, metadata) → VoidLatency`

### 2. Internal Crust Transit — `Tp`

Time spent routing through a planet's equatorial tower ring:

```
Tp = (2π × r × s) / (N × f × C) + m × Δt
   = fiber_ms + tower_ms
```

- `r` — planet radius (km)
- `s` — ring segments between entry and exit tower
- `N` — number of active towers
- `f` — fiber speed fraction (fraction of C)
- `m = s + 1` — distinct towers hit (always at least 1 even when entry = exit)
- `Δt` — tower processing delay (ms), from metadata

**Function:** `computeInternalLatency(planet, entryTower, exitTower, geometry, metadata) → InternalLatency`

---

## Result Types

### `VoidLatency`

```ts
{
  (void_distance_km, atmosphere_ms, void_ms, total_ms);
}
```

### `InternalLatency`

```ts
{
  (segments, towers_hit, arc_length_km, fiber_ms, tower_ms, total_ms);
}
```

### `combineRouteLatency(internal[], voids[]) → LatencyBreakdown`

Aggregates all per-planet internals and per-hop void values into the route-level 4-component breakdown.

---

## Edge Cases

| Scenario                 | Behavior                                          |
| ------------------------ | ------------------------------------------------- |
| Origin = Destination     | `s = 0`, `m = 1`, charges exactly one tower delay |
| Entry tower = Exit tower | `segmentsBetween` returns 0, fiber_ms = 0         |
| Invalid tower index      | `assertTowerInRange` throws `RangeError`          |

---

## Design Notes

- **All results in milliseconds** — consistent with `tower_processing_delay_ms` which is already in ms.
- `MS_PER_SECOND = 1000` constant avoids magic numbers in divisions.
- `computeVoidLatency` uses center-based L (not tower-to-tower). This is the Void Distance Simplification from the spec.

---

Back to [[00 - Index]]
