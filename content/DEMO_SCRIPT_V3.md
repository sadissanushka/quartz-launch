# Stack Kings · Relic Ring Protocol · Demo Script

**17 minutes. F12 open. `npm run dev` running.**

```
0:00  Block 1 — Architecture                 2 min
2:00  Block 2 — Universe graph               3 min
5:00  Block 3 — Routing + codec pipeline     4 min
9:00  Block 4 — Chaos / rerouting            3 min
12:00 Block 5 — Chimera Co-Pilot             2 min
14:00 Block 6 — Web UI walkthrough           3 min
17:00 Block 7 — Q&A
```

---

# Block 1 — Architecture `0:00`

Open `obsidian-vault/00 - Index.md`. Point at the data flow diagram.

> "Two phases. Phase 1 — Relic — is pure physics. Dijkstra over a solar-system graph with real latency math. Phase 2 — Chimera — is the AI layer: it overlays congestion, trust scoring, and targeting risk on top of that physics. The browser is our main interface."

---

# Block 2 — Universe Graph `2:00`

Open `src/lib/relic/types.ts`. Point at `PlanetNode`:

> "Each planet has a codex — its number base. Aegis is base 8, Dawn is base 6, Fenix is base 16. Every byte of payload gets re-encoded into that base at each hop. The graph is built from these nodes plus real void distances."

Open `src/lib/relic/engine.ts`, point at `createEngine()`:

> "This is the composition root. Geometry, codec, graph, resilient network — all wired here."

Paste in DevTools:

```javascript
const health = await fetch('/api/health').then(r => r.json());
console.log(health);
```

Then:

```javascript
const universe = await fetch('/api/universe').then(r => r.json());
console.table(universe.nodes.map(n => ({
  id: n.id,
  codex: `base ${n.codex}`,
  towers: n.active_towers,
  radius_km: n.radius_km
})));
```

Then:

```javascript
const reachable = universe.edges?.filter(e => e.within_lmax)
  ?? universe.interplanetaryLinks;
console.table(reachable.map(e => ({
  link: `${e.from ?? e.planet_a} ↔ ${e.to ?? e.planet_b}`,
  distance_Mkm: ((e.void_distance_km ?? 0) / 1_000_000).toFixed(2) + 'M km'
})));
```

> "Aegis–Boreas is the shortest hop at 18M km. There's no direct Aegis–Caelum link — 57M km exceeds Lmax. Packets have to route through intermediaries."

---

# Block 3 — Routing + Codec Pipeline `5:00`

Open `src/lib/relic/router.ts`, top comment (lines 1–14):

> "Standard Dijkstra over planet nodes would be wrong. Internal crust transit — Tp — depends on which tower the packet enters *and* exits. Same planet, different tower pair, different fiber arc cost. So we expand the state space to planet plus entry tower. The state key is `planetId#entryTower`."

Open `src/lib/relic/latency.ts`, show the two formulas:

> "Void travel Tv: atmospheric refraction on both ends, then light-speed across the void. Internal transit Tp: the fiber arc length divided by number of towers and fiber speed, plus per-tower processing overhead."

Paste in DevTools:

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

