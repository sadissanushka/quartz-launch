# 🌌 Stack Kings — Relic Ring Protocol · Data Flow Diagrams

> All request paths, data transformations, and module handoffs in one place.  
> Related: [[00 - Index]] · [[DEMO_SCRIPT_V3]]

---

## 1. High-Level System Overview

```
┌──────────────────────────────────────────────────────────────────────┐
│                         Browser / Client                             │
│                    (RelicDashboard UI + DevTools)                    │
└──────────────┬───────────────────────────────┬───────────────────────┘
               │ GET /api/health               │ GET /api/universe
               │ POST /api/transmit            │ POST /api/route
               ▼                               ▼
┌──────────────────────────────────────────────────────────────────────┐
│                       Next.js API Routes                             │
│         src/app/api/{health,universe,transmit,route}/route.ts        │
│                                                                      │
│  ┌─────────────┐  ┌──────────────┐  ┌───────────────────────────┐   │
│  │ rate-limit  │  │  validate-*  │  │  errors / constants       │   │
│  │ (api/rate-  │  │  (Zod parse) │  │  (src/lib/api/)           │   │
│  │  limit.ts)  │  │              │  │                           │   │
│  └─────────────┘  └──────────────┘  └───────────────────────────┘   │
└──────────┬───────────────────────────────────┬───────────────────────┘
           │                                   │
           ▼                                   ▼
┌──────────────────────┐           ┌───────────────────────────────────┐
│   Phase 1 · Relic    │           │        Phase 2 · Chimera          │
│   (pure physics)     │           │        (AI overlay)               │
│   src/lib/relic/     │           │        src/lib/chimera/           │
└──────────────────────┘           └───────────────────────────────────┘
```

---

## 2. Phase 1 — Relic Physics Engine

### 2a. `POST /api/transmit` — Full Packet Transmission

```
POST /api/transmit
  { origin, destination, payload, blockedNodes?, blockedEdges? }
         │
         │ validate-transmit.ts (Zod)
         ▼
   src/app/api/transmit/route.ts
         │
         │ createEngine()  ──────────────────────────────────────────┐
         ▼                                                            │
   engine.ts                                                          │
   ┌──────────────────────────────────────────────────────────────┐  │
   │  loadUniverse()     → server/universe.ts → planets.json      │  │
   │  buildNetworkGraph()→ graph.ts                               │  │
   │  ResilientNetwork() → resilience.ts (apply blockedNodes/     │  │
   │                                      blockedEdges)           │  │
   └──────────────────────────────────────────────────────────────┘  │
         │                                                            │
         │ transmit(origin, destination, payload, resilientNetwork)   │
         ▼                                                            │
   transmission.ts                                                    │
   ┌──────────────────────────────────────────────────────────────┐  │
   │  1. findShortestRoute()  ─────────────────────────────────┐  │  │
   │       router.ts                                           │  │  │
   │       ┌──────────────────────────────────────────────┐   │  │  │
   │       │ State space: (planetId, entryTower)          │   │  │  │
   │       │ Priority queue (min-heap) over total_ms      │   │  │  │
   │       │ Per edge:                                    │   │  │  │
   │       │   Tv = computeVoidLatency()    latency.ts    │   │  │  │
   │       │   Tp = computeInternalLatency() latency.ts   │   │  │  │
   │       │ reachable() checks blockedNodes + blockedEdges│  │  │  │
   │       └──────────────────────────────────────────────┘   │  │  │
   │       → RouteResult { path[], total_latency_ms,          │  │  │
   │                       breakdown, hop_sequence[] }        │  │  │
   │                                                          │  │  │
   │  2. Codec loop per hop  ──────────────────────────────────┘  │  │
   │       codec.ts · RelicCodec                                   │  │
   │       ┌──────────────────────────────────────────────┐       │  │
   │       │ For each planet in path:                     │       │  │
   │       │   encodeToCodex(payload, codex)              │       │  │
   │       │     ASCII bytes → base-N digit strings       │       │  │
   │       │     serialised → 8-bit binary groups         │       │  │
   │       │     reversed (integrity layer)               │       │  │
   │       │   decodeFromCodex() → original bytes         │       │  │
   │       └──────────────────────────────────────────────┘       │  │
   │       → HopLogEntry[] { sequence, planet_id, codex,          │  │
   │                         entry_tower, exit_tower,             │  │
   │                         segments, towers_hit,                │  │
   │                         internal_latency_ms,                 │  │
   │                         void_latency_ms,                     │  │
   │                         cumulative_latency_ms,               │  │
   │                         payload_dialect }                    │  │
   │                                                              │  │
   │  3. Assemble Packet                                          │  │
   │       { status, hop_log, delivered_payload,                  │  │
   │         undeliverable_reason? }                              │  │
   └──────────────────────────────────────────────────────────────┘  │
         │                                                            │
         ▼                                                            │
   API response JSON                                                  │
   { route, packet, delivered_payload }                               │
```

### 2b. Latency Formula Detail

