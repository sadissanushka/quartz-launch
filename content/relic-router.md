# `src/lib/relic/router.ts` — Lowest-Latency Router (Dijkstra)

**Tags:** `#routing` `#dijkstra` `#phase-1` `#phase-2`  
**See also:** [[relic-latency]] · [[relic-graph]] · [[chimera-router-true-cost]] · [[chimera-agent-copilot]]

---

## Purpose

Finds the **lowest-latency route** from origin to destination using a modified Dijkstra over an **expanded state space**. The key insight: internal crust transit `Tp` depends on BOTH the entry tower (set by the incoming hop) AND the exit tower (set by the outgoing hop). A plain node-weighted Dijkstra would be incorrect here.

---

## State Space Expansion

Instead of `state = (planet_id)`, the algorithm uses:

```
state = (planet_id, entry_tower)
stateKey = "${planetId}#${entryTower}"
```

This ensures the correct `Tp` is charged when the packet leaves each planet, since the exit tower (determined by the next hop) is already known in the Dijkstra relaxation step.

---

## Algorithm Walk-Through

```
1. Push start state: (originId, NO_ENTRY=-1) with dist=0
2. While heap not empty:
   a. Pop lowest-dist state (planetId, entryTower)
   b. If planetId == destination → done
   c. For each neighbor within Lmax (not blocked):
      - exitTower = closestTowerPair(planet, neighbor).origin_tower
      - effectiveEntry = (entryTower == NO_ENTRY) ? exitTower : entryTower
      - Tp = computeInternalLatency(planet, effectiveEntry, exitTower)
      - Tv = computeVoidLatency(planet, neighbor) + optional surcharge
      - nextDist = dist + Tp + Tv
      - Push (neighbor, destinationTower) if nextDist improves
3. Reconstruct path by walking prev[] back to origin
4. Compute destination Tp (entry_tower === exit_tower → s=0, m=1)
5. Return Route with steps, hops, breakdown, total_latency_ms
```

---

## Public API

```ts
findShortestRoute(
  universe: Universe,
  geometry: GeometryProvider,
  originId: string,
  destinationId: string,
  options?: RouteOptions,
): Route
```

**`RouteOptions`:**

| Option               | Effect                                                          |
| -------------------- | --------------------------------------------------------------- |
| `blockedNodes`       | Planets skipped entirely (node failures)                        |
| `blockedEdges`       | Severed links (edge failures)                                   |
| `voidHopSurchargeMs` | Extra cost added per hop — used by [[chimera-router-true-cost]] |

**`Route`:**

```ts
{
  deliverable: boolean;
  path: string[];        // ordered planet IDs
  steps: RouteStep[];    // per-planet internal transit
  hops: RouteHop[];      // per-hop void details
  breakdown: LatencyBreakdown;
  total_latency_ms: number;
}
```

---

## MinHeap

A custom **binary min-heap** is implemented directly (lines 86-161), avoiding a dependency on a priority-queue library. It's a standard textbook heap with `bubbleUp` and `bubbleDown`. This is safe in TypeScript with small-to-medium planet counts.

> [!TIP]
> For very large universe configs (100+ planets), a Fibonacci heap would improve theoretical complexity. In practice, the challenge universe has ~6 planets so this is not a concern.

---

## Edge Cases

| Scenario                     | Behavior                                             |
| ---------------------------- | ---------------------------------------------------- |
| Origin = Destination         | Returns trivial route with single step (1 tower hit) |
| Blocked origin/dest          | Returns `undeliverable` immediately                  |
| No path within Lmax          | Returns `undeliverable` with descriptive reason      |
| Surcharge returns `Infinity` | Edge is effectively blocked (Dijkstra never relaxes) |

---

## `findBaselineRoute`

Alias for `findShortestRoute` — marks the Phase 1 physics-only path, used in documentation and by the Co-Pilot to compute a baseline for comparison.

---

Back to [[00 - Index]]
