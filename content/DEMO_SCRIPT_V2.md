# 🌌 Stack Kings · Relic Ring Protocol
## Live Demo Script — 15 Minutes + Parallel Q&A
### Version 2 — DevTools-First Edition

> **Primary tool:** Browser DevTools Console (F12) at `http://localhost:3000`  
> **Secondary tool:** Terminal for milestone verification only  
> **Prereq:** `npm run dev` already running · Obsidian vault open at `obsidian-vault/00 - Index.md`

---

## 🗺️ Before You Start — Open These Side by Side

| Window | What to open |
|--------|-------------|
| **Browser tab 1** | `http://localhost:3000` — live dashboard |
| **Browser tab 2** | DevTools Console (F12) ready to paste snippets |
| **Editor** | `obsidian-vault/00 - Index.md` |
| **Terminal** | Already running `npm run dev` |

---

## ⏱️ Run Sheet

```
0:00  Block 1 — Intro + Architecture tour          (2 min)
2:00  Block 2 — Universe graph via DevTools         (3 min)  ← DevTools
5:00  Block 3 — Routing + Codec pipeline            (4 min)  ← DevTools + terminal verify
9:00  Block 4 — Chaos / Rerouting                   (3 min)  ← DevTools
12:00 Block 5 — Chimera Co-Pilot                    (2 min)  ← DevTools
14:00 Block 6 — Open Q&A                            (1 min)
```

---

---

# BLOCK 1 — Intro + Architecture Tour `0:00 – 2:00`

## 🔵 SHOW (no code yet)

Open `obsidian-vault/00 - Index.md`. Scroll to **"🔗 Key Data Flow"** and read aloud:

```
POST /api/route (NL)
  └─→ Chimera Co-Pilot
        ├─→ Parser          (NL → origin/destination)
        ├─→ Relic Router    (Dijkstra physics path)
        ├─→ Link Evaluation (score each hop)
        └─→ Report Schema   (validate output)
```

Say:
> "Two phases. **Phase 1 (Relic):** pure physics — Dijkstra over a solar-system graph.
> **Phase 2 (Chimera):** AI Co-Pilot layer — overlays congestion, trust, and targeting risk.
> The browser is our main interface. Let's start by inspecting the live universe."

---

---

# BLOCK 2 — Universe Graph via DevTools `2:00 – 5:00`

## 🔵 SHOW (open in editor)

File: [`src/lib/relic/types.ts`](src/lib/relic/types.ts)  
Point out the `PlanetNode` shape:
```ts
// Each planet has:
id: string               // "Aegis", "Boreas" …
codex: number            // the number base (8, 5, 6 …)
radius_km: number
active_towers: number
atmosphere_thickness_km: number
refraction_index: number
```

Then open [`src/lib/relic/engine.ts`](src/lib/relic/engine.ts) to show `createEngine()` — the one place where everything wires together (geometry + codec + graph + resilient network).

---

## 🟢 RUN — DevTools Console (paste and run)

### Step 1 — Health check
```javascript
// Paste into DevTools Console
const health = await fetch('/api/health').then(r => r.json());
console.log(health);
```

Expected: `{ status: "ok", ... }`

---

### Step 2 — Fetch the full universe graph
```javascript
const universe = await fetch('/api/universe').then(r => r.json());
console.table(universe.nodes.map(n => ({
  id: n.id,
  codex: `base ${n.codex}`,
  towers: n.active_towers,
  radius_km: n.radius_km
})));
```

**Real output from Zeta-26:**

| id | codex | towers | radius_km |
|----|-------|--------|-----------|
| Aegis | base 8 | — | — |
| Boreas | base 5 | — | — |
| Dawn | base 6 | — | — |
| Elysium | base 10 | — | — |
| Fenix | base 16 | — | — |
| Caelum | base 14 | — | — |

---

### Step 3 — Inspect reachable links (Lmax filter)
```javascript
// Show only links within laser range (within_lmax = true)
const reachable = universe.edges?.filter(e => e.within_lmax) 
  ?? universe.interplanetaryLinks;
console.table(reachable.map(e => ({
  link: `${e.from ?? e.planet_a} ↔ ${e.to ?? e.planet_b}`,
  distance_Mkm: ((e.void_distance_km ?? 0) / 1_000_000).toFixed(2) + 'M km'
})));
```

