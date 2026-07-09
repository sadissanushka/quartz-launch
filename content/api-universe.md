# `src/app/api/universe/route.ts` — GET /api/universe

**Tags:** `#api` `#universe` `#milestone-M1`  
**See also:** [[relic-server-universe]] · [[relic-graph]] · [[api-constants]]

---

## Purpose

**M1 — Universe Initialization.** Returns the full parsed universe: metadata, planet nodes, and the derived hop graph.

---

## Endpoint

```
GET /api/universe
```

No authentication. Rate-limited. Responses are heavily cached.

---

## Response Body

```json
{
  "metadata": {
    "system_name": "Zeta-26",
    "speed_of_light_kms": 300000,
    "max_void_hop_distance_km": 50000000,
    "coordinate_scale_unit_km": 1000,
    "tower_processing_delay_ms": 7,
    "fiber_speed_fraction": 0.67
  },
  "nodes": [
    {
      "id": "Aegis",
      "codex": 5,
      "x": 10, "y": 20,
      "radius_km": 6000,
      "active_towers": 8,
      "atmosphere_thickness_km": 200,
      "refraction_index": 1.3
    },
    ...
  ],
  "adjacency": {
    "Aegis": ["Boreas", "Caelum", ...],
    ...
  },
  "edges": [
    { "from": "Aegis", "to": "Boreas", "void_distance_km": 60093.2, "within_lmax": true },
    ...
  ]
}
```

---

## Caching

```ts
headers: {
  "Cache-Control": `public, max-age=${UNIVERSE_CACHE_MAX_AGE_SEC}, s-maxage=${UNIVERSE_CACHE_MAX_AGE_SEC}`
}
```

`UNIVERSE_CACHE_MAX_AGE_SEC = 3600` — cached for **1 hour** by CDN and browser.  
Universe config doesn't change at runtime so this is safe.

---

## Implementation Notes

- `adjacency` is a plain object (from `Object.fromEntries(graph.adjacency)`) because `Map` doesn't serialize to JSON.
- `runtime = "nodejs"` for filesystem access.
- Errors return HTTP 500 with `ENGINE_ERROR`.

---

Back to [[00 - Index]]
