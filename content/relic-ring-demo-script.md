# Stack Kings · Relic Ring Protocol — Demo Script

**Runtime:** 17 min · **Setup:** `npm run dev` running, browser DevTools (F12) open  
**Companion reference:** full per-file code review vault at `launch-help.wssat.me`

---

## What This Demo Is Showing

The project has two layers, and the whole script walks through them in order:

| Phase | Name | What it is |
|---|---|---|
| **Phase 1** | **Relic** | Pure physics — Dijkstra shortest-path routing over a solar-system graph, with real latency math (void travel + internal planet transit) and a base-N encoding "codec" applied to every packet at every hop. |
| **Phase 2** | **Chimera** | An AI/heuristics layer on top of Relic — adds congestion prediction, trust scoring (spoofing detection), and targeting-risk scoring to the same physics graph. Exposes both a structured router and a natural-language "Co-Pilot" router. |

The demo proves this in three ways: **(1)** console/API calls showing raw numbers, **(2)** chaos tests showing the router adapts live with no stored state, **(3)** the web UI showing the same data rendered visually.

---

## Timeline

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

## Block 1 — Architecture (`0:00`, 2 min)

**Purpose:** Orient the audience before touching any code — establish the two-phase mental model (physics vs. AI overlay) so every later demo slots into place.

**Action:** Open `obsidian-vault/00 - Index.md`. Point at the data-flow diagram.

> "Two phases. Phase 1 — Relic — is pure physics. Dijkstra over a solar-system graph with real latency math. Phase 2 — Chimera — is the AI layer: it overlays congestion, trust scoring, and targeting risk on top of that physics. The browser is our main interface."

---

## Block 2 — Universe Graph (`2:00`, 3 min)

**Purpose:** Introduce the data model — planets, codex bases, and void-distance edges — that everything else routes over.

**Action:** Open `src/lib/relic/types.ts` → `PlanetNode`.

> "Each planet has a codex — its number base. Aegis is base 8, Dawn is base 6, Fenix is base 16. Every byte of payload gets re-encoded into that base at each hop. The graph is built from these nodes plus real void distances."

Then open `src/lib/relic/engine.ts` → `createEngine()`:

> "This is the composition root. Geometry, codec, graph, resilient network — all wired here."

**DevTools calls, in order:**

**1. Health check:**

```javascript
const health = await fetch('/api/health').then(r => r.json());
console.log(health);
```

**2. Universe table:**

```javascript
const universe = await fetch('/api/universe').then(r => r.json());
console.table(universe.nodes.map(n => ({
  id: n.id,
  codex: `base ${n.codex}`,
  towers: n.active_towers,
  radius_km: n.radius_km
})));
```

**3. Reachable links (within Lmax):**

```javascript
const reachable = universe.edges?.filter(e => e.within_lmax)
  ?? universe.interplanetaryLinks;
console.table(reachable.map(e => ({
  link: `${e.from ?? e.planet_a} ↔ ${e.to ?? e.planet_b}`,
  distance_Mkm: ((e.void_distance_km ?? 0) / 1_000_000).toFixed(2) + 'M km'
})));
```

**Payoff line:**
> "Aegis–Boreas is the shortest hop at 18M km. There's no direct Aegis–Caelum link — 57M km exceeds Lmax. Packets have to route through intermediaries."

---

## Block 3 — Routing + Codec Pipeline (`5:00`, 4 min)

**Purpose:** This is the technical core of Phase 1 — show *why* plain Dijkstra isn't enough, then prove a full send/encode/decode round-trip end to end.

**Action 1 — explain the state-space trick.** Open `src/lib/relic/router.ts` (lines 1–14):

> "Standard Dijkstra over planet nodes would be wrong. Internal crust transit — Tp — depends on which tower the packet enters *and* exits. Same planet, different tower pair, different fiber arc cost. So we expand the state space to planet plus entry tower. The state key is `planetId#entryTower`."

**Action 2 — show the two latency formulas.** Open `src/lib/relic/latency.ts`:

> "Void travel Tv: atmospheric refraction on both ends, then light-speed across the void. Internal transit Tp: the fiber arc length divided by number of towers and fiber speed, plus per-tower processing overhead."

**Action 3 — run a live transmission.** `POST /api/transmit`:

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

**Action 4 — walk the codec hop-by-hop:**

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

**Action 5 — explain the codec mechanics.** Open `src/lib/relic/codec.ts`:

> "Per hop: ASCII bytes to base-N digit strings, serialized to a binary stream — each digit character encoded as 8-bit groups with a space separator — then fully reversed on arrival. The space is the unambiguous delimiter because codex digits are 0–9 and A–Z only."