**Real Zeta-26 links (from `npm run relic -- init`):**

| Link | Distance |
|------|----------|
| Aegis ↔ Boreas | 18.02M km |
| Aegis ↔ Dawn | 35.35M km |
| Aegis ↔ Elysium | 46.08M km |
| Boreas ↔ Dawn | 20.61M km |
| Boreas ↔ Elysium | 29.14M km |
| Boreas ↔ Fenix | 40.31M km |
| Dawn ↔ Elysium | 30.41M km |
| Dawn ↔ Fenix | 21.21M km |
| Dawn ↔ Caelum | 33.48M km |
| Elysium ↔ Fenix | 49.24M km |
| Elysium ↔ Caelum | 38.01M km |
| Fenix ↔ Caelum | 33.48M km |

> Point at the list: *"Aegis-Boreas is the shortest hop at 18M km. There is no direct Aegis–Caelum link — that distance exceeds Lmax, so packets must route via intermediaries."*

---

## 🙋 Q&A after Block 2

**Q: What is `codex` on each planet?**
> Each planet speaks a different number base. `codex: 5` → base-5. Every ASCII character in the payload is encoded into that base before crossing the void. See `src/lib/relic/codec.ts` — `encodeToCodex()` does `byte.toString(base).toUpperCase()`.

**Q: Why is Lmax a hard constraint?**
> It models the physical laser range limit. Beyond Lmax the signal can't punch through the void. Enforced in `graph.ts → buildNetworkGraph()`: `voidDistanceKm <= lmax`. Packets cannot jump over unreachable gaps — they need intermediate planets.

**Q: What is the `edgeKey()` function in graph.ts?**
> It creates a normalized, alphabetically-sorted string key for undirected pairs: `A|B` not `B|A`. Used by `ResilientNetwork` to deduplicate link failure records regardless of direction.

---

---

# BLOCK 3 — Routing + Codec Pipeline `5:00 – 9:00`

## 🔵 SHOW (open in editor)

Open [`src/lib/relic/router.ts`](src/lib/relic/router.ts) top comment (lines 1–14):

```ts
// Key insight: Tp (internal crust transit) depends on BOTH
// the entry tower (set by the incoming hop) AND
// the exit tower (set by the outgoing hop).
// Plain Dijkstra over (planet) nodes would be WRONG.
// We expand the state space to (planet, entry_tower).
//
// stateKey = "${planetId}#${entryTower}"
```

Then open [`src/lib/relic/latency.ts`](src/lib/relic/latency.ts) and show the two formulas:

```
Void travel (Tv):   ((h1×n1) + (h2×n2) + L) / C
Internal transit (Tp): (2π×r×s)/(N×f×C) + m×Δt
                        ↑ fiber arc          ↑ tower processing
```

---

## 🟢 RUN — DevTools Console

### Step 1 — Baseline physics route (Aegis → Caelum)
```javascript
const result = await fetch('/api/transmit', {
  method: 'POST',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify({
    origin: 'Aegis',
    destination: 'Caelum', 
    payload: 'Hello world'
  })
}).then(r => r.json());

// Print the route summary
const { route, delivered_payload } = result;
console.log('Path:', route.path.join(' → '));
console.log('Total latency:', route.total_latency_ms.toFixed(3), 'ms');
console.log('Breakdown:', route.breakdown);
console.log('Delivered payload:', delivered_payload);
```

**Real output from Zeta-26:**
```
Path:           Aegis → Dawn → Caelum
Total latency:  229495.168 ms
Breakdown:
  fiber_ms:        23.445
  tower_ms:        42.000
  atmosphere_ms:    2.804
  void_ms:      229426.919
Delivered payload: "Hello world"
```

> *"Total ~229 seconds. Almost all of it is void travel — light-speed across 35M+ km of empty space. The fiber and tower overhead is tiny by comparison."*

---