```
computeVoidLatency(Tv)                computeInternalLatency(Tp)
─────────────────────────────         ──────────────────────────────────
voidDistanceKm                        fiberArcLen = (2π × radius_km ×
  + atmRefraction(origin)                           segments) / towers
  + atmRefraction(destination)        fiberSpeed   = FIBER_SPEED_FACTOR
─────────────────────────────         ──────────────────────────────────
÷ SPEED_OF_LIGHT_KM_PER_MS     →Tv   fiberArcLen / fiberSpeed
                                        + towersHit × TOWER_OVERHEAD_MS → Tp
```

### 2c. Resilience — Failure Injection

```
ResilientNetwork (resilience.ts)
         │
         ├── markNodeDown(planetId)   → adds to _deadNodes Set
         ├── markEdgeDown(a, b)       → adds to _deadEdges Set
         ├── isNodeAlive(planetId)    → !_deadNodes.has(id)
         └── isEdgeAlive(a, b)        → !_deadEdges.has(edgeKey)

router.ts · reachable(edge, opts)
   checks: ResilientNetwork.isNodeAlive(from)
         + ResilientNetwork.isNodeAlive(to)
         + ResilientNetwork.isEdgeAlive(from, to)
         + blockedNodes (per-request override)
         + blockedEdges (per-request override)
```

---

## 3. Phase 2 — Chimera Co-Pilot Engine

### 3a. `POST /api/route` — Routing Request Fan-Out

```
POST /api/route
  { origin_id, destination_id }          ← structured (True Cost Router)
  { request: "NL string" }               ← natural language (Co-Pilot)
         │
         │ validate-route.ts (Zod)
         ▼
   src/app/api/route/route.ts
         │
         ├─ has (origin_id + destination_id)?
         │       └── routeWithTrueCost()  ──────────────────────────┐
         │                                                           │
         └─ has request?                                             │
                 └── routeWithCopilot()  ───────────────────────────┤
                                                                     │
                                                                     ▼
```

### 3b. True Cost Router Path

```
routeWithTrueCost(origin_id, destination_id)
   ├── chimeraClient.getState()   → live Chimera /state JSON
   │       (chimera/client.ts — HTTP fetch to external Chimera service)
   │
   ├── findShortestRoute()        → physics path (relic/router.ts)
   │       base Dijkstra, no surcharges yet
   │
   ├── For each link in path:
   │     evaluateLink(link, chimeraState)   (chimera/link-evaluation.ts)
   │     ┌──────────────────────────────────────────────────────────┐
   │     │  detectLinkAnomaly(link)                                 │
   │     │    → clamp out-of-range load_ratio / capacity            │
   │     │    → flag anomaly, apply floors:                         │
   │     │        trust_score ≥ 0.3, targeting_risk ≥ 0.85         │
   │     │                                                          │
   │     │  predictCongestion(link)   (models/congestion.ts)        │
   │     │    k × load_ratio^p  (power-law)                         │
   │     │    → hard-block if load_ratio ≥ 0.90 or "saturated"     │
   │     │                                                          │
   │     │  scoreTrust(link)          (models/trust.ts)             │
   │     │    physics Tv + congestion_ms vs self-reported latency   │
   │     │    delta > threshold → spoofing signal → lower score     │
   │     │                                                          │
   │     │  scoreTargetingRisk(link)  (models/targeting.ts)         │
   │     │    exposure/interception probability per hop             │
   │     │                                                          │
   │     │  combined_cost =                                         │
   │     │    void_latency_ms                                       │
   │     │    + congestion_penalty_ms                               │
   │     │    + (1 - trust_score) × TRUST_PENALTY_FACTOR           │
   │     │    + targeting_risk × TARGETING_PENALTY_FACTOR          │
   │     │    - entropy_bonus   (steers load toward busier links)   │
   │     └──────────────────────────────────────────────────────────┘
   │
   ├── Re-run Dijkstra with combined_cost as edge weights
   │     (5% diversification tolerance applied)
   │
   └── Return RoutingReport
         { path[], explanation, link_evaluations[],
           congestion_penalties[], trust_scores[],
           targeting_risks[], combined_costs[] }
```

### 3c. Co-Pilot Path (Sequential Agent)

```
routeWithCopilot(request: string)
   │
   ├── parseRoutingRequest(request)     (chimera/parser/hybrid.ts)
   │     ┌──────────────────────────────────────────────────────┐
   │     │  Step 1 — Regex rules                                │
   │     │    "from X to Y", "X to Y", "X → Y" patterns        │
   │     │    → { origin, destination, parse_source: "regex" }  │
   │     │                                                      │
   │     │  Step 2 — LLM fallback (if regex fails)             │
   │     │    parseLLM(request)   (chimera/parser/llm.ts)       │
   │     │    → Gemini/OpenAI structured output                 │
   │     │    → { origin, destination, parse_source: "llm" }   │
   │     └──────────────────────────────────────────────────────┘
   │
   ├── chimeraClient.getState()    (live network state)
   │
   ├── Sequential hop evaluation (unlike batch Dijkstra):
   │     current = origin
   │     while current ≠ destination:
   │       candidates = neighbours(current)
   │       score each candidate with evaluateLink()
   │       pick best → next hop
   │       if stuck → reroute_count++, backtrack
   │
   └── Return RoutingReport
         { path[], explanation, reroute_count,
           parse_source, link_evaluations[] }
```

