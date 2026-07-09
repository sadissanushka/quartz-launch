# `src/lib/chimera/routing-report.ts` — Report Builder Helpers

**Tags:** `#helpers` `#phase-2`  
**See also:** [[chimera-agent-copilot]] · [[chimera-router-true-cost]] · [[chimera-link-evaluation]]

---

## Purpose

**Shared utilities** for building `Phase2RoutingReport` objects. Both the Co-Pilot agent and the True Cost router use these helpers to avoid duplicating logic.

---

## Functions

### `buildRoutingReport(params) → Phase2RoutingReport`

```ts
buildRoutingReport({
  origin_id,
  destination_id,
  path,
  link_evaluations,
  physics_latency_ms,
  explanation,
});
```

Computes `final_latency_estimate_ms = physics_latency_ms + Σ(congestion penalties)` and assembles the complete report object.

```ts
final_latency_estimate_ms = round(
  physics_latency_ms + sum(link_evaluations[i].predicted_congestion_penalty_ms),
);
```

### `computePhysicsLatencyForPath(universe, geometry, path) → number`

Re-runs the physics router constrained to only the chosen path's edges:

1. Builds `blockedEdgesExceptPath` — all edges NOT on the chosen path
2. Calls `findShortestRoute` with those blocked edges
3. Verifies the returned path matches the intended path
4. Returns `route.total_latency_ms`

> [!WARNING]
> Throws if the physics router cannot reconstruct the given path. This should never happen in practice since the path was derived from the same router.

### `buildBlockedEdgesExceptPath(universe, geometry, path) → [string, string][]`

Collects all pairs of planets that are within Lmax but **not** on the given path. Used to constrain the physics router to exactly one path for the hop_log in [[api-transmit]].

### `evaluatePathLinks(route, linkStateById, capacityByLinkId) → LinkEvaluation[]`

Evaluates every hop on a route:

```ts
route.hops.map(hop => {
  const linkState = linkStateById.get(linkId) ?? neutralLinkState(...)
  return evaluateLink(linkState, hop.void.total_ms).evaluation
})
```

Uses [[chimera-link-evaluation#neutralLinkState|`neutralLinkState`]] as fallback when Chimera doesn't have data for a link.

### `formatPathExplanation(path) → string`

`["Aegis", "Boreas", "Caelum"] → "Aegis → Boreas → Caelum"`

---

## Usage Pattern

Both routers follow the same assembly pattern:

```
1. Get live state (chimeraClient.getState())
2. Find route (findShortestRoute with surcharges or sequential eval)
3. evaluatePathLinks() → LinkEvaluation[]
4. computePhysicsLatencyForPath() → physics_latency_ms
5. buildRoutingReport() → Phase2RoutingReport
```

---

Back to [[00 - Index]]
