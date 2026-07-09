# `src/lib/relic/config.ts` — Universe Config Parser & Validator

**Tags:** `#config` `#validation` `#phase-1` `#phase-2`  
**See also:** [[relic-types]] · [[relic-server-universe]]

---

## Purpose

Parses and strictly validates `universe-config.json` into a typed [`Universe`](relic-types) object. This is where raw JSON becomes a safe, structured object — every field is validated before the engine touches it.

---

## Public API

```ts
parseUniverseConfig(raw: unknown): Universe
```

Main entry point. Throws `RelicConfigError` on any structural or semantic error.

```ts
class RelicConfigError extends Error
```

Typed error subclass used for all config failures — caught by the server and API layers.

```ts
const METADATA_DEFAULTS = {
  speed_of_light_kms: 300_000,
  max_void_hop_distance_km: 50_000_000,
  tower_processing_delay_ms: 7,
  fiber_speed_fraction: 0.67,
};
```

---

## Validation Pipeline

```
raw: unknown
  │
  ├─ parseMetadata()    → UniverseMetadata
  │    ├─ requireFiniteNumber() for mandatory fields
  │    └─ optionalFiniteNumber() with METADATA_DEFAULTS for optional fields
  │
  ├─ parseNode() × N    → PlanetNode[]
  │    ├─ codex must be integer ≥ MIN_CODEX_BASE (2)
  │    └─ active_towers must be integer ≥ MIN_ACTIVE_TOWERS (4)
  │
  ├─ Build nodesById Map (detects duplicate IDs)
  │
  └─ parseInterplanetaryLink() × M   → InterplanetaryLink[]
       ├─ planet_a/planet_b must exist in nodesById
       ├─ link_id must match canonicalLinkId(planet_a, planet_b)
       └─ validateInterplanetaryLinksAgainstPhysics() — each link L ≤ Lmax
```

---

## Key Design Decisions

### No Hardcoded Constants

Every physics value (speed of light, max hop distance, etc.) is read from `universe_metadata`. Defaults are only applied for the four explicitly "optional" fields the spec marks as defaultable.

### Phase 2 Backward Compatibility

`interplanetary_links` is **optional** — when absent, `interplanetaryLinks` is an empty array and Phase 2 features degrade gracefully. The Phase 1 engine still works against a config without links.

### Physics Validation for Links

After parsing links, `validateInterplanetaryLinksAgainstPhysics()` checks that every link's void distance ≤ Lmax using the stub geometry provider. Chimera links that exceed the physics constraint are rejected.

### Canonical Link ID Enforcement

`link_id` must equal `canonicalLinkId(planet_a, planet_b)` — i.e., alphabetically sorted planets joined by `-`. This is cross-checked at parse time to prevent mismatches between config and runtime.

---

## Internal Helpers

| Helper                    | Purpose                                     |
| ------------------------- | ------------------------------------------- |
| `requireFiniteNumber()`   | Throws on non-finite or missing fields      |
| `optionalFiniteNumber()`  | Falls back to a default when absent         |
| `requireNonEmptyString()` | Throws on empty/missing strings             |
| `positive()`              | Asserts `value > 0`                         |
| `nonNegative()`           | Asserts `value >= 0`                        |
| `describe()`              | Human-readable stringify for error messages |

---

## Error Examples

```
RelicConfigError: universe_metadata: "coordinate_scale_unit_km" must be a finite number, received undefined.
RelicConfigError: node "Aegis": "active_towers" must be an integer >= 4, received 3.
RelicConfigError: interplanetary_links[0]: "link_id" must be "Aegis-Boreas" (alphabetical), received "Boreas-Aegis".
```

---

Back to [[00 - Index]]