### Step 2 — Inspect the hop_log (codec pipeline proof)
```javascript
// Show each hop's dialect encoding
result.packet.hop_log.forEach(hop => {
  console.group(`[${hop.sequence}] ${hop.planet_id} (base ${hop.codex})`);
  console.log('Entry→Exit tower:', hop.entry_tower, '→', hop.exit_tower);
  console.log('Segments (s):', hop.segments, '| Towers hit (m):', hop.towers_hit);
  console.log('Internal Tp:', hop.internal_latency_ms?.toFixed(3), 'ms');
  console.log('Void Tv:    ', hop.void_latency_ms?.toFixed(3) ?? '—', 'ms');
  console.log('Cumulative: ', hop.cumulative_latency_ms.toFixed(3), 'ms');
  console.log('Dialect digits:', hop.payload_dialect.digits.slice(0,5), '...');
  console.groupEnd();
});
```

**Real hop_log from Zeta-26:**

```
[0] Aegis (base 8)
    Entry→Exit tower: 2 → 2    (s=0, m=1 — no fiber traversal)
    Internal Tp:  7.000 ms
    Void Tv:     117824.895 ms
    Cumulative:  117831.895 ms
    Dialect:     [110, 145, 154, 154, 157, ...]   ← "Hello world" in base-8

[1] Dawn (base 6)
    Entry→Exit tower: 4 → 1    (s=3, m=4 — crosses 3 ring segments)
    Internal Tp:  51.445 ms
    Void Tv:     111604.828 ms
    Cumulative:  229488.168 ms
    Dialect:     [200, 245, 300, 300, 303, ...]   ← re-encoded in base-6

[2] Caelum (base 14)
    Entry→Exit tower: 11 → 11  (s=0, m=1 — destination, no transit)
    Internal Tp:  7.000 ms
    Void Tv:     — ms
    Cumulative:  229495.168 ms
    Dialect:     [52, 73, 7A, 7A, 7D, ...]        ← re-encoded in base-14
```

> *"Each hop proves the codec ran. The same ASCII payload — 'Hello world' — arrives in different base representations at each planet. The round-trip proves integrity: `delivered_payload` must equal the original string."*

---

### Step 3 — Show the codec pipeline in the source
Open [`src/lib/relic/codec.ts`](src/lib/relic/codec.ts) and walk through:

```ts
// Per hop across the void:
const encoded = codec.encodeToCodex(asciiBytes, nextNode.codex);  // ASCII → base-N digits
const stream  = codec.serializeToBinary(encoded);                  // digits → flat bit stream
const received = codec.deserializeFromBinary(stream, nextNode.codex); // bit stream → digits
const decoded  = codec.decodeFromCodex(received);                  // digits → ASCII bytes
```

> *"The binary stream is what physically crosses the void. Digit strings are joined by a space delimiter, then each character is emitted as an 8-bit group. Because codex digits are [0-9A-Z], the space is an unambiguous separator — the stream is fully reversible."*

Also open [`src/lib/relic/transmission.ts`](src/lib/relic/transmission.ts) — `transmit()` is the orchestrator that calls `findShortestRoute`, then runs this codec loop, then builds the full `hop_log`.

---

## 🙋 Q&A after Block 3

**Q: Why is the Dijkstra state `(planet, entry_tower)` and not just `planet`?**
> `Tp` (internal crust transit) is a *turn-dependent* cost — the fiber arc you traverse depends on which tower the packet entered from AND which tower it exits to. A plain node-weighted Dijkstra ignores this and charges the wrong `Tp`. The expanded state space `(planet, entry_tower)` ensures the correct tower pair is known at relaxation time. See `router.ts` line 294: `effectiveEntry = entryTower === NO_ENTRY ? exitTower : entryTower`.

**Q: Why does Aegis charge `s=0, m=1` but Dawn charges `s=3, m=4`?**
> At Aegis (origin), the packet exits from the same tower it's sending from — no fiber traversal. At Dawn (transit), the packet arrives at tower 4 but must exit from tower 1 to line up for the Dawn→Caelum hop. 3 ring segments → 3 fiber sections + 1 entry tower = 4 towers hit. At Caelum (destination) it's `s=0` again because it terminates there.

