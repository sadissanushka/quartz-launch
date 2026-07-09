# `src/lib/chimera/models/congestion.ts` — Congestion Prediction Model

**Tags:** `#model` `#congestion` `#phase-2` `#ruwan`  
**See also:** [[chimera-models-trust]] · [[chimera-models-targeting]] · [[chimera-link-evaluation]] · [[chimera-constants]]

---

## Purpose

Predicts the **Chimera-induced congestion penalty** (extra latency above the physics baseline) for a single interplanetary link. Trained from `link_traffic_history.csv`.

**Owner:** Ruwan (Phase 1, section 1.2)

---

## Model Architecture

**Piecewise power-law regression** per link:

```
penalty_ms = k × (load_ratio ^ p)
```

Where `k` and `p` are per-link fitted coefficients stored in `congestion.model.json`:

```json
{
  "Aegis-Boreas": { "k": 450000, "p": 2.1 },
  ...
}
```

If a link ID is not in the JSON, the **global fallback** is used:

```ts
const GLOBAL_FALLBACK = { k: 600_000, p: 2.25 };
```

---

## `predictCongestion(linkState, physicsVoidMs) → CongestionResult`

```ts
interface CongestionResult {
  penalty_ms: number;
  is_saturated: boolean; // true = link is blocked (unavailable)
}
```

### Saturation Checks (in priority order)

```
1. status === "saturated"                → { penalty_ms: 0, is_saturated: true }
2. self_reported_latency_ms === null     → { penalty_ms: 0, is_saturated: true }
3. load_ratio >= 0.90                   → { penalty_ms: 0, is_saturated: true }
4. Otherwise: penalty_ms = k × load_ratio^p
```

> [!IMPORTANT]
> `physicsVoidMs` is accepted but currently unused (`_physicsVoidMs`). The eslint-disable comment acknowledges this. It was included in the interface to allow future models that need physics baseline context.

---

## Saturation Threshold

The 0.90 load_ratio threshold is a conservative fallback — even if `status === "ok"`, a link at 90%+ utilisation is treated as saturated. This prevents routing to a link that is effectively full.

---

## Data Flow

```
link_traffic_history.csv
  ↓ scripts/train-models.ts
congestion.model.json
  ↓ import (JSON)
CONGESTION_COEFFS map
  ↓ predictCongestion()
CongestionResult { penalty_ms, is_saturated }
```

---

Back to [[00 - Index]]
