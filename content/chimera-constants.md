# `src/lib/chimera/constants.ts` — Chimera Routing Constants

**Tags:** `#constants` `#tuning` `#phase-2`  
**See also:** [[chimera-link-evaluation]] · [[chimera-router-true-cost]] · [[chimera-client]]

---

## Purpose

All **tuneable numeric constants** for the Co-Pilot routing system in one place. Every value is documented with a comment explaining its physical meaning.

---

## Constants Reference

### Trust

| Constant         | Value    | Meaning                                                        |
| ---------------- | -------- | -------------------------------------------------------------- |
| `TRUST_FLOOR`    | `0.5`    | Trust scores below this hard-block the link (unless anomalous) |
| `TRUST_SCALE_MS` | `80,000` | Multiplier: `(1 - trust) × 80,000 ms` added to combined cost   |

### Targeting

| Constant             | Value    | Meaning                                                         |
| -------------------- | -------- | --------------------------------------------------------------- |
| `TARGETING_SCALE_MS` | `50,000` | Multiplier: `targeting_risk × 50,000 ms` added to combined cost |

### Entropy

| Constant                 | Value    | Meaning                                                         |
| ------------------------ | -------- | --------------------------------------------------------------- |
| `ENTROPY_BONUS_SCALE_MS` | `25,000` | Reduction: `-traffic_share × 25,000 ms` rewards route diversity |

### Polling

| Constant                   | Value   | Meaning                                  |
| -------------------------- | ------- | ---------------------------------------- |
| `CHIMERA_POLL_INTERVAL_MS` | `1,500` | Min interval between live `/state` polls |

### Parser

| Constant                      | Value | Meaning                                         |
| ----------------------------- | ----- | ----------------------------------------------- |
| `PARSER_CONFIDENCE_THRESHOLD` | `0.6` | Below this, rules parser defers to LLM fallback |

### Routing

| Constant                        | Value  | Meaning                                              |
| ------------------------------- | ------ | ---------------------------------------------------- |
| `ROUTE_DIVERSIFICATION_EPSILON` | `0.05` | Near-optimal tolerance (5%) for entropy maximisation |

### Anomaly

| Constant                         | Value  | Meaning                                     |
| -------------------------------- | ------ | ------------------------------------------- |
| `ANOMALY_CONSERVATIVE_TRUST`     | `0.3`  | Trust cap for out-of-distribution telemetry |
| `ANOMALY_CONSERVATIVE_TARGETING` | `0.85` | Targeting risk floor for anomalous links    |

---

## True Cost Formula (Reference)

```
combined_cost = physics_void_ms
              + penalty_ms                          (congestion)
              + (1 - trust_score) × TRUST_SCALE_MS
              + targeting_risk_score × TARGETING_SCALE_MS
              - traffic_share × ENTROPY_BONUS_SCALE_MS
```

---

## Design Rationale

- **Trust scale (80k ms)** is much larger than targeting scale (50k ms), reflecting that a known-spoofed link is a hard security risk while targeting is probabilistic.
- **Entropy bonus** deliberately under-penalizes low-traffic links to encourage route diversity over time.
- **Anomaly conservative values** keep links usable (not hard-blocked) while strongly steering away from them via inflated combined_cost.

---

Back to [[00 - Index]]