**Q: What is `combineRouteLatency()` doing?**
> It's the final aggregation in `latency.ts`: sum all `Tp` values (fiber + tower components) and all `Tv` values (atmosphere + void components) into a single `LatencyBreakdown`. Gives the four-component breakdown you see in the output.

**Q: Is the binary stream actually transmitted?**
> It's a deterministic simulation. The round-trip `encode → serialize → deserialize → decode` is a proof of correctness — `delivered_payload` must equal the original string or the codec has a bug. In `transmission.ts` the hop_log captures both the binary stream *and* the dialect at each planet as evidence.

---

---

# BLOCK 4 — Chaos Engineering / Dynamic Rerouting `9:00 – 12:00`

## 🔵 SHOW (open in editor)

Open [`src/lib/relic/resilience.ts`](src/lib/relic/resilience.ts):

```ts
class ResilientNetwork {
  killNode(id)        // marks planet offline → blockedNodes Set
  killLink(a, b)      // severs direct link → blockedEdges Map
  reviveNode(id)      // brings planet back
  reviveLink(a, b)    // restores link
  send(from, to, msg) // Dijkstra runs fresh against CURRENT failure set
}
```

> *"There is no in-flight packet state. Every `send()` is a fresh Dijkstra run against whatever failures are currently active. Instant rerouting."*

---

## 🟢 RUN — DevTools Console

### Step 1 — Route with Dawn killed (M4 chaos scenario)
```javascript
// Simulate Dawn going offline
const chaos = await fetch('/api/transmit', {
  method: 'POST',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify({
    origin_id: 'Aegis',
    destination_id: 'Caelum',
    payload: 'Hello world',
    options: { blockedNodes: ['Dawn'] }   // kill Dawn
  })
}).then(r => r.json());

console.log('Rerouted path:', chaos.route.path.join(' → '));
console.log('New latency:  ', chaos.route.total_latency_ms.toFixed(3), 'ms');
console.log('Breakdown:', chaos.route.breakdown);
```

**Real output (from `npm run relic -- demo` chaos segment):**
```
Normal path:    Aegis → Dawn → Caelum      (229,495 ms)
Rerouted path:  Aegis → Elysium → Caelum  (280,423 ms)
New latency:    280423.074 ms
Breakdown:
  fiber_ms:        47.288
  tower_ms:        42.000
  atmosphere_ms:    4.577
  void_ms:      280329.209
```

> *"+51 seconds extra. The rerouted path is longer in void distance — Aegis→Elysium is 46M km vs Aegis→Dawn's 35M km — but it's the best available when Dawn is down."*

---

### Step 2 — Cut a link instead of killing a node
```javascript
// Sever the Aegis-Dawn link but leave both planets alive
const cutLink = await fetch('/api/transmit', {
  method: 'POST',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify({
    origin_id: 'Aegis',
    destination_id: 'Caelum',
    payload: 'Hello world',
    options: { blockedEdges: [['Aegis', 'Dawn']] }
  })
}).then(r => r.json());

console.log('Link-cut path:', cutLink.route.path.join(' → '));
```

---

### Step 3 — Force undeliverable (isolate Caelum)
```javascript
// Kill every neighbor of Caelum
const isolated = await fetch('/api/transmit', {
  method: 'POST',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify({
    origin_id: 'Aegis',
    destination_id: 'Caelum',
    payload: 'Hello world',
    options: { blockedNodes: ['Dawn', 'Elysium', 'Fenix'] }
  })
}).then(r => r.json());

console.log('Status:', isolated.packet.status);
console.log('Reason:', isolated.packet.undeliverable_reason);
```

Expected: `status: "undeliverable"`, reason: `"No route from Aegis to Caelum within Lmax"`

---

## ✅ TERMINAL VERIFY — (optional, only if audience wants to see CLI)
```bash
# Cross-check: CLI matches DevTools output
npm run relic -- demo
# Shows: M4 chaos - killed Dawn → Aegis → Elysium → Caelum → 280423.074 ms ✓
```

---

## 🙋 Q&A after Block 4

