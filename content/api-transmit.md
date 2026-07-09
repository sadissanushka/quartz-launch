# `src/app/api/transmit/route.ts` — POST /api/transmit

**Tags:** `#api` `#transmission` `#hop-log` `#milestone-M2` `#milestone-M3` `#milestone-M4`  
**See also:** [[relic-transmission]] · [[chimera-router-true-cost]] · [[chimera-routing-report]] · [[api-validate-transmit]] · [[api-rate-limit]]

---

## Purpose

**M2/M3/M4 — Full packet transmission.** Transmits a payload from origin to destination, returning the complete packet with hop_log, the route, and the reconstructed payload. Optionally overlays a Phase 2 Co-Pilot routing report and constrains transmission to the intelligent path.

---

## Endpoint

```
POST /api/transmit
```

Rate-limited.

---

## Request Body

```json
{
  "origin": "Aegis",
  "destination": "Caelum",
  "payload": "Hello world",
  "blockedNodes": ["Boreas"],
  "blockedEdges": [["Aegis", "Dawn"]],
  "use_copilot": true
}
```

| Field          | Required | Description                                         |
| -------------- | -------- | --------------------------------------------------- |
| `origin`       | ✅       | Source planet ID                                    |
| `destination`  | ✅       | Target planet ID                                    |
| `payload`      | ✅       | Message string (max 10KB)                           |
| `blockedNodes` | Optional | Planet IDs to treat as offline (M4 chaos)           |
| `blockedEdges` | Optional | `[a, b]` pairs to treat as severed                  |
| `use_copilot`  | Optional | Attach Co-Pilot report + route via intelligent path |

---

## Transmission Flow

### Without `use_copilot`

```
transmit(universe, geometry, codec, origin, destination, payload, options)
→ Response.json(result)
```

Returns: `{ packet, route, delivered_payload }`

### With `use_copilot: true`

```
1. Base transmission (physics route)
2. routeWithTrueCost(origin, destination, payload)
   → routing_report
3. buildBlockedEdgesExceptPath(routing_report.chosen_path)
   → blocks all edges EXCEPT the co-pilot's chosen path
4. transmit with constrained options
   → if deliverable → copilotResult replaces base result
                    → transmitted_on_copilot_path = true
5. Return { ...result, routing_report, routing_report_error, transmitted_on_copilot_path }
```

> [!IMPORTANT]
> The Co-Pilot overlay is **best-effort** — if the constrained transmission is undeliverable (e.g. a manually severed node is on the chosen path), the response falls back to the physics baseline. `routing_report_error` is populated with the failure reason.

---

## Response Body (with Co-Pilot)

```json
{
  "packet": { ... hop_log with codec proof ... },
  "route": { "path": [...], "total_latency_ms": 75580, "breakdown": {...} },
  "delivered_payload": "Hello world",
  "routing_report": { ... Phase2RoutingReport ... },
  "routing_report_error": null,
  "transmitted_on_copilot_path": true
}
```

---

## Error Handling

| Condition                  | Code                         |
| -------------------------- | ---------------------------- |
| Invalid JSON               | 400 `INVALID_JSON`           |
| Missing fields             | 400 `VALIDATION_ERROR`       |
| Payload > 10KB             | 413 `PAYLOAD_TOO_LARGE`      |
| > 50 blocked nodes         | 400 `BLOCKED_LIST_TOO_LARGE` |
| > 100 blocked edges        | 400 `BLOCKED_LIST_TOO_LARGE` |
| Unknown origin/destination | 400 `VALIDATION_ERROR`       |
| Engine failure             | 500 `ENGINE_ERROR`           |

---

Back to [[00 - Index]]
