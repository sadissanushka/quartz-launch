# `src/lib/chimera/router/true-cost-router.ts` — True Cost Router

**Tags:** `#routing` `#optimization` `#phase-2` `#entropy`  
**See also:** [[chimera-link-evaluation]] · [[chimera-constants]] · [[relic-router]] · [[chimera-routing-report]]

---

## Purpose

A **batch routing approach** that computes the globally optimal path using True Cost weights, vs. the Co-Pilot agent's sequential per-hop approach. Adds route diversification via entropy maximisation.

---

## Algorithm: `routeWithTrueCost(options)`

### Step 1: Fetch Live State

```ts
const state = await chimeraClient.getState();
const linkStateById = new Map(state.links.map((l) => [l.link_id, l]));
```

### Step 2: Pre-compute Hard Blocks

Evaluate all known links upfront; any that are blocked (saturated or low trust) are added to `hardBlocked: Set<string>`.

### Step 3: Surcharge Function

```ts
const surcharge = (from, to, physicsVoidMs) => {
  const scored = evaluateLink(linkState, physicsVoidMs);
  if (scored.blocked) return Infinity; // Dijkstra never relaxes this edge
  return scored.evaluation.combined_cost - physicsVoidMs;
};
```

This function is passed to [[relic-router]] via `voidHopSurchargeMs` — the router adds it on top of physical void latency during Dijkstra.

### Step 4: Find Optimal Candidate

```ts
const optimal = buildCandidate(new Set());
// = findShortestRoute with surcharges + hardBlocked
```

`buildCandidate(extraBlocked)`:

1. Run `findShortestRoute` with hardBlocked ∪ extraBlocked + surchargeMs
2. `evaluatePathLinks()` — evaluate all hops on the returned path
3. Compute `trueCostSum`, `targetingSum`

### Step 5: Route Diversification

```
threshold = optimal.trueCostSum × (1 + ROUTE_DIVERSIFICATION_EPSILON = 0.05)

for each hop on optimal.route:
  candidate = buildCandidate(blocked that hop)
  if candidate.trueCostSum > threshold: skip

  prefer candidate if:
    candidate.targetingSum < chosen.targetingSum - 1e-9
    OR (sums nearly equal AND candidate.trueCostSum < chosen.trueCostSum)
```

The diversification loop trades up to 5% extra True Cost for the path with the lowest **aggregate targeting risk** — entropy maximisation.

---

## Explanation Generation

```ts
explanationParts = [
  "True Cost route from X to Y: A → B → C.",
  "Avoided N blocked links (saturated or low trust)." / "No links hard-blocked this tick.",
  "Near-optimal path selected for lower targeting risk." / "Chosen path was lowest True Cost.",
  "Elevated targeting risk on: ..." / "Targeting risk moderate on all hops.",
  "Anomaly detected on ..." / "All link telemetry within expected distribution.",
  "Payload length: N characters." (if payload provided),
]
```

---

## Comparison: True Cost Router vs. Co-Pilot Agent

| Aspect           | True Cost Router                            | Co-Pilot Agent                     |
| ---------------- | ------------------------------------------- | ---------------------------------- |
| Evaluation style | **Batch** (global Dijkstra with surcharges) | **Sequential** (hop-by-hop)        |
| Input            | Structured `{origin_id, destination_id}`    | NL string                          |
| Rerouting        | Diversification (near-optimal alternatives) | Block + re-route from current node |
| LLM usage        | None                                        | Via [[chimera-parser-hybrid]]      |
| Use via API      | `POST /api/route` (structured body)         | `POST /api/route` (NL request)     |

---

Back to [[00 - Index]]