**Q: Why does killing Dawn force Elysium as the next hop?**
> Aegis's only other reachable neighbors (L ≤ Lmax) are Boreas and Elysium. From Boreas you can reach Fenix and Caelum — the router tries all paths. But Aegis→Elysium→Caelum has lower total latency than Aegis→Boreas→anything→Caelum, so Dijkstra picks it. See the real link table: Elysium↔Caelum = 38.01M km, Boreas→Caelum has no direct link so it needs an extra hop.

**Q: What is `blockedNodes` vs `blockedEdges`?**
> `blockedNodes` removes an entire planet — no traffic can enter or leave it at all. `blockedEdges` only severs a specific pair — both planets remain alive and reachable via other hops. The router checks both in `reachable()` at `router.ts` line 255–259.

**Q: What if you kill the origin or destination itself?**
> `findShortestRoute()` checks `blockedNodes.has(originId)` and `blockedNodes.has(destinationId)` before starting Dijkstra and returns `undeliverable` immediately with `"Origin is offline"` / `"Destination is offline"`. Try: `options: { blockedNodes: ['Aegis'] }`.

**Q: How does the CLI `--kill` flag work?**
> `npm run relic -- send Aegis Caelum "msg" --kill Dawn` parses into `blockedNodes: ['Dawn']` in `src/cli/relic.ts`, passed directly as `RouteOptions` to `transmit()`. Same code path as the DevTools snippet above.

---

---

# BLOCK 5 — Chimera Co-Pilot `12:00 – 14:00`

## 🔵 SHOW (open in editor)

Open `obsidian-vault/chimera-router-true-cost.md` — read the comparison table:

| | **True Cost Router** | **Co-Pilot Agent** |
|--|--|--|
| Evaluation | Batch — global Dijkstra with surcharges | Sequential — hop-by-hop |
| Input | Structured `{origin_id, destination_id}` | NL string |
| Rerouting | Diversification (5% tolerance) | Block + re-route from current node |
| LLM | No | Yes (fallback parser) |

Then open [`src/lib/chimera/agent/copilot.ts`](src/lib/chimera/agent/copilot.ts) — read the sequential flow comment at the top.

---

## 🟢 RUN — DevTools Console

### Step 1 — True Cost Router (structured input)
```javascript
const trueCost = await fetch('/api/route', {
  method: 'POST',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify({
    origin_id: 'Aegis',
    destination_id: 'Caelum'
  })
}).then(r => r.json());

console.log('Chosen path:', trueCost.path?.join(' → '));
console.log('Explanation:', trueCost.explanation);

// Inspect link scores
trueCost.link_evaluations?.forEach(ev => {
  console.log(`\n${ev.link_id}`);
  console.log('  congestion penalty:', ev.predicted_congestion_penalty_ms, 'ms');
  console.log('  trust score:       ', ev.trust_score);
  console.log('  targeting risk:    ', ev.targeting_risk_score);
  console.log('  combined_cost:     ', ev.combined_cost, 'ms');
});
```

> **Point at `combined_cost` formula:**
> ```
> combined_cost = physicsVoidMs
>               + congestion_penalty_ms
>               + (1 − trust_score)    × TRUST_SCALE_MS
>               + targeting_risk_score × TARGETING_SCALE_MS
>               − traffic_share        × ENTROPY_BONUS_SCALE_MS
>               ↑ lower trust = higher penalty (spoofed link costs more)
>               ↑ entropy bonus steers away from over-concentrated paths
> ```

---

### Step 2 — Co-Pilot Agent (natural language input)
```javascript
const copilot = await fetch('/api/route', {
  method: 'POST',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify({
    request: 'send a message from Aegis to Caelum as fast as possible'
  })
}).then(r => r.json());

console.log('NL parse source:', copilot.parse_source ?? '(check explanation)');
console.log('Chosen path:', copilot.path?.join(' → '));
console.log('Reroutes:', copilot.reroute_count ?? 0);
console.log('Explanation:', copilot.explanation);
```

> *"The NL parser (`parser/hybrid.ts`) first tries regex rules ('from X to Y'), then falls back to an LLM call if needed. The `parse_source` field tells you which path ran."*

---