---

## 4. `GET /api/universe` — Universe Graph Loader

```
GET /api/universe
         │
         ▼
   src/app/api/universe/route.ts
         │
         │ loadUniverse()   (relic/server/universe.ts)
         │   → reads planets.json (node-only, server-safe)
         │   → builds Universe { nodes: PlanetNode[], edges: VoidEdge[] }
         ▼
   buildNetworkGraph(universe)   (relic/graph.ts)
         │
         │  For each pair of planets:
         │    voidDistanceKm = geometry.distance(a.position, b.position)
         │    if voidDistanceKm <= LMAX:
         │      push VoidEdge { from, to, void_distance_km,
         │                      within_lmax: true }
         ▼
   Response: { nodes[], edges[], interplanetaryLinks[] }
```

---

## 5. UI → API Data Flow (RelicDashboard)

```
RelicDashboard.tsx (src/components/telemetry/)
   │
   ├── SpaceMap.tsx
   │     fetch GET /api/universe
   │     → renders planet nodes + edges on canvas/SVG
   │     → highlights active route path
   │
   ├── CodexTerminal.tsx
   │     user input: origin, destination, payload
   │     fetch POST /api/transmit
   │     → renders hop_log codec steps
   │     → shows base-N digit dialects per planet
   │
   ├── LatencyMetrics.tsx
   │     parses route.breakdown from /api/transmit response
   │     → bar chart: fiber_ms, tower_ms, atmosphere_ms, void_ms
   │
   ├── LinkEvaluationsPanel.tsx
   │     fetch POST /api/route (structured)
   │     → renders link_evaluations[]
   │     → congestion / trust / targeting / combined_cost per hop
   │
   └── IntelligenceSummary.tsx
         reads RoutingReport.explanation
         → prose summary of chosen path reasoning
```

---

## 6. Key Data Types — Transformation Chain

```
Input                  Intermediate                     Output
──────────────────────────────────────────────────────────────────────
"Hello world"    →    ASCII bytes [72,101,108...]  →  base-8 digits
                 →    binary stream (8-bit groups)  →  reversed stream
                 →    re-encoded base-6 at Dawn     →  re-encoded base-14
                 →    decoded back at destination   →  "Hello world" ✓

PlanetNode[]     →    adjacency list (graph.ts)    →  priority queue
                 →    Dijkstra states (planet,tower)→  RouteResult.path[]
                 →    hop_sequence[]               →  HopLogEntry[]

ChimeraState     →    evaluateLink() per edge      →  LinkEvaluation[]
                 →    combined_cost per edge        →  weighted Dijkstra
                 →    RoutingReport                 →  API response JSON
```

---

## 7. Module Dependency Graph

```
engine.ts
  ├── server/universe.ts   (loads planet data)
  ├── graph.ts             (builds adjacency graph)
  ├── resilience.ts        (failure injection)
  └── transmission.ts
        ├── router.ts      (Dijkstra — depends on graph, latency)
        │     └── latency.ts
        └── codec.ts       (ASCII ↔ base-N ↔ binary)

chimera/index.ts (barrel)
  ├── agent/copilot.ts
  │     ├── parser/hybrid.ts
  │     │     └── parser/llm.ts
  │     ├── client.ts
  │     └── link-evaluation.ts
  │           ├── models/congestion.ts
  │           ├── models/trust.ts
  │           └── models/targeting.ts
  └── router/true-cost-router.ts
        ├── client.ts
        └── link-evaluation.ts
```

---

## 8. Error & Edge Case Flows

```
findShortestRoute() undeliverable paths:
  ┌─ origin is in blockedNodes/deadNodes
  │     → return immediately: { status: "undeliverable",
  │                              reason: "Origin is offline" }
  ├─ destination is in blockedNodes/deadNodes
  │     → return immediately: { status: "undeliverable",
  │                              reason: "Destination is offline" }
  └─ no path within Lmax after exclusions
        → exhausted priority queue: { status: "undeliverable",
                                       reason: "No route within Lmax" }

detectLinkAnomaly() anomaly handling:
  load_ratio > 1.0        → clamp to 1.0, flag anomaly
  capacity < 0            → clamp to 0, flag anomaly
  flagged link scores:
    trust_score      = max(reported, 0.3)   ← floor
    targeting_risk   = max(reported, 0.85)  ← floor
    (remains usable but expensive — last resort only)

parseRoutingRequest() fallback chain:
  regex match found   → { parse_source: "regex" }  (fast path)
  regex fails         → LLM call                   (slow path)
  LLM fails/timeout   → 500 error propagated to client
```

---

## Tags

`#stack-kings` `#relic-ring-protocol` `#chimera` `#data-flow` `#architecture`