console.log('Path:', result.route.path.join(' → '));
console.log('Total latency:', result.route.total_latency_ms.toFixed(3), 'ms');
console.log('Breakdown:', result.route.breakdown);
console.log('Delivered payload:', result.delivered_payload);
```

Expected output:
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

> "229 seconds. Almost all of it is void travel — light-speed across 35M km of empty space. The fiber and tower overhead is tiny."

Now show the codec hop-by-hop:

```javascript
result.packet.hop_log.forEach(hop => {
  console.group(`[${hop.sequence}] ${hop.planet_id} (base ${hop.codex})`);
  console.log('Entry→Exit tower:', hop.entry_tower, '→', hop.exit_tower);
  console.log('Segments (s):', hop.segments, '| Towers hit (m):', hop.towers_hit);
  console.log('Internal Tp:', hop.internal_latency_ms?.toFixed(3), 'ms');
  console.log('Void Tv:    ', hop.void_latency_ms?.toFixed(3) ?? '—', 'ms');
  console.log('Cumulative: ', hop.cumulative_latency_ms.toFixed(3), 'ms');
  console.log('Dialect digits:', hop.payload_dialect.digits.slice(0, 5), '...');
  console.groupEnd();
});
```

> "Same payload — 'Hello world' — encoded as base-8 digits leaving Aegis, re-encoded as base-6 at Dawn, base-14 at Caelum. The `delivered_payload` matching the original is our integrity proof."

Open `src/lib/relic/codec.ts`:

> "Per hop: ASCII bytes to base-N digit strings, serialized to a binary stream — each digit character encoded as 8-bit groups with a space separator — then fully reversed on arrival. The space is the unambiguous delimiter because codex digits are 0–9 and A–Z only."

Open `src/lib/relic/transmission.ts`:

> "`transmit()` orchestrates all of this: shortest route, then the codec loop per hop, then assembles the full hop log."

---

# Block 4 — Chaos / Rerouting `9:00`

Open `src/lib/relic/resilience.ts`:

> "No in-flight packet state. Every `send()` is a fresh Dijkstra run against whatever failures are currently active. Kill a node, cut a link — next packet routes around it immediately."

Kill Dawn:

```javascript
const chaos = await fetch('/api/transmit', {
  method: 'POST',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify({
    origin: 'Aegis',
    destination: 'Caelum',
    payload: 'Hello world',
    blockedNodes: ['Dawn'],
  })
}).then(r => r.json());

console.log('Rerouted path:', chaos.route.path.join(' → '));
console.log('New latency:  ', chaos.route.total_latency_ms.toFixed(3), 'ms');

```

Expected:
```
Rerouted path:  Aegis → Elysium → Caelum
New latency:    280423.074 ms
```

> "+51 seconds. Aegis–Elysium is 46M km versus Aegis–Dawn's 35M km — longer void, but it's the best available path."

Cut a link instead of a node:

```javascript
const cutLink = await fetch('/api/transmit', {
  method: 'POST',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify({
    origin: 'Aegis',           // ✅ was origin_id
    destination: 'Caelum',     // ✅ was destination_id
    payload: 'Hello world',
    blockedEdges: [['Aegis', 'Dawn']],  // ✅ top-level, not nested under options
  })
}).then(r => r.json());

console.log('Link-cut path:', cutLink.route.path.join(' → '));

```

> "Dawn is still alive, Aegis is still alive — just that specific link is severed. Both planets remain reachable via other hops."

Force undeliverable:

```javascript
const isolated = await fetch('/api/transmit', {
  method: 'POST',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify({
    origin: 'Aegis',                              // ✅ was origin_id
    destination: 'Caelum',                        // ✅ was destination_id
    payload: 'Hello world',
    blockedNodes: ['Dawn', 'Elysium', 'Fenix'],   // ✅ top-level, not nested
  })
}).then(r => r.json());

console.log('Status:', isolated.packet.status);
console.log('Reason:', isolated.packet.undeliverable_reason);

```

> "Kill every neighbor of Caelum and you get `undeliverable`. No route within Lmax — the system fails cleanly, not silently."

---

# Block 5 — Chimera Co-Pilot `12:00`

Open `obsidian-vault/chimera-router-true-cost.md`, point at the comparison table:

> "Two modes. True Cost Router — batch Dijkstra with surcharges baked into edge weights, takes structured input. Co-Pilot — sequential, hop-by-hop evaluation, takes a natural language string."

Open `src/lib/chimera/agent/copilot.ts`, point at the top comment.

True Cost Router:

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

trueCost.link_evaluations?.forEach(ev => {
  console.log(`\n${ev.link_id}`);
  console.log('  congestion penalty:', ev.predicted_congestion_penalty_ms, 'ms');
  console.log('  trust score:       ', ev.trust_score);
  console.log('  targeting risk:    ', ev.targeting_risk_score);
  console.log('  combined_cost:     ', ev.combined_cost, 'ms');
});
```

> "Combined cost is physics void latency plus congestion penalty, plus a trust penalty — lower trust means the link is acting suspiciously, costs more — plus targeting risk, minus an entropy bonus that steers traffic away from over-concentrated paths."

Co-Pilot with natural language:

