# `src/lib/chimera/agent/copilot.ts` — Co-Pilot Agent Orchestrator

**Tags:** `#agent` `#orchestration` `#phase-2` `#phase-3`  
**See also:** [[chimera-parser-hybrid]] · [[chimera-client]] · [[relic-router]] · [[chimera-link-evaluation]] · [[chimera-routing-report]]

---

## Purpose

The **main Co-Pilot agent** implementing the sequential evaluation loop defined in challenge §4. Accepts a natural-language routing request and produces a `Phase2RoutingReport`.

---

## Algorithm: `routeWithCopilot(nlRequest)`

### Phase 1: Parse NL Request

```ts
const intent = await parseRoutingRequest(nlRequest, nodeIds);
// → { origin_id, destination_id, payload, confidence, source }
```

### Phase 2: Prefetch Live State

```ts
await chimeraClient.getState();
```

Populates `chimeraClient.linksById` map for O(1) per-hop lookup.

### Phase 3: Baseline Physics Path (for explanation only)

```ts
const baseline = findShortestRoute(universe, geometry, origin, destination);
```

Used only for the explanation narrative — not the routing decision.

### Phase 4: Sequential Link Evaluation Loop

```
current = origin
blockedEdges = Set<string>()
while current !== destination:
  segment = findShortestRoute(current → destination, blocked)
  next = segment.path[1]
  linkState = chimeraClient.getLinkState(linkId) ?? neutralLinkState(...)
  scored = evaluateLink(linkState, physicsVoidMs)

  if scored.blocked:
    blockedEdges.add(edgeKey(current, next))
    reroutes++
    continue  ← try again from same current node

  evaluations.push(scored.evaluation)
  path.push(next)
  current = next
```

**Why sequential?** The challenge §4 requires evaluating one hop at a time, not all at once. Each rejected hop adds to `blockedEdges` and forces rerouting from the same node.

**Reroute cap:** `MAX_REROUTES = 64` prevents infinite loops if the network is severely congested.

### Phase 5: Build Report

```ts
physicsLatency = computePhysicsLatencyForPath(path)
explanation = [
  "Co-Pilot routed ... from X to Y via ..."
  "Baseline physics path was ..."
  "Sequential eval triggered N reroutes; blocked: ..."
  "Final path diverged/matched baseline."
  "Congestion penalties on ..."
  "Anomaly detected on ... / All telemetry normal."
].join(" ")

return buildRoutingReport({ origin, destination, path, evaluations, physicsLatency, explanation })
```

---

## Anomaly Tracking

When a link is anomalous but not blocked, its anomaly reason is tracked:

```ts
if (scored.anomaly && scored.anomalyReason) {
  anomalies.push(`${linkId} (${reason})`);
}
```

Anomalies appear in the explanation string, giving the Council visibility into OOD telemetry.

---

## Key Design Decisions

| Decision                           | Rationale                                                                       |
| ---------------------------------- | ------------------------------------------------------------------------------- |
| Sequential (not batch) evaluation  | Challenge §4 requirement                                                        |
| Reroute from same node, not origin | More efficient — avoids revisiting already-safe hops                            |
| Anomalous links NOT hard-blocked   | Keep them as last resort; combined_cost steers away when alternatives exist     |
| Baseline computed separately       | For explanation narrative only; the actual chosen path is the sequential result |
| Explanation built as array.join()  | Each sentence is an independent clause — easy to add/remove conditions          |

---

Back to [[00 - Index]]
