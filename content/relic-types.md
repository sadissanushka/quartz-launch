# `src/lib/relic/types.ts` — Core Domain Types

**Tags:** `#types` `#domain-model` `#phase-1` `#phase-2`  
**See also:** [[relic-contracts]] · [[relic-config]] · [[chimera-types]]

---

## Purpose

The single source of truth for **every domain type** in the project. Both Phase 1 (physics engine) and Phase 2 (Chimera Co-Pilot) types live here, since the Co-Pilot builds on top of the base universe model.

---

## Key Types

### `UniverseMetadata`

Physical constants for the Zeta-26 star system, read from `universe-config.json`.

| Field                       | Default    | Description                  |
| --------------------------- | ---------- | ---------------------------- |
| `speed_of_light_kms`        | 300,000    | `C` — speed of light in km/s |
| `max_void_hop_distance_km`  | 50,000,000 | `Lmax` — max single hop      |
| `coordinate_scale_unit_km`  | required   | converts grid units → km     |
| `tower_processing_delay_ms` | 7          | `Δt` per tower hit           |
| `fiber_speed_fraction`      | 0.67       | fraction of `C` for fiber    |

> [!NOTE]
> None of these values are hardcoded in the engine — they are always read from `universe_metadata`. This is a design constraint of the challenge.

### `PlanetNode`

Represents a single planet on the universe grid.  
Key fields: `id`, `codex` (numerical base), `x`/`y` (grid coordinates), `radius_km`, `active_towers`, `atmosphere_thickness_km`, `refraction_index`.

### `Universe`

The fully-parsed config: `metadata` + `nodes` + `nodesById` (fast `Map<id, PlanetNode>`) + `interplanetaryLinks` + `linksById`.

### `LatencyBreakdown`

4-component latency split: `fiber_ms`, `tower_ms`, `atmosphere_ms`, `void_ms`, `total_ms`. Built by [[relic-latency]].

### `HopLogEntry`

Per-planet proof-of-route record in a Packet. Documents every tower traversal, dialect encoding, binary stream, and cumulative latency. Built by [[relic-transmission]].

### Phase 2 Types

| Type                  | Description                                                                                                                             |
| --------------------- | --------------------------------------------------------------------------------------------------------------------------------------- |
| `InterplanetaryLink`  | Static link definition from config (`link_id`, `planet_a`, `planet_b`, `capacity_units`)                                                |
| `ChimeraLinkState`    | Live state from `GET /state` — includes `load_ratio`, `traffic_share`, `self_reported_latency_ms`, `status`                             |
| `LinkEvaluation`      | Scores for one link: `trust_score`, `targeting_risk_score`, `predicted_congestion_penalty_ms`, `combined_cost`                          |
| `Phase2RoutingReport` | Mandatory Council output — `origin_id`, `destination_id`, `chosen_path`, `link_evaluations`, `final_latency_estimate_ms`, `explanation` |
| `Packet`              | Full transmission record with `hop_log`, `status`, and optional `undeliverable_reason`                                                  |

---

## Design Notes

- `ChimeraLinkState.self_reported_latency_ms` can be **`null`** when saturated — callers must never treat null as 0.
- `Universe.nodesById` and `Universe.linksById` are `Map` objects for O(1) lookup by the router.
- `Phase2RoutingReport` matches the Council's mandatory schema exactly — missing fields cause disqualification.

---

## Relationships

```
UniverseMetadata ──→ Universe ──→ [ PlanetNode[] ]
                              └──→ [ InterplanetaryLink[] ]

Packet ──→ [ HopLogEntry[] ]
         └──→ PayloadStatus

LinkEvaluation ──→ Phase2RoutingReport
```

Back to [[00 - Index]]