```javascript
const copilot = await fetch('/api/route', {
  method: 'POST',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify({
    request: 'send a message from Aegis to Caelum as fast as possible'
  })
}).then(r => r.json());

console.log('Parse source:', copilot.parse_source ?? '(check explanation)');
console.log('Chosen path:', copilot.path?.join(' → '));
console.log('Reroutes:', copilot.reroute_count ?? 0);
console.log('Explanation:', copilot.explanation);
```

> "Parser tries regex rules first — 'from X to Y' patterns — falls back to an LLM call if it can't extract origin and destination. `parse_source` tells you which path ran."

Open `src/lib/chimera/link-evaluation.ts`, show `detectLinkAnomaly()`:

> "If a link reports out-of-range values — load ratio above 1, negative capacity — we clamp, flag it, and apply conservative scores. Trust floor of 0.3, targeting risk floor of 0.85. It stays usable as a last resort but costs a lot."

---

# Block 6 — Web UI Walkthrough `14:00`

Navigate to `http://localhost:3000` (or `http://localhost:3000/relic`).

> "Everything you just ran in the console is wired up in the dashboard. Let me show you the same data paths through the UI."

---

## 6.1 — Space Map

Point at the **SpaceMap** canvas:

> "This is the live universe graph — every planet and every edge that passed the Lmax constraint. Colour intensity represents void distance. The highlighted path is the last route we computed."

Click a planet node:

> "Click any node — you see its codex base, active tower count, and radius. These are the exact `PlanetNode` fields from `types.ts`."

---

## 6.2 — Codex Terminal

Open the **Codex Terminal** panel. Type `Aegis`, `Caelum`, `Hello world` into the three fields and hit **Transmit**:

> "Same as our DevTools demo but rendered step-by-step. Each row is one hop. Base-8 leaving Aegis, base-6 at Dawn, back to ASCII at Caelum. The green checkmark is our integrity proof — `delivered_payload` matched the original."

---

## 6.3 — Latency Metrics

Point at the **LatencyMetrics** bar chart:

> "The breakdown — `fiber_ms`, `tower_ms`, `atmosphere_ms`, `void_ms` — rendered as stacked bars. Void dominates at 229 seconds. Everything else is sub-millisecond noise."

---

## 6.4 — Chaos Mode in the UI

Toggle **Block Dawn** in the UI controls (or equivalent node failure toggle):

> "This is the same `blockedNodes: ['Dawn']` flag the API accepts — the UI just wraps it. Watch the path on the SpaceMap redraw to Aegis → Elysium → Caelum."

Point at the latency delta:

> "+51 seconds on the map — same numbers we saw in the console. No state to flush. Every render is a fresh Dijkstra run."

---

## 6.5 — Link Evaluations Panel (Chimera)

Open the **LinkEvaluationsPanel**:

> "This calls `POST /api/route` with the structured payload — the True Cost Router path. Each row is a link. Congestion penalty, trust score, targeting risk, and combined cost side-by-side."

Hover over the trust score column:

> "Trust score below 0.5 means the link's self-reported latency is faster than physics allows — that's the spoofing detection in `trust.ts`."

---

## 6.6 — Co-Pilot Natural Language (Chimera)

Paste in DevTools (or use the Co-Pilot input if visible in the UI):

```javascript
const copilot = await fetch('/api/route', {
  method: 'POST',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify({
    request: 'send a message from Aegis to Caelum avoiding congested links'
  })
}).then(r => r.json());
console.log('parse_source:', copilot.parse_source);
console.log('path:', copilot.path?.join(' → '));
console.log('explanation:', copilot.explanation);
```

> "The `explanation` field populates the Intelligence Summary panel — prose from the Co-Pilot agent describing *why* it chose this path. `parse_source: "regex"` means it matched the pattern without hitting the LLM. Change the request to something more ambiguous and you'll see it flip to `"llm"`."

---

# Reference — Where Things Live

