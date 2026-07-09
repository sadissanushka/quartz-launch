# 🌌 Stack Kings — Relic Ring Protocol · Demo Script
### Live 15-Minute Demo + Parallel Q&A

> **Format:** Presenter runs commands in a terminal; audience asks questions after each segment.  
> **Prerequisite:** `npm run dev` running in one terminal tab; a second tab open at `/home/anushka/Music/stack-kings`.  
> **Reference:** Open `obsidian-vault/00 - Index.md` in Obsidian for live vault navigation.

---

## 🕐 Timeline Overview

| Time          | Segment                                                          | Tool    |
| ------------- | ---------------------------------------------------------------- | ------- |
| 0:00 – 1:30   | **Intro + repo orientation**                                     | —       |
| 1:30 – 4:00   | **M1 — Universe initialization** (`npm run relic -- init`)       | CLI     |
| 4:00 – 7:30   | **M2/M3 — Routing + Codec pipeline** (`npm run relic -- demo`)   | CLI     |
| 7:30 – 10:00  | **M4 — Chaos engineering / rerouting** (`npm run relic -- send`) | CLI     |
| 10:00 – 12:30 | **Phase 2 — Chimera Co-Pilot via browser DevTools**              | Browser |
| 12:30 – 15:00 | **Architecture deep-dive + open Q&A**                            | Vault   |

---

## 🎤 SEGMENT 0 · Intro (0:00 – 1:30)

### Presenter says:

> "Stack Kings implements the **Relic Ring Protocol** — an interplanetary message-routing system.
> It has two phases:
> - **Phase 1 (Relic):** Pure physics engine. Dijkstra over a solar-system graph.
> - **Phase 2 (Chimera):** AI Co-Pilot that overlays live congestion, trust, and targeting risk on top of the physics path.
>
> The UI is a Next.js dashboard. The CLI tool (`npm run relic`) is the developer demo harness for milestones M1–M4.
> Let's start."

### Show in editor:

Open [`src/lib/relic/index.ts`](src/lib/relic/index.ts) to show public barrel, then flip to the vault:
```
obsidian-vault/00 - Index.md  →  scroll to "🔗 Key Data Flow"
```

---

## 🪐 SEGMENT 1 · M1: Universe Initialization (1:30 – 4:00)

### Command:

```bash
npm run relic -- init
```

### Expected output (example):

```
== M1: Universe Initialization ==
System: Sol-Relic-7
Planets: A(base 5), B(base 8), C(base 12), D(base 16), E(base 7), F(base 3)
Reachable links (L <= Lmax):
  A <-> B   L = 1.20M km
  A <-> C   L = 2.30M km
  B <-> D   L = 1.80M km
  ...
```

### Explain while output is on screen:

| Concept | Where it lives |
|---------|---------------|
| `PlanetNode` struct | [`src/lib/relic/types.ts`](src/lib/relic/types.ts) — `id`, `codex` (number base), `radius_km`, `active_towers`, `atmosphere_thickness_km`, `refraction_index` |
| Universe loader | [`src/lib/relic/server/universe.ts`](src/lib/relic/server/universe.ts) — reads `UNIVERSE_CONFIG` env, parses JSON, builds `nodesById` map and edge list |
| Engine composition | [`src/lib/relic/engine.ts`](src/lib/relic/engine.ts) — `createEngine()` wires universe + geometry provider + codec into one `Engine` object |
| Lmax filter | Edges where `void_distance_km > max_void_hop_distance_km` are excluded — packets can't jump too far |

---

### 🙋 Q&A SLOT 1 (run during/after init output)

**Q: "What does the `codex` number mean on each planet?"**
> A: Each planet speaks a different number base — `codex: 5` means base-5. Every payload character is encoded into that base before crossing the void. See [`src/lib/relic/codec.ts`](src/lib/relic/codec.ts). The `RelicCodec.encodeToCodex()` converts ASCII byte values to their base-N representation using `byte.toString(base).toUpperCase()`.