### Step 3 — Anomaly detection in link scoring
```javascript
// Open chimera-link-evaluation.ts while explaining this
const eval1 = await fetch('/api/route', {
  method: 'POST',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify({ origin_id: 'Aegis', destination_id: 'Caelum' })
}).then(r => r.json());

const anomalousLinks = eval1.link_evaluations?.filter(e => e.anomaly_detected);
console.log('Anomalous links:', anomalousLinks?.map(e => e.link_id) ?? 'none');
```

Open [`src/lib/chimera/link-evaluation.ts`](src/lib/chimera/link-evaluation.ts) and explain `detectLinkAnomaly()`:
```
Checks: capacity_units > 0, load_ratio ∈ [0,1], traffic_share ∈ [0,1]
If OOD: clamp values, flag anomaly, apply conservative scores
  trust = min(trust, 0.3)      ← assume spoofing
  targeting = max(risk, 0.85)  ← assume high exposure
BUT: anomalous links are NOT hard-blocked — still usable as last resort
```

---

## 🙋 Q&A after Block 5

**Q: Why doesn't the Co-Pilot just use Dijkstra globally like the True Cost Router?**
> The Co-Pilot mimics a sequential decision-making agent — it evaluates one hop at a time and reroutes mid-path if it finds a bad link. This is intentional (per the challenge spec §4). The True Cost Router is batch-optimal but can't take an NL request.

**Q: What are the 3 sub-models doing?**
> `congestion.ts` — piecewise power-law regression `penalty = k × load_ratio^p` (fitted per-link from CSV). Hard-blocks if `load_ratio ≥ 0.90` or `status = "saturated"`.  
> `trust.ts` — compares self-reported latency vs expected physics `Tv + congestion`. Positive delta = link claiming to be faster than physics allows → spoofing signal. Known bad links use a historical `mean_delta_ms` ratio; novel attacks caught by a 15,000ms threshold.  
> `targeting.ts` — scores targeting risk exposure for each hop.

**Q: What does the entropy bonus do?**
> `traffic_share × ENTROPY_BONUS_SCALE_MS` reduces `combined_cost` for links with higher traffic share — counterintuitively steering *toward* busy links. This is entropy maximisation: spread traffic across the network rather than concentrating on one path. Also used for route diversification (5% True Cost tolerance) in `true-cost-router.ts`.

---

---

# BLOCK 6 — Open Q&A `14:00 – 15:00`

## Quick fire reference snippets for any live question

```javascript
// Check rate limiting behavior (hit it twice fast)
for (let i = 0; i < 3; i++) {
  const r = await fetch('/api/route', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({ origin_id: 'Aegis', destination_id: 'Caelum' })
  });
  console.log(`Request ${i+1}:`, r.status, r.statusText);
}
// Rate limiter in: src/lib/api/rate-limit.ts (sliding window, keyed by IP)

// Run unit tests (if projected on terminal)
// npm test
// npm run test:coverage

// Type-check the whole project
// npm run typecheck
```

---

## Full Component Map (for any "where does X live?" question)

