# `src/app/api/route/route.ts` — POST /api/route

**Tags:** `#api` `#routing` `#co-pilot` `#phase-3`  
**See also:** [[chimera-agent-copilot]] · [[chimera-router-true-cost]] · [[chimera-report-schema]] · [[api-rate-limit]] · [[api-validate-route]]

---

## Purpose

**Phase 3 Co-Pilot routing endpoint.** Accepts either a natural-language request or a structured origin/destination body. Returns a mandatory Council `Phase2RoutingReport`.

---

## Endpoint

```
POST /api/route
```

Rate-limited. Requires Chimera team key for `/state` access.

---

## Request Formats

### Natural Language

```json
{ "request": "Send Hello world from Aegis to Caelum" }
```

→ Routes via [[chimera-agent-copilot]] (sequential per-hop evaluation)

### Structured True Cost

```json
{
  "origin_id": "Aegis",
  "destination_id": "Caelum",
  "payload": "Hello world"
}
```

→ Routes via [[chimera-router-true-cost]] (global Dijkstra with surcharges)

---

## Response

```json
{
  "origin_id": "Aegis",
  "destination_id": "Caelum",
  "chosen_path": ["Aegis", "Boreas", "Caelum"],
  "link_evaluations": [
    {
      "link_id": "Aegis-Boreas",
      "predicted_congestion_penalty_ms": 1234.5,
      "trust_score": 0.9200,
      "targeting_risk_score": 0.1534,
      "combined_cost": 72345.6
    },
    ...
  ],
  "final_latency_estimate_ms": 75580.1,
  "explanation": "True Cost route from Aegis to Caelum: Aegis → Boreas → Caelum. ..."
}
```

---

## Error Handling Map

| Error Condition                   | HTTP Code | Error Code            |
| --------------------------------- | --------- | --------------------- |
| Invalid JSON body                 | 400       | `INVALID_JSON`        |
| Missing/invalid fields            | 400       | `VALIDATION_ERROR`    |
| Unknown origin/destination planet | 400       | `VALIDATION_ERROR`    |
| NL parser can't parse request     | 400       | `PARSER_ERROR`        |
| No route available                | 422       | `ROUTING_ERROR`       |
| CHIMERA_TEAM_KEY missing          | 503       | `CHIMERA_UNAVAILABLE` |
| Other engine error                | 500       | `ENGINE_ERROR`        |

---

## Validation Flow

```
1. enforceRateLimit()
2. request.json()         → parse body
3. validateRouteBody()    → determine mode: "nl" or "structured"
4. getEngine()            → validate planet IDs for structured mode
5. routeWithCopilot() OR routeWithTrueCost()
6. assertValidRoutingReport()   ← Council schema guard
7. Response.json(report)
```

---

## Design Notes

- **Output validation** (`assertValidRoutingReport`) is the last line of defense — even if our code generates a logically correct report, a Zod schema failure here prevents schema violations from reaching the Council evaluator.
- The route distinguishes error types precisely so clients can programmatically react to each case.

---

Back to [[00 - Index]]