**Action 6 — tie it together.** Open `src/lib/relic/transmission.ts`:

> "`transmit()` orchestrates all of this: shortest route, then the codec loop per hop, then assembles the full hop log."

---

## Block 4 — Chaos / Rerouting (`9:00`, 3 min)

**Purpose:** Prove the router is stateless and resilient — no packet ever depends on prior routing state, so failures are handled by simply re-running Dijkstra against the current failure set.

**Action 1 — explain the model.** Open `src/lib/relic/resilience.ts`:

> "No in-flight packet state. Every `send()` is a fresh Dijkstra run against whatever failures are currently active. Kill a node, cut a link — next packet routes around it immediately."

**Action 2 — kill a node.** Resend with `blockedNodes: ['Dawn']`:

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

**Action 3 — cut a single link instead of a whole node.** Resend with `blockedEdges: [['Aegis', 'Dawn']]`:

```javascript
const cutLink = await fetch('/api/transmit', {
  method: 'POST',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify({
    origin: 'Aegis',
    destination: 'Caelum',
    payload: 'Hello world',
    blockedEdges: [['Aegis', 'Dawn']],
  })
}).then(r => r.json());

console.log('Link-cut path:', cutLink.route.path.join(' → '));
```

> "Dawn is still alive, Aegis is still alive — just that specific link is severed. Both planets remain reachable via other hops."

**Action 4 — force total isolation.** Resend with `blockedNodes: ['Dawn', 'Elysium', 'Fenix']` (every neighbor of Caelum):

```javascript
const isolated = await fetch('/api/transmit', {
  method: 'POST',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify({
    origin: 'Aegis',
    destination: 'Caelum',
    payload: 'Hello world',
    blockedNodes: ['Dawn', 'Elysium', 'Fenix'],
  })
}).then(r => r.json());

console.log('Status:', isolated.packet.status);
console.log('Reason:', isolated.packet.undeliverable_reason);
```

> "Kill every neighbor of Caelum and you get `undeliverable`. No route within Lmax — the system fails cleanly, not silently."

---

## Block 5 — Chimera Co-Pilot (`12:00`, 2 min)

**Purpose:** Show Phase 2 — the same physics graph, but now scored by congestion/trust/targeting models, accessible either as a structured batch router or a natural-language agent.

**Action:** Open `obsidian-vault/chimera-router-true-cost.md` (comparison table), then `src/lib/chimera/agent/copilot.ts` (top comment):

> "Two modes. True Cost Router — batch Dijkstra with surcharges baked into edge weights, takes structured input. Co-Pilot — sequential, hop-by-hop evaluation, takes a natural language string."

**Action — True Cost Router.** `POST /api/route` with `{ origin_id, destination_id }`:

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

**Action — Co-Pilot with natural language.** `POST /api/route` with `{ request }`:

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

**Action — anomaly handling.** Open `src/lib/chimera/link-evaluation.ts` → `detectLinkAnomaly()`:

> "If a link reports out-of-range values — load ratio above 1, negative capacity — we clamp, flag it, and apply conservative scores. Trust floor of 0.3, targeting risk floor of 0.85. It stays usable as a last resort but costs a lot."

---

## Block 6 — Web UI Walkthrough (`14:00`, 3 min)

**Purpose:** Show that every console/API call from Blocks 2–5 is the exact data driving the dashboard — nothing in the UI is mocked separately.

Navigate to `http://localhost:3000` (or `/relic`):

> "Everything you just ran in the console is wired up in the dashboard. Let me show you the same data paths through the UI."

| Sub-section | What it shows | Maps back to |
|---|---|---|
| **6.1 Space Map** | Live universe graph canvas; edges within Lmax only; colour = void distance; highlighted path = last computed route. Click a node to see its `PlanetNode` fields (codex base, active towers, radius). | Block 2 (`GET /api/universe`) |
| **6.2 Codex Terminal** | Type origin/destination/payload, hit Transmit — renders the hop-by-hop codec trace with a green integrity checkmark. | Block 3 (`POST /api/transmit`) |
| **6.3 Latency Metrics** | Stacked bar chart of `fiber_ms` / `tower_ms` / `atmosphere_ms` / `void_ms`. Void dominates at 229 s; everything else is noise. | Block 3 latency breakdown |
| **6.4 Chaos Mode** | UI toggle for `blockedNodes`; watch the Space Map redraw and the latency delta update live (+51 s when Dawn is blocked). No state to flush — every render is a fresh Dijkstra run. | Block 4 (node failure) |
| **6.5 Link Evaluations Panel** | Renders `POST /api/route` (structured/True Cost) results — congestion, trust, targeting, combined cost per link. Trust < 0.5 flags spoofing (self-reported latency faster than physics allows). | Block 5 (True Cost Router) |
| **6.6 Co-Pilot Natural Language** | Run a more ambiguous NL request; `explanation` field populates the Intelligence Summary panel; `parse_source` shows whether regex or LLM handled the parse. | Block 5 (Co-Pilot) |