```
src/lib/
├── relic/                           PHASE 1 — Physics Engine
│   ├── types.ts          ← PlanetNode, Universe, Packet, HopLogEntry types
│   ├── contracts.ts      ← GeometryProvider + Codec interfaces (DI boundaries)
│   ├── config.ts         ← Universe JSON schema + Zod validation
│   ├── graph.ts          ← buildNetworkGraph(), edgeKey(), VoidEdge
│   ├── latency.ts        ← computeVoidLatency(Tv), computeInternalLatency(Tp)
│   ├── codec.ts          ← RelicCodec: ASCII ↔ base-N ↔ binary stream
│   ├── router.ts         ← findShortestRoute() — Dijkstra over (planet, tower) states
│   ├── transmission.ts   ← transmit() — orchestrates router + codec + hop_log
│   ├── resilience.ts     ← ResilientNetwork — stateful failure tracking
│   ├── engine.ts         ← createEngine() — composition root (DI wiring)
│   └── server/universe.ts← Node-only universe JSON loader
│
├── chimera/                         PHASE 2 — AI Co-Pilot
│   ├── constants.ts      ← Scoring weights (TRUST_SCALE_MS, ENTROPY_BONUS_SCALE_MS …)
│   ├── types.ts          ← Phase2RoutingReport, LinkEvaluation types
│   ├── client.ts         ← ChimeraClient: HTTP tick cache + getLinkState()
│   ├── link-evaluation.ts← detectLinkAnomaly() + evaluateLink() — scoring hub
│   ├── report-schema.ts  ← Zod schema for Phase2RoutingReport
│   ├── routing-report.ts ← buildRoutingReport() + computePhysicsLatencyForPath()
│   ├── models/
│   │   ├── congestion.ts ← predictCongestion(): k × load_ratio^p
│   │   ├── trust.ts      ← scoreTrust(): physics vs self-reported delta
│   │   └── targeting.ts  ← scoreTargetingRisk(): targeting exposure
│   ├── parser/
│   │   ├── hybrid.ts     ← parseRoutingRequest(): regex rules → LLM fallback
│   │   └── llm.ts        ← LLM-backed NL parser
│   ├── router/
│   │   └── true-cost-router.ts ← routeWithTrueCost(): batch Dijkstra + diversification
│   └── agent/
│       └── copilot.ts    ← routeWithCopilot(): sequential Co-Pilot flow
│
└── api/                             SHARED HTTP UTILITIES
    ├── constants.ts      ← Rate limit constants, header names
    ├── errors.ts         ← ApiError class + HTTP error helpers
    ├── rate-limit.ts     ← Sliding-window rate limiter (Map-based, keyed by IP)
    ├── validate-route.ts ← Zod schema for POST /api/route
    └── validate-transmit.ts ← Zod schema for POST /api/transmit
```

---

## Algorithm → File → Function Reference Card

| Algorithm | File | Function |
|-----------|------|----------|
| Dijkstra (expanded state space) | `relic/router.ts` | `findShortestRoute()` |
| Void latency Tv | `relic/latency.ts` | `computeVoidLatency()` |
| Internal transit Tp | `relic/latency.ts` | `computeInternalLatency()` |
| Route aggregation | `relic/latency.ts` | `combineRouteLatency()` |
| ASCII ↔ base-N encoding | `relic/codec.ts` | `RelicCodec.encodeToCodex()` |
| Binary serialization | `relic/codec.ts` | `RelicCodec.serializeToBinary()` |
| Full transmission pipeline | `relic/transmission.ts` | `transmit()` |
| Failure state tracking | `relic/resilience.ts` | `ResilientNetwork` |
| Engine wiring | `relic/engine.ts` | `createEngine()` |
| Link anomaly detection | `chimera/link-evaluation.ts` | `detectLinkAnomaly()` |
| Combined scoring | `chimera/link-evaluation.ts` | `evaluateLink()` |
| Congestion model | `chimera/models/congestion.ts` | `predictCongestion()` |
| Trust / spoofing model | `chimera/models/trust.ts` | `scoreTrust()` |
| Targeting risk model | `chimera/models/targeting.ts` | `scoreTargetingRisk()` |
| True Cost routing | `chimera/router/true-cost-router.ts` | `routeWithTrueCost()` |
| Co-Pilot orchestration | `chimera/agent/copilot.ts` | `routeWithCopilot()` |
| NL parsing (hybrid) | `chimera/parser/hybrid.ts` | `parseRoutingRequest()` |

---

## Obsidian Vault Cross-Reference

| Topic | Vault doc |
|-------|-----------|
| Full architecture + data flow | `00 - Index.md` |
| Dijkstra algorithm walk-through | `relic-router.md` |
| Tv and Tp formulas | `relic-latency.md` |
| Codec binary pipeline | `relic-codec.md` |
| Link scoring formula | `chimera-link-evaluation.md` |
| True Cost vs Co-Pilot | `chimera-router-true-cost.md` |
| Co-Pilot sequential flow | `chimera-agent-copilot.md` |
| Congestion model | `chimera-models-congestion.md` |
| Trust model | `chimera-models-trust.md` |
| API rate limiting | `api-rate-limit.md` |

---

*Stack Kings · Relic Ring Protocol · Demo Script v2 · DevTools Edition · 2026-07-09*
