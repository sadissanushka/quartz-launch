# `src/lib/chimera/link-evaluation.ts` — Link Scoring & Anomaly Detection

**Tags:** `#scoring` `#anomaly-detection` `#phase-2` `#phase-5`  
**See also:** [[chimera-constants]] · [[chimera-models-congestion]] · [[chimera-models-trust]] · [[chimera-models-targeting]]

---

## Purpose

The **central scoring hub** for the Co-Pilot. Given live link telemetry, it:

1. Detects and sanitises out-of-distribution (OOD) values (`detectLinkAnomaly`)
2. Runs all three analytical sub-models
3. Computes the `combined_cost` used by the router
4. Decides if the link is blocked or safe

This is Phase 5 hardened — every model output is guarded against NaN/Infinity.

---

## `detectLinkAnomaly(linkState) → AnomalyResult`

Validates every numeric field and clamps anomalous values:

| Field                      | Valid Range             | Anomaly Fallback                      |
| -------------------------- | ----------------------- | ------------------------------------- |
| `capacity_units`           | `> 0`                   | `100`                                 |
| `current_load`             | `>= 0`                  | `0`                                   |
| `load_ratio`               | `[0, 1]`                | `0.5` (NaN) or clamped                |
| `traffic_share`            | `[0, 1]`                | `0.5` (NaN) or clamped                |
| `self_reported_latency_ms` | `>= 0` or `null`        | `0` (null is OK, negative/NaN is not) |
| `status`                   | `"ok"` or `"saturated"` | `"ok"`                                |

Returns `{ anomalous, reasons[], sanitized }`.

> [!NOTE]
> `self_reported_latency_ms === null` is **not** an anomaly — it signals saturation and is handled explicitly by the congestion model.

---

## `evaluateLink(linkState, physicsVoidMs) → LinkScoreResult`

Full scoring pipeline:

```
1. detectLinkAnomaly(linkState)               → sanitized state
2. predictCongestion(sanitized, physicsVoidMs) → { penalty_ms, is_saturated }
3. scoreTrust(sanitized)                       → trust_score
4. scoreTargetingRisk(sanitized)               → targeting_risk_score

5. If anomalous:
     trustScore = min(trustScore, ANOMALY_CONSERVATIVE_TRUST=0.3)
     targetingRisk = max(targetingRisk, ANOMALY_CONSERVATIVE_TARGETING=0.85)

6. Clamp both to [0, 1]

7. combined_cost = physicsVoidMs
                 + penalty_ms
                 + (1 - trustScore) × TRUST_SCALE_MS
                 + targetingRisk × TARGETING_SCALE_MS
                 - traffic_share × ENTROPY_BONUS_SCALE_MS

8. Block if:
   - is_saturated → blocked (saturation)
   - trustScore < TRUST_FLOOR && !anomalous → blocked (known bad link)
     (anomalous links are NOT hard-blocked on trust alone)
```

### `LinkScoreResult`

```ts
{
  evaluation: LinkEvaluation;
  blocked: boolean;
  blockReason?: string;
  anomaly: boolean;
  anomalyReason?: string;
}
```

---

## Anomaly Routing Policy

| Condition                                                 | Action                                                                                                |
| --------------------------------------------------------- | ----------------------------------------------------------------------------------------------------- |
| Known spoofed link (`trust < TRUST_FLOOR`, not anomalous) | **Hard block** — never route through                                                                  |
| OOD telemetry (anomalous) with low trust                  | Conservative scores but **NOT hard-blocked** — still usable as last resort, combined_cost steers away |
| Saturated (`is_saturated = true`)                         | **Hard block**                                                                                        |

This distinction is critical: anomalous links stay available when no alternative exists.

---

## `neutralLinkState(planetA, planetB, capacityUnits=100) → ChimeraLinkState`

Creates a safe default link state for hops without live telemetry:

- `load_ratio = 0`, `traffic_share = 0`, `self_reported_latency_ms = 0`, `status = "ok"`
- Used by the router when Chimera doesn't return data for a specific link.

---

## Helper Functions

| Function         | Description                                 |
| ---------------- | ------------------------------------------- |
| `isBadNumber(v)` | True when not a finite number               |
| `clamp01(v)`     | Clamp to [0, 1], returning 0 for non-finite |
| `round3(v)`      | Round to 3 decimal places                   |
| `round4(v)`      | Round to 4 decimal places                   |

---

Back to [[00 - Index]]