```
src/lib/
├── relic/
│   ├── types.ts          PlanetNode, Universe, Packet, HopLogEntry
│   ├── graph.ts          buildNetworkGraph(), edgeKey(), VoidEdge
│   ├── latency.ts        computeVoidLatency(Tv), computeInternalLatency(Tp)
│   ├── codec.ts          RelicCodec — ASCII ↔ base-N ↔ binary stream
│   ├── router.ts         findShortestRoute() — Dijkstra over (planet, tower) states
│   ├── transmission.ts   transmit() — router + codec + hop_log orchestration
│   ├── resilience.ts     ResilientNetwork — stateful failure tracking
│   └── engine.ts         createEngine() — composition root
│
└── chimera/
    ├── link-evaluation.ts    detectLinkAnomaly() + evaluateLink()
    ├── models/congestion.ts  predictCongestion(): k × load_ratio^p
    ├── models/trust.ts       scoreTrust(): physics vs self-reported delta
    ├── models/targeting.ts   scoreTargetingRisk()
    ├── router/true-cost-router.ts  routeWithTrueCost()
    ├── agent/copilot.ts      routeWithCopilot()
    └── parser/hybrid.ts      parseRoutingRequest(): regex → LLM fallback
```

| Algorithm | File | Function |
|-----------|------|----------|
| Dijkstra (expanded state) | `relic/router.ts` | `findShortestRoute()` |
| Void latency Tv | `relic/latency.ts` | `computeVoidLatency()` |
| Internal transit Tp | `relic/latency.ts` | `computeInternalLatency()` |
| ASCII ↔ base-N | `relic/codec.ts` | `RelicCodec.encodeToCodex()` |
| Full transmission | `relic/transmission.ts` | `transmit()` |
| Failure tracking | `relic/resilience.ts` | `ResilientNetwork` |
| Link scoring | `chimera/link-evaluation.ts` | `evaluateLink()` |
| Congestion model | `chimera/models/congestion.ts` | `predictCongestion()` |
| Trust model | `chimera/models/trust.ts` | `scoreTrust()` |
| True Cost routing | `chimera/router/true-cost-router.ts` | `routeWithTrueCost()` |
| Co-Pilot agent | `chimera/agent/copilot.ts` | `routeWithCopilot()` |

---

# Block 7 — Q&A Pocket Answers

**What is `codex`?**
Each planet speaks a different number base. Every ASCII byte is encoded into that base before crossing the void. `encodeToCodex()` in `codec.ts` — `byte.toString(base).toUpperCase()`.

**Why is Lmax a hard constraint?**
Models laser range. Beyond Lmax the signal can't punch through the void. Enforced in `graph.ts → buildNetworkGraph()`: `voidDistanceKm <= lmax`.

**Why `(planet, entry_tower)` state and not just planet?**
Tp is turn-dependent. The fiber arc you traverse depends on entry tower *and* exit tower. Plain node-weighted Dijkstra charges the wrong Tp. See `router.ts`: `effectiveEntry = entryTower === NO_ENTRY ? exitTower : entryTower`.

**Why does Aegis charge `s=0, m=1` but Dawn charges `s=3, m=4`?**
At origin, packet exits from the same tower it sends from — no traversal. At Dawn, packet arrives at tower 4 but needs to exit tower 1 for the Dawn→Caelum hop — 3 segments, 4 towers hit. At destination, `s=0` again.

**`blockedNodes` vs `blockedEdges`?**
`blockedNodes` removes an entire planet. `blockedEdges` severs one specific link — both planets stay alive via other paths. Router checks both in `reachable()` at `router.ts` lines 255–259.

**What if you kill the origin or destination?**
`findShortestRoute()` checks that first and returns `undeliverable` immediately with `"Origin is offline"` or `"Destination is offline"`. Try: `options: { blockedNodes: ['Aegis'] }`.

**Why doesn't Co-Pilot use global Dijkstra?**
It's intentional — mimics a sequential decision agent, evaluates one hop at a time, reroutes mid-path. True Cost Router is batch-optimal but can't take NL input.

**What are the three sub-models?**
`congestion.ts` — piecewise power-law `penalty = k × load_ratio^p`, hard-blocks if load ≥ 0.90 or status `saturated`.
`trust.ts` — compares self-reported latency vs physics Tv + congestion. If the link claims to be faster than physics allows, that's a spoofing signal.
`targeting.ts` — scores exposure risk per hop.

**What does the entropy bonus do?**
Reduces `combined_cost` for links with higher traffic share — steers *toward* busy links to spread load rather than concentrating everything on the optimal path. Also drives 5% diversification tolerance in `true-cost-router.ts`.