**Q: "Why is Lmax a constraint? Can't we just route through any planet?"**
> A: Lmax models the maximum laser range of the ring-laser arrays — beyond it, the signal can't punch through the void. It's set in `universe_metadata.max_void_hop_distance_km`. The `reachable()` closure in [`router.ts`](src/lib/relic/router.ts#L255-L258) enforces it: `geometry.voidDistanceKm(a, b) <= lmax`.

**Q: "Where does the universe JSON come from?"**
> A: `UNIVERSE_CONFIG` environment variable, or a bundled fallback in `.env`. The loader in `server/universe.ts` is Node-only (not imported in browser/edge runtimes) — that's why it lives under `server/`.

---

## 📦 SEGMENT 2 · M2/M3: Routing + Codec Pipeline (4:00 – 7:30)

### Command:

```bash
npm run relic -- demo
```

### Expected output (example):

```
== M1: Universe Initialization ==
  ... (planets + links)

== M2/M3: A -> F ==
Route: A -> C -> F
Total latency: 14.823 ms
Breakdown: fiber=0.412 tower=0.300 atmosphere=3.211 void=10.900 (ms)
Delivered payload: "Hello world"
hop_log:
  [0] A base5 tower 0->2 s=2 m=3 Tp=1.234 Tv=- cum=1.234 dialect=[242, 3042, ...]
  [1] C base12 tower 1->3 s=2 m=3 Tp=2.100 Tv=8.421 cum=11.755 dialect=[68, 65, ...]
  [2] F base3 tower 2->2 s=0 m=1 Tp=3.068 Tv=6.490 cum=21.313 dialect=[72, 65, ...]
```

### Walk through the pipeline live:

**Step 1 — Router finds path:**
```
src/lib/relic/router.ts  →  findShortestRoute()
```
- State space: `(planet_id, entry_tower)` — NOT just `planet_id`
- Why? Internal transit `Tp` depends on BOTH the tower you enter AND the tower you exit. Plain Dijkstra gets it wrong.
- MinHeap (lines 86–161): Custom binary min-heap. No external dependency.

**Step 2 — Latency formulas (point at output values):**

```
Tv = ((h1 × n1) + (h2 × n2) + L) / C    ← atmosphere_ms + void_ms
Tp = (2π × r × s) / (N × f × C) + m × Δt  ← fiber_ms + tower_ms
```
> Open `obsidian-vault/relic-latency.md` in Obsidian to show the formatted formula table.

**Step 3 — Codec pipeline per hop:**

```
ASCII bytes → encodeToCodex(bytes, next.codex)
           → serializeToBinary(encoded)       [flat bit stream across the void]
           → deserializeFromBinary(stream)    [received at next planet]
           → decodeFromCodex(received)        [back to ASCII]
```
> Show in [`src/lib/relic/codec.ts`](src/lib/relic/codec.ts): `RelicCodec.serializeToBinary()` — digit strings joined by space, each char emitted as 8-bit group. Fully reversible because codex digits are `[0-9A-Z]` and space is the only delimiter.

**Step 4 — Transmission orchestrator:**
> [`src/lib/relic/transmission.ts`](src/lib/relic/transmission.ts) — `transmit()` calls `findShortestRoute`, then walks hops, runs the codec pipeline, accumulates `hop_log`.

---

### 🙋 Q&A SLOT 2

**Q: "Why is the state space `(planet, entry_tower)` rather than just `planet`?"**
> A: Because `Tp` (internal crust transit) is a turn-dependent cost — the fiber arc you traverse depends on WHERE you entered from (the incoming hop's destination tower) AND where you're leaving to (the outgoing hop's origin tower). A plain node-weighted Dijkstra would either under-count or over-count this. See the comment block at the top of [`router.ts`](src/lib/relic/router.ts#L1-L14) and the vault doc `relic-router.md → "State Space Expansion"`.

**Q: "What happens if `Tp` or `Tv` gets called with an invalid tower index?"**
> A: `assertTowerInRange()` in [`latency.ts`](src/lib/relic/latency.ts#L51-L61) throws a `RangeError`. The router only ever passes tower indices it got from `geometry.closestTowerPair()`, so this is a defensive guard.

**Q: "Is the binary stream literally transmitted, or is it symbolic?"**
> A: Symbolic — it's a deterministic simulation. The round-trip `encode → serialize → deserialize → decode` proves codec integrity: `delivered_payload` must equal the original string, or the codec has a bug. The hop_log records each planet's dialect as proof of the conversion sequence.

**Q: "What is `combineRouteLatency` doing?"**
> A: It's the final aggregation: sum all `Tp` internal latencies (fiber + tower components) and all `Tv` void latencies (atmosphere + void components) into a single `LatencyBreakdown`. See [`latency.ts`](src/lib/relic/latency.ts#L133-L149).

---

## 💥 SEGMENT 3 · M4: Chaos Engineering (7:30 – 10:00)

### Command (kill an intermediate node manually):

```bash
# Normal delivery
npm run relic -- send A F "Test payload"

# Kill node B, cut link A-C
npm run relic -- send A F "Test payload" --kill B --cut A-C
```

### What to observe:
- First run: optimal path, e.g. `A -> B -> F`
- Second run: rerouted path avoiding B and link A-C, e.g. `A -> D -> E -> F`
- If no alternate path exists: `UNDELIVERABLE: No route from "A" to "F" within Lmax`

### Explain `ResilientNetwork`:

> Open [`src/lib/relic/resilience.ts`](src/lib/relic/resilience.ts):

```
ResilientNetwork
  .killNode(id)   → adds to failedNodes Set
  .killLink(a, b) → adds edgeKey to failedLinks Map
  .send(...)      → calls transmit() with current failure sets as RouteOptions
  .reviveNode(id) → removes from failedNodes
```

> The CLI's `--kill` and `--cut` flags parse into `blockedNodes` / `blockedEdges` passed directly to `transmit()`. In the demo mode (`npm run relic`), `engine.network.killNode(intermediate)` simulates a node going offline mid-demo.

### Show the `edgeKey` function in [`graph.ts`](src/lib/relic/graph.ts):
> Normalized to `A|B` (alphabetical) so `killLink("B","A")` === `killLink("A","B")`.

---

### 🙋 Q&A SLOT 3

**Q: "Does rerouting happen instantly or does a packet in transit get dropped?"**
> A: Routes are recomputed on every `send()` call against the CURRENT failure set. There's no in-flight state — each `transmit()` is a fresh Dijkstra run. In a real system, in-flight packets would be dropped; here the simulation just picks the best available route at request time.

**Q: "What happens if you kill the origin or destination?"**
> A: `findShortestRoute` checks `blockedNodes.has(originId)` and `blockedNodes.has(destinationId)` before even starting Dijkstra and returns `undeliverable` immediately with a descriptive message. See [`router.ts`](src/lib/relic/router.ts#L225-L234).

**Q: "Can you kill ALL nodes and still get routing?"**
> A: Only trivial self-route (origin === destination) works then. Everything else returns `UNDELIVERABLE: No route within Lmax`. Try it:
> ```bash
> npm run relic -- send A F "payload" --kill B,C,D,E
> ```

---

## 🤖 SEGMENT 4 · Phase 2: Chimera Co-Pilot via Browser DevTools (10:00 – 12:30)

### Setup: Open the running dev server

```
http://localhost:3000
```

Open **DevTools → Network tab** (filter: `Fetch/XHR`).

---

### Demo A: Structured routing (True Cost Router)

Paste into DevTools Console:

```javascript
const res = await fetch('/api/route', {
  method: 'POST',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify({ origin_id: 'A', destination_id: 'F' })
});
const data = await res.json();
console.log(JSON.stringify(data, null, 2));
```

**What to point out in the response:**
- `path` — chosen route after True Cost scoring
- `link_evaluations[]` — one per hop:
  - `predicted_congestion_penalty_ms` — congestion model output
  - `trust_score` — trust model output
  - `targeting_risk_score` — targeting risk model output
  - `combined_cost` — composite weight used during Dijkstra
- `explanation` — human-readable narrative from `routing-report.ts`

> Open `obsidian-vault/chimera-router-true-cost.md` — walk through the **True Cost formula**:
> ```
> combined_cost = physicsVoidMs
>               + penalty_ms
>               + (1 − trustScore) × TRUST_SCALE_MS
>               + targetingRisk  × TARGETING_SCALE_MS
>               − traffic_share × ENTROPY_BONUS_SCALE_MS
> ```

---

### Demo B: Natural-language routing (Co-Pilot Agent)

```javascript
const res = await fetch('/api/route', {
  method: 'POST',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify({ request: 'send from A to F as fast as possible' })
});
const data = await res.json();
console.log(JSON.stringify(data, null, 2));
```

**Walk through the Co-Pilot flow** (`obsidian-vault/chimera-agent-copilot.md`):

```
1. parseRoutingRequest(nlRequest)   → { origin_id, destination_id, source: "rules"|"llm" }
2. findShortestRoute(...)           → baseline physics path
3. FOR EACH HOP (sequential):
     a. chimeraClient.getLinkState(linkId)  → live telemetry
     b. evaluateLink(state, physicsVoidMs)  → { blocked, trust, congestion, targeting }
     c. if blocked → add to blockedEdges, reroute from current node
4. Build link_evaluations[], compute final_latency_estimate_ms
5. Return Phase2RoutingReport
```

**Key difference from True Cost Router:**

| | Co-Pilot Agent | True Cost Router |
|-|----------------|-----------------|
| Evaluation | Sequential, hop-by-hop | Batch, global Dijkstra with surcharges |
| Input | NL string | Structured `{origin_id, destination_id}` |
| Rerouting | Block + re-route from current node | Diversification (5% tolerance) |
| LLM | Yes (fallback parser) | No |

---

### Demo C: Anomaly Detection

```javascript
// POST /api/transmit to see full hop_log with codec pipeline
const res = await fetch('/api/transmit', {
  method: 'POST',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify({ origin_id: 'A', destination_id: 'F', payload: 'Hello world' })
});
const data = await res.json();
console.log(data.packet.hop_log);
```

> Point at `payload_dialect` in each hop_log entry — proves the codec ran on every hop.

---

### 🙋 Q&A SLOT 4

**Q: "How does the link anomaly detection work?"**
> A: `detectLinkAnomaly()` in [`link-evaluation.ts`](src/lib/relic/chimera/../../../lib/chimera/link-evaluation.ts) validates every numeric field against expected ranges (`load_ratio ∈ [0,1]`, `capacity_units > 0`, etc.). Out-of-distribution values are clamped and flagged. Anomalous links get conservative scores: `trust = min(trust, 0.3)`, `targetingRisk = max(risk, 0.85)` — but they are **NOT hard-blocked** on trust alone (anomalous ≠ known bad). See vault doc `chimera-link-evaluation.md → "Anomaly Routing Policy"`.

**Q: "What is `traffic_share × ENTROPY_BONUS_SCALE_MS` doing in the combined_cost formula?"**
> A: It's entropy maximisation / route diversification. Links with higher traffic share get a cost *reduction*, steering packets away from concentrating on a single path — maximising entropy across the network. This is why the True Cost Router can pick a slightly-higher-latency path if it lowers aggregate targeting risk.

**Q: "What's the difference between `blockedNodes` and a failed link?"**
> A: `blockedNodes` removes an entire planet from the graph — no traffic can enter or leave it. A `blockedEdge` only severs the direct link between two specific planets; both planets remain reachable via other paths. The Co-Pilot uses `blockedEdges` for its sequential hop blocking; `blockedNodes` is a harder failure used by `ResilientNetwork.killNode()`.

---

## 🏗️ SEGMENT 5 · Architecture Deep-Dive + Open Q&A (12:30 – 15:00)

### Live vault navigation — open these in Obsidian:

1. **`00 - Index.md`** → "🔗 Key Data Flow" diagram  
2. **`relic-router.md`** → State Space Expansion, MinHeap note  
3. **`chimera-link-evaluation.md`** → Scoring pipeline table  
4. **`chimera-router-true-cost.md`** → True Cost vs Co-Pilot comparison table  
5. **`api-rate-limit.md`** → Rate limiting implementation  

### Full `src/lib` component map:

```
src/lib/
├── relic/                      ← Phase 1: Pure physics engine
│   ├── types.ts                   PlanetNode, Universe, Packet, HopLogEntry
│   ├── contracts.ts               GeometryProvider + Codec interfaces
│   ├── config.ts                  Universe JSON schema + env parsing
│   ├── graph.ts                   Edge list + edgeKey() normalization
│   ├── latency.ts                 Tv (void) + Tp (internal) formulas
│   ├── codec.ts                   RelicCodec: ASCII ↔ base-N ↔ binary
│   ├── router.ts                  Dijkstra over (planet, entry_tower) states
│   ├── transmission.ts            End-to-end orchestration + hop_log
│   ├── resilience.ts              ResilientNetwork: stateful failure tracking
│   ├── engine.ts                  createEngine() composition root
│   └── server/universe.ts         Node-only universe JSON loader
│
├── chimera/                    ← Phase 2: AI Co-Pilot
│   ├── constants.ts               Scoring weights (TRUST_SCALE_MS etc.)
│   ├── types.ts                   Phase2RoutingReport, LinkEvaluation
│   ├── client.ts                  ChimeraClient: HTTP tick cache + getLinkState()
│   ├── health.ts                  /chimera/health probe
│   ├── link-id.ts                 canonicalLinkId() normalization
│   ├── link-evaluation.ts         Scoring hub: anomaly detect + 3 models + combined_cost
│   ├── report-schema.ts           Zod schema for Phase2RoutingReport
│   ├── routing-report.ts          buildRoutingReport() + helpers
│   ├── models/
│   │   ├── congestion.ts          predictCongestion(): penalty_ms + is_saturated
│   │   ├── trust.ts               scoreTrust(): trust_score
│   │   └── targeting.ts           scoreTargetingRisk(): targeting_risk_score
│   ├── parser/
│   │   ├── hybrid.ts              parseRoutingRequest(): rules → LLM fallback
│   │   └── llm.ts                 LLM-backed NL parser
│   ├── router/true-cost-router.ts routeWithTrueCost(): batch Dijkstra + diversification
│   └── agent/copilot.ts           routeWithCopilot(): sequential Co-Pilot flow
│
└── api/                        ← Shared HTTP utilities
    ├── constants.ts               API_RATE_LIMIT, headers
    ├── errors.ts                  ApiError class + HTTP error helpers
    ├── rate-limit.ts              Sliding-window rate limiter (Map-based)
    ├── validate-route.ts          Zod schema for POST /api/route
    └── validate-transmit.ts       Zod schema for POST /api/transmit
```

---

### 🙋 OPEN Q&A (12:30 – 15:00)

**Q: "Run the unit test suite — how much is covered?"**

```bash
npm test
# or with coverage:
npm run test:coverage
```

> Tests live alongside source files: `router.test.ts`, `codec.test.ts`, `latency.test.ts`, `transmission.test.ts`, `resilience.test.ts`, `config.test.ts`, `graph.test.ts`.

**Q: "How is the API protected from abuse?"**
> A: Sliding-window rate limiter in [`src/lib/api/rate-limit.ts`](src/lib/relic/../api/rate-limit.ts) — keyed by IP. Max N requests per window. Applied in the Next.js route handlers at `src/app/api/`. Input validated with Zod schemas (`validate-route.ts`, `validate-transmit.ts`).

**Q: "Walk me through the full request lifecycle for `POST /api/transmit`"**
> A:
> 1. `src/app/api/transmit/route.ts` → rate-limit check → Zod validate
> 2. `transmit(universe, geometry, codec, origin, destination, payload)` in [`transmission.ts`](src/lib/relic/transmission.ts)
> 3. `findShortestRoute(...)` → Dijkstra
> 4. Codec pipeline loop over hops
> 5. Build `hop_log`
> 6. Return `{ packet, route, delivered_payload }` as JSON

**Q: "What does the Sentry integration add?"**
> A: Error boundary components in `src/components/observability/` catch React errors and report to Sentry. Server config in `sentry.server.config.ts` + `sentry.edge.config.ts`. The `/api/debug` route intentionally throws to test the integration.

**Q: "How is the CLI invoked exactly?"**
> A: `npm run relic` maps to `tsx src/cli/relic.ts` in `package.json`. `tsx` is the TypeScript execution engine — no compile step needed. Arguments after `--` are forwarded: `npm run relic -- send A F "payload" --kill B`.

---

## 📎 Quick Reference Card

### CLI Commands

```bash
npm run relic                         # Full M1–M4 scripted demo
npm run relic -- init                 # M1: Print universe
npm run relic -- send A F "msg"       # M2/M3: Route + deliver
npm run relic -- send A F "msg" --kill B          # M4: Kill node B
npm run relic -- send A F "msg" --cut A-C,B-D    # M4: Cut links
npm test                              # Vitest unit suite
npm run test:coverage                 # Coverage report
npm run typecheck                     # TypeScript check
```

### Browser DevTools Snippets

```javascript
// Health check
await fetch('/api/health').then(r => r.json())

// Universe graph
await fetch('/api/universe').then(r => r.json())

// Structured routing (True Cost Router)
await fetch('/api/route', {
  method:'POST', headers:{'Content-Type':'application/json'},
  body: JSON.stringify({ origin_id:'A', destination_id:'F' })
}).then(r => r.json())

// NL routing (Co-Pilot)
await fetch('/api/route', {
  method:'POST', headers:{'Content-Type':'application/json'},
  body: JSON.stringify({ request:'send from A to F securely' })
}).then(r => r.json())

// Full transmission with hop_log
await fetch('/api/transmit', {
  method:'POST', headers:{'Content-Type':'application/json'},
  body: JSON.stringify({ origin_id:'A', destination_id:'F', payload:'Hello world' })
}).then(r => r.json())
```

### Key Algorithm Reference

| Algorithm | File | Key function |
|-----------|------|-------------|
| Dijkstra (expanded state) | `src/lib/relic/router.ts` | `findShortestRoute()` |
| Void latency `Tv` | `src/lib/relic/latency.ts` | `computeVoidLatency()` |
| Internal latency `Tp` | `src/lib/relic/latency.ts` | `computeInternalLatency()` |
| Codec: ASCII ↔ base-N | `src/lib/relic/codec.ts` | `RelicCodec` class |
| Binary serialization | `src/lib/relic/codec.ts` | `serializeToBinary()` |
| Chaos rerouting | `src/lib/relic/resilience.ts` | `ResilientNetwork` |
| Link scoring | `src/lib/chimera/link-evaluation.ts` | `evaluateLink()` |
| Anomaly detection | `src/lib/chimera/link-evaluation.ts` | `detectLinkAnomaly()` |
| True Cost routing | `src/lib/chimera/router/true-cost-router.ts` | `routeWithTrueCost()` |
| Co-Pilot orchestration | `src/lib/chimera/agent/copilot.ts` | `routeWithCopilot()` |
| NL parsing | `src/lib/chimera/parser/hybrid.ts` | `parseRoutingRequest()` |

### Obsidian Vault Quick Links

| Topic | Vault file |
|-------|-----------|
| Full architecture | `00 - Index.md` |
| Router algorithm | `relic-router.md` |
| Latency formulas | `relic-latency.md` |
| Codec internals | `relic-codec.md` |
| Link scoring formula | `chimera-link-evaluation.md` |
| True Cost vs Co-Pilot | `chimera-router-true-cost.md` |
| Co-Pilot flow | `chimera-agent-copilot.md` |
| API rate limiting | `api-rate-limit.md` |
| CLI reference | `cli-relic.md` |

---

*Generated for Stack Kings · Relic Ring Protocol · Demo v1.0 · 2026-07-09*