**6.6 snippet — paste in DevTools:**

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

> "Change the request to something more ambiguous and you'll see `parse_source` flip from `\"regex\"` to `\"llm\"`."

---

## Reference — Where Things Live

> The full file-by-file code review vault is published at **[launch-help.wssat.me](https://launch-help.wssat.me/)** — every file below has its own detailed page there (e.g. `relic-router`, `chimera-agent-copilot`).

### Repo layout

```
stack-kings/
├── src/
│   ├── app/                   # Next.js App Router (pages + API routes)
│   │   ├── api/
│   │   │   ├── health/        GET  /api/health
│   │   │   ├── route/         POST /api/route     (Co-Pilot / True Cost routing)
│   │   │   ├── transmit/      POST /api/transmit
│   │   │   ├── universe/      GET  /api/universe
│   │   │   └── debug/         Sentry debug trigger
│   │   ├── layout.tsx
│   │   └── page.tsx
│   ├── cli/                   # Terminal demo (milestones M1–M4)
│   ├── components/
│   │   ├── observability/     # Sentry error boundary components
│   │   └── telemetry/         # Dashboard UI components
│   └── lib/
│       ├── api/               # Shared API utilities (rate-limit, validate, errors)
│       ├── chimera/           # Phase 2 — Chimera Co-Pilot engine
│       │   ├── agent/         # Co-Pilot orchestrator
│       │   ├── models/        # Congestion / trust / targeting sub-models
│       │   ├── parser/        # NL parser (regex rules → LLM fallback)
│       │   ├── router/        # True Cost router
│       │   └── csv/           # CSV data loaders
│       ├── observability/     # Sentry integration
│       └── relic/             # Phase 1 — core physics engine
│           ├── server/        # Node-only universe loader
│           └── stubs/         # Geometry / codec stubs
```

### Core algorithms by file

| Algorithm | File | Function | What it does |
|---|---|---|---|
| Dijkstra (expanded state) | `relic/router.ts` | `findShortestRoute()` | Shortest path over `(planet, entry_tower)` states, not plain planet nodes |
| Void latency (Tv) | `relic/latency.ts` | `computeVoidLatency()` | Atmospheric refraction + light-speed across void distance |
| Internal transit (Tp) | `relic/latency.ts` | `computeInternalLatency()` | Fiber arc length ÷ towers × fiber speed, + per-tower overhead |
| ASCII ↔ base-N codec | `relic/codec.ts` | `RelicCodec.encodeToCodex()` | Re-encodes payload bytes into each planet's codex base per hop |
| Full transmission | `relic/transmission.ts` | `transmit()` | Orchestrates routing + codec loop + hop log assembly |
| Failure tracking | `relic/resilience.ts` | `ResilientNetwork` | Stateless re-routing around blocked nodes/edges on every send |
| Link scoring | `chimera/link-evaluation.ts` | `evaluateLink()` | Combines congestion, trust, targeting into one score; flags anomalies |
| Congestion model | `chimera/models/congestion.ts` | `predictCongestion()` | Piecewise power-law penalty; hard-blocks near-saturated links |
| Trust model | `chimera/models/trust.ts` | `scoreTrust()` | Flags links whose self-reported latency beats physics (spoofing) |
| Targeting model | `chimera/models/targeting.ts` | `scoreTargetingRisk()` | Scores exposure/interception risk per hop |
| True Cost routing | `chimera/router/true-cost-router.ts` | `routeWithTrueCost()` | Batch Dijkstra with congestion/trust/targeting baked into edge weights |
| Co-Pilot agent | `chimera/agent/copilot.ts` | `routeWithCopilot()` | Sequential, hop-by-hop agent; takes natural-language input |
| NL parsing | `chimera/parser/hybrid.ts` | `parseRoutingRequest()` | Regex "from X to Y" rules first, LLM fallback if that fails |

### Other files (per the code review vault)

| File | Purpose |
|---|---|
| `relic/types.ts` | Core types — `PlanetNode`, `Universe`, `Packet`, `HopLogEntry` |
| `relic/contracts.ts` | Shared interface contracts for the physics layer |
| `relic/config.ts` | Tunable constants (Lmax, fiber speed, etc.) |
| `relic/graph.ts` | `buildNetworkGraph()`, `edgeKey()`, `VoidEdge` — builds the reachable-edge graph respecting Lmax |
| `relic/engine.ts` | `createEngine()` — composition root wiring geometry, codec, graph, resilient network |
| `relic/server/universe.ts` | Node-only server-side universe loader |
| `chimera/index.ts` | Public barrel export for the Chimera module |
| `chimera/types.ts` | Chimera-specific types |
| `chimera/constants.ts` | Chimera tuning constants |
| `chimera/client.ts` | Fetches live Chimera `/state` data |
| `chimera/health.ts` | Chimera health check |
| `chimera/link-id.ts` | Canonical link ID helper |
| `chimera/report-schema.ts` | Schema used to validate router output |
| `chimera/routing-report.ts` | Assembles the final routing report/explanation |
| `chimera/parser/llm.ts` | LLM-based NL parsing fallback |
| `api/constants.ts`, `api/errors.ts`, `api/rate-limit.ts` | Shared API request handling |
| `api/validate-route.ts`, `api/validate-transmit.ts` | Request payload validation |
| `app/layout.tsx` | App shell layout |
| `cli/relic.ts` | Terminal-based demo CLI |

### Data flow (from the vault)

```
POST /api/route (natural language)
  └─→ chimera/agent/copilot.ts
        ├─→ chimera/parser/hybrid.ts     (NL → origin/destination)
        ├─→ chimera/client.ts            (fetch live Chimera /state)
        ├─→ relic/router.ts              (Dijkstra physics path)
        ├─→ chimera/link-evaluation.ts   (score each hop)
        └─→ chimera/report-schema.ts     (validate output)

POST /api/route (structured)
  └─→ chimera/router/true-cost-router.ts
        ├─→ chimera/client.ts
        ├─→ chimera/link-evaluation.ts
        └─→ chimera/models/congestion.ts + trust.ts + targeting.ts
```

---

## Block 7 — Q&A Pocket Answers (`17:00`)

**What is `codex`?**
Each planet speaks a different number base. Every ASCII byte is encoded into that base before crossing the void — `encodeToCodex()` in `codec.ts`: `byte.toString(base).toUpperCase()`.

**Why is Lmax a hard constraint?**
Models laser range — beyond Lmax the signal can't punch through the void. Enforced in `graph.ts → buildNetworkGraph()`: `voidDistanceKm <= lmax`.

**Why `(planet, entry_tower)` state and not just planet?**
Tp is turn-dependent — the fiber arc traversed depends on entry tower *and* exit tower. Plain node-weighted Dijkstra would charge the wrong Tp. See `router.ts`: `effectiveEntry = entryTower === NO_ENTRY ? exitTower : entryTower`.

**Why does Aegis charge `s=0, m=1` but Dawn charges `s=3, m=4`?**
At origin, the packet exits from the same tower it sends from — no traversal. At Dawn, it arrives at tower 4 but needs to exit tower 1 for the next hop — 3 segments, 4 towers hit. At destination, `s=0` again.

**`blockedNodes` vs `blockedEdges`?**
`blockedNodes` removes an entire planet. `blockedEdges` severs one specific link — both planets stay alive via other paths. Checked in `reachable()` at `router.ts` lines 255–259.

**What if you kill the origin or destination?**
`findShortestRoute()` checks that first and returns `undeliverable` immediately with `"Origin is offline"` or `"Destination is offline"`. Try `blockedNodes: ['Aegis']`.

**Why doesn't Co-Pilot use global Dijkstra?**
Intentional — it mimics a sequential decision agent, evaluating one hop at a time and rerouting mid-path. True Cost Router is batch-optimal but can't take natural-language input.

**What are the three Chimera sub-models?**
- `congestion.ts` — piecewise power-law: `penalty = k × load_ratio^p`; hard-blocks if load ≥ 0.90 or status `saturated`.
- `trust.ts` — compares self-reported latency vs. physics Tv + congestion; a link claiming to be faster than physics allows is a spoofing signal.
- `targeting.ts` — scores exposure/interception risk per hop.

**What does the entropy bonus do?**
Reduces `combined_cost` for links with higher traffic share — steers *toward* busy links to spread load rather than concentrating everything on the optimal path. Also drives the 5% diversification tolerance in `true-cost-router.ts`.
