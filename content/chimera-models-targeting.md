# `src/lib/chimera/models/targeting.ts` — Targeting-Risk Model

**Tags:** `#model` `#targeting` `#jamming` `#phase-2` `#ruwan`  
**See also:** [[chimera-models-congestion]] · [[chimera-models-trust]] · [[chimera-link-evaluation]]

---

## Purpose

Scores the **probability that Chimera will target (jam) a link** given its current traffic share. Trained from `link_incident_history.csv` using logistic regression.

**Owner:** Ruwan (Phase 1, section 1.4)

---

## Model Architecture

**Per-link logistic regression:**

```
z = b0 + b1 × traffic_share
P(jammed) = 1 / (1 + exp(-z))
```

Where `b0`, `b1` are per-link fitted coefficients stored in `targeting.model.json`:

```json
{
  "Aegis-Boreas": { "b0": -3.1, "b1": 4.2 },
  ...
}
```

**Global fallback** for unseen link IDs:

```ts
const GLOBAL_FALLBACK: LogisticParams = { b0: -2.73647, b1: 3.61594 };
```

---

## `scoreTargetingRisk(linkState) → number [0, 1]`

```ts
const params = TARGETING_MODEL[linkState.link_id] || GLOBAL_FALLBACK;
const z = params.b0 + params.b1 * linkState.traffic_share;
const probability = 1.0 / (1.0 + Math.exp(-z));
return parseFloat(probability.toFixed(4));
```

- Higher `traffic_share` → higher `z` → higher `P(jammed)` → higher risk
- Links carrying a large share of network traffic are more attractive targets for Chimera

---

## Interpretation

| `targeting_risk_score` | Meaning                                         |
| ---------------------- | ----------------------------------------------- |
| `< 0.3`                | Low risk — link is not a likely jamming target  |
| `0.3 – 0.5`            | Moderate risk                                   |
| `>= 0.5`               | Elevated risk (flagged in Co-Pilot explanation) |
| Near 1.0               | Very high — link is a likely primary target     |

---

## Impact on Routing

Used in `combined_cost`:

```
+ targeting_risk_score × TARGETING_SCALE_MS (50,000)
```

A high-risk link adds up to 50,000ms to its cost even with zero congestion, strongly discouraging routing through it.

The entropy bonus (`-traffic_share × ENTROPY_BONUS_SCALE_MS`) works in the opposite direction: lowering the cost for lightly-used links. Since `b1 > 0`, links with low `traffic_share` also have low targeting risk — the two mechanisms reinforce each other.

---

Back to [[00 - Index]]
