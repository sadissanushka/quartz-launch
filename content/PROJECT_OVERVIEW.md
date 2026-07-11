# Stack Kings — Complete Project Overview

**Team:** Stack Kings (LAUNCH 26)  
**Members:** Malawige Inusha Thathsara Gunasekara · Anushka · Pasindu Ruwan  
**Live demo:** [relic.inusha.me/relic](https://relic.inusha.me/relic) · [launch-26-stack-kings.vercel.app/relic](https://launch-26-stack-kings.vercel.app/relic)  
**Repositories:** [Launch26/stack-kings](https://github.com/Launch26/stack-kings) · personal fork also synced for Vercel  

This document is the single place to understand **the entire project** — Phase 1 Relic Ring physics routing and Phase 2 Chimera Co-Pilot — from problem statement through architecture, models, APIs, UI, testing, and deployment.

Related docs:

| Doc | Purpose |
|-----|---------|
| [`DEMO_SCRIPT.md`](DEMO_SCRIPT.md) | Live demo talking points (~10–12 min) |
| [`challenge/DECISION_AUDIT.md`](challenge/DECISION_AUDIT.md) | Score formulas for Decision Audit trial |
| [`challenge/INTELLIGENCE_REPORT.md`](challenge/INTELLIGENCE_REPORT.md) | Model evaluation metrics |
| [`challenge/PHASE2_PROJECT_REPORT.md`](challenge/PHASE2_PROJECT_REPORT.md) | Formal ≤8-page submission report |
| [`QA_SESSION_GUIDE.md`](QA_SESSION_GUIDE.md) | Q&A prep, traps, limitations |

---

## 1. What problem are we solving?

### 1.1 The fiction (challenge narrative)

The **Zeta-26** star system lost reliable interplanetary communication. Planets sit on a 2D map; each has equatorial **towers**, a **codex** (numeric base for encoding), atmosphere, and fiber rings. Messages must:

1. Travel through a planet’s internal fiber/tower network.
2. Cross the **void** as a laser hop (subject to max distance `Lmax`).
3. Be re-encoded into the **next hop’s** dialect before each void crossing.
4. Survive node/link failures by dynamic rerouting.

### 1.2 Phase 1 — Relic Ring Protocol

Build a **physics-honest** routing and transmission stack:

- Parse `universe-config.json`.
- Compute void and internal latencies from documented equations.
- Find lowest-latency routes under `Lmax`.
- Encode/decode between codices; prove integrity with a `hop_log`.
- Reroute around killed nodes/links (chaos / resilience).

### 1.3 Phase 2 — Chimera Co-Pilot

An adversary, **Chimera**, sabotages **interplanetary links only** (not planet physics):

| Attack | What it does |
|--------|----------------|
| **Congestion** | Artificial delay as load rises; hard failure when saturated |
| **Spoofed telemetry** | Links under-report latency so naive routers prefer them |
| **Predictable-route targeting** | Jams high-traffic / predictable links |

**Requirement:** Do **not** replace the Phase 1 router. Add an **Analytical Co-Pilot** that scores links with three trained models, routes on a **True Cost**, and emits a mandatory Council JSON schema (`Phase2RoutingReport`).

---

## 2. High-level architecture

```
┌─────────────────────────────────────────────────────────────────────────┐
│                         USER / JUDGES                                    │
│   Dashboard /relic  ·  CLI `npm run relic`  ·  curl APIs                 │
└───────────────────────────────┬─────────────────────────────────────────┘
                                │
┌───────────────────────────────▼─────────────────────────────────────────┐
│  Next.js App Router (TypeScript)                                         │
│  GET /api/health · GET /api/universe · POST /api/transmit · POST /api/route│
└───────┬───────────────────────────────┬─────────────────────────────────┘
        │                               │
┌───────▼──────────┐          ┌─────────▼──────────────────────────────────┐
│ Phase 1 Relic    │          │ Phase 2 Chimera Co-Pilot                   │
│ src/lib/relic/   │          │ src/lib/chimera/                           │
│ · config         │◄─────────│ · hybrid NL parser (+ optional LLM)        │
│ · geometry/codec │  wraps   │ · ChimeraClient (/state tick cache)        │
│ · latency        │  physics │ · congestion / trust / targeting models    │
│ · Dijkstra router│  router  │ · link-evaluation + anomaly sanitisation   │
│ · transmission   │          │ · True Cost router + sequential agent      │
│ · resilience     │          │ · Zod Phase2RoutingReport schema           │
└──────────────────┘          └─────────┬──────────────────────────────────┘
                                        │
                              ┌─────────▼─────────┐
                              │ chimera.launch26  │
                              │ .space /state     │
                              └───────────────────┘
```

**Design rule:** Phase 1 is the foundation. Phase 2 sits **in front of** it — scoring and path selection — then optionally **constrains** transmission so the packet’s `hop_log` follows the Co-Pilot path.

---

## 3. Phase 1 — Relic Ring (physics stack)

### 3.1 Universe config

Loaded from (first existing path wins):

1. `challenge p2/universe-config.json` — Phase 2 extended (12 interplanetary links + capacities)
2. `challenge p1/universe-config.json` — Phase 1 baseline
3. `universe-config.json` — root fallback

Override with `UNIVERSE_CONFIG_PATH`.

Planets include: coordinates, radius, atmosphere, towers, **codex base**, etc. Phase 2 adds `interplanetary_links[]` with `link_id`, `planet_a`, `planet_b`, `capacity_units`. Link IDs are **canonical alphabetical** (`Aegis-Boreas`, never `Boreas-Aegis`).

### 3.2 Latency model (all ms; constants from metadata)

| Component | Idea |
|-----------|------|
| Void distance `L` | Scaled center distance minus atmospheres |
| Void travel `Tv` | Atmosphere + void / speed of light |
| Internal crust `Tp` | Fiber arc + tower processing delay |
| Total | Sum of `Tp` over planets + `Tv` over void hops |

### 3.3 Routing

- **Dijkstra** over expanded state `(planet, entryTower)` because internal cost depends on entry vs exit tower.
- Enforces `Lmax` per void hop.
- Optional `blockedNodes` / `blockedEdges` for resilience.
- Phase 2 adds `voidHopSurchargeMs` so True Cost can inflate edge weights without rewriting Dijkstra.

### 3.4 Codec & transmission

Flow per hop:

```
ASCII → encode to next-hop codex → serialize binary → void → deserialize → decode to ASCII
```

`transmit()` builds a `packet` with `hop_log` (per-planet dialect, next-hop encoding, binary stream, latencies). Status: `delivered` or `undeliverable`.

### 3.5 Resilience

`ResilientNetwork` / dashboard scenarios (Hyper-Flare, Distortion, Blackout, Chaos) kill nodes/links; next send recomputes the route.

### 3.6 Phase 1 milestones (demo)

| Milestone | What judges see |
|-----------|-----------------|
| M1 | Universe init — planets, links, `/api/universe` |
| M2 | Multi-hop dialect proof — Codex Terminal |
| M3 | Latency breakdown — fiber / tower / atmosphere / void |
| M4 | Chaos — kill node/link, reroute |

---

## 4. Phase 2 — Chimera Co-Pilot

### 4.1 Module map (`src/lib/chimera/`)

| Path | Role | Owner |
|------|------|-------|
| `client.ts` | HTTP client for Chimera `/links`, `/state`; tick cache | Inusha |
| `csv/` | Load traffic / telemetry / incident CSVs | Ruwan |
| `models/` | Congestion, trust, targeting + `*.model.json` | Ruwan |
| `parser/hybrid.ts` | Rules-first NL → structured intent | Inusha |
| `parser/llm.ts` | Optional Ollama / Gemini fallback | Inusha |
| `link-evaluation.ts` | Anomaly sanitize + `combined_cost` | Inusha |
| `router/true-cost-router.ts` | Weighted Dijkstra + diversification | Inusha |
| `agent/copilot.ts` | Sequential hop loop + reroute | Inusha |
| `report-schema.ts` | Zod validation of Council JSON | Inusha |
| `health.ts` | Health probe fields for `/api/health` | Inusha |

### 4.2 Live Chimera API

- Base URL: `https://chimera.launch26.space` (`CHIMERA_API_BASE_URL`)
- `GET /links` — no key
- `GET /state` — requires `X-Team-Key: $CHIMERA_TEAM_KEY` (server-side only; never in browser)
- Tick-aware cache (~1.5 s minimum poll interval)
- If unreachable: Co-Pilot uses **neutral link state** and continues (no crash)

### 4.3 Three analytical models (classical ML — not LLMs)

Trained **only on CSVs** in `challenge p2/` via `npm run train:models`. Never trained on scrambled live `/state` during early competition days.

#### Congestion — `predictCongestion`

\[
\text{penalty\_ms} = k \cdot (\text{load\_ratio})^{p}
\]

- Per-link power-law coefficients (global fallback if needed).
- **Hard block** if `status === "saturated"`, `self_reported_latency_ms === null`, or `load_ratio ≥ 0.90`.

#### Trust — `scoreTrust`

- Expected honest latency ≈ physics `Tv` + congestion penalty.
- Large positive delta (self-report much faster than expected) ⇒ spoofing.
- Known compromised from history: **Aegis-Elysium**, **Boreas-Fenix**.
- Unseen-vector decay if delta &gt; 15,000 ms.
- **Trust floor 0.5** ⇒ hard block.

#### Targeting — `scoreTargetingRisk`

\[
P(\text{jammed}) = \sigma(b_0 + b_1 \cdot \text{traffic\_share})
\]

- Higher share ⇒ higher jam risk ⇒ **route entropy** (don’t always use the same “optimal” path).

### 4.4 True Cost (how signals combine)

Per void hop:

\[
\begin{aligned}
\text{combined\_cost} =\;& \text{physics\_void\_ms} \\
&+ \text{congestion\_penalty\_ms} \\
&+ (1 - \text{trust}) \times 80{,}000 \\
&+ \text{targeting\_risk} \times 50{,}000 \\
&- \text{traffic\_share} \times 25{,}000
\end{aligned}
\]

| Constant | Value | Meaning |
|----------|-------|---------|
| `TRUST_SCALE_MS` | 80,000 | Cost of distrust |
| `TARGETING_SCALE_MS` | 50,000 | Cost of jam risk |
| `ENTROPY_BONUS_SCALE_MS` | 25,000 | Reward quiet links |
| `ROUTE_DIVERSIFICATION_EPSILON` | 5% | Near-optimal tie-break on targeting |

Dijkstra minimises path sum of `combined_cost`. Among paths within 5% of best, pick **lowest aggregate targeting risk**.

### 4.5 Sequential Co-Pilot agent

Required challenge flow (not a single static solve):

1. Parse NL → `{ origin_id, destination_id, payload }`.
2. Compute baseline physics path.
3. **For each hop:** fetch live state → congestion / trust / targeting tools → `combined_cost` → if unsafe, **reroute** from current node with updated blocklist.
4. Emit `link_evaluations[]` for final path, `final_latency_estimate_ms`, `explanation`.

### 4.6 Natural language & GenAI

| Layer | What it is | When used |
|-------|------------|-----------|
| **Rules** (default) | Regex `from/to`, arrows, fuzzy planet names, quoted/`send …` payload | Almost all demo utterances |
| **LLM fallback** (optional) | Ollama or Gemini extracts JSON intent | Only if rules confidence &lt; 0.6 **and** `CHIMERA_LLM_PROVIDER` set |

**Important for judges:** Generative AI is **only** this optional NL fallback. Core routing uses classical models + physics. Default is rules-only (CI offline, deterministic).

### 4.7 Anomaly / unseen-vector handling

`detectLinkAnomaly`:

1. Sanitize OOD telemetry (NaN, out-of-range load/traffic, unknown status).
2. Conservative scores: trust ≤ 0.3, targeting ≥ 0.85.
3. Flag in `explanation`.
4. Prefer steering away via cost; do not always hard-block (preserve deliverability).

### 4.8 Mandatory Council output

`Phase2RoutingReport` (Zod-validated before response):

```json
{
  "origin_id": "Aegis",
  "destination_id": "Caelum",
  "chosen_path": ["Aegis", "Dawn", "Caelum"],
  "link_evaluations": [
    {
      "link_id": "Aegis-Dawn",
      "predicted_congestion_penalty_ms": 1234.5,
      "trust_score": 0.92,
      "targeting_risk_score": 0.15,
      "combined_cost": 60123.4
    }
  ],
  "final_latency_estimate_ms": 120000.0,
  "explanation": "..."
}
```

Missing/malformed fields ⇒ disqualification risk — hence strict Zod + tests.

### 4.9 Transmit integration

`POST /api/transmit` with `"use_copilot": true`:

1. Compute Co-Pilot report.
2. Constrain physics router to edges on `chosen_path`.
3. Transmit; `hop_log` proves the intelligent path.
4. If constrained path undeliverable → fall back to physics baseline; `transmitted_on_copilot_path: false`.

---

## 5. APIs

| Endpoint | Purpose |
|----------|---------|
| `GET /api/health` | Version, config hash, engine, `models_loaded`, `chimera_reachable`, `last_tick` |
| `GET /api/universe` | Nodes, adjacency, edges (cached) |
| `POST /api/transmit` | Packet + route + `delivered_payload`; optional Co-Pilot |
| `POST /api/route` | NL or structured intent → `Phase2RoutingReport` |

All routes: structured errors, rate limit **120 req/min** per client, optional Sentry.

---

## 6. Dashboard (`/relic`)

Main demo surface (`src/components/telemetry/`):

| Panel | Purpose |
|-------|---------|
| **Co-Pilot Natural Language** | NL request, **parsed intent preview**, Route with Co-Pilot, Live Chaos Monitor |
| **Simulation scenarios** | Baseline / Hyper-Flare / Distortion / Blackout / Chaos |
| **Routing parameters** | Manual origin / destination / payload; Initiate Void Beam |
| **Quantum Radar (SpaceMap)** | Planets, trust-coloured links, chosen vs baseline path, targeting pulse |
| **Link Evaluations** | Council scores; click row → decision audit breakdown |
| **Intelligence Walkthrough** | Congestion / spoofed links / targeting findings |
| **Latency metrics** | Fiber / atmosphere / towers / void |
| **Codex Terminal** | Per-hop dialect + binary void stream |

**Live Chaos Monitor:** re-polls Co-Pilot ~every 4 s; on path change shows pivot banner; next beam follows updated path.

---

## 7. Team ownership

| Member | Primary | Secondary |
|--------|---------|-----------|
| **Inusha** | Co-Pilot agent, True Cost router, Chimera client, NL parser, APIs, schema | Architecture, decision-audit talking points |
| **Ruwan** | CSV loaders, three models, train/eval, intelligence report | Model smoke tests |
| **Anushka** | Dashboard UI, map overlays, evaluations panel, E2E | Demo script / presentation |

**Integration rule:** Ruwan exposes pure `predict*` / `score*` functions. Inusha wires them as agent tools. Anushka consumes report JSON only — no model calls from the browser.

---

## 8. Tech stack & quality

| Layer | Choice |
|-------|--------|
| Framework | Next.js 16 (App Router), React 19, TypeScript |
| Styling | Tailwind CSS |
| Validation | Zod (Council schema) |
| Tests | Vitest (~216 unit), Playwright (7 E2E) |
| Observability | Structured API logs; optional Sentry |
| Deploy | Vercel (+ Docker standalone image) |
| CI | GitHub Actions: lint, typecheck, coverage, build, E2E, Docker |

Commands:

```bash
npm install
npm run dev                 # http://localhost:3000/relic
npm test
npm run test:e2e
npm run train:models
npm run evaluate:models
npm run build && npm start
```

---

## 9. Environment variables

| Variable | Required? | Purpose |
|----------|-----------|---------|
| `CHIMERA_TEAM_KEY` | For live Co-Pilot | Chimera `/state` auth (server only) |
| `CHIMERA_API_BASE_URL` | Optional | Default `https://chimera.launch26.space` |
| `UNIVERSE_CONFIG_PATH` | Optional | Override universe JSON |
| `CHIMERA_LLM_PROVIDER` | Optional | `off` / `ollama` / `gemini` |
| `GEMINI_API_KEY` / `OLLAMA_BASE_URL` | If LLM on | Provider credentials / host |
| `SENTRY_DSN` / `NEXT_PUBLIC_SENTRY_DSN` | Optional | Error tracking |
| `SENTRY_AUTH_TOKEN` | Optional | Source-map upload at build |

See [`.env.example`](.env.example). Never commit real keys.

---

## 10. Evaluation trials (what judges run)

| Trial | What to show |
|-------|----------------|
| **System Init** | Extended config (12 links), `/api/health` models + Chimera tick |
| **Intelligence Walkthrough** | Three findings from models / Intelligence panel |
| **Live NL Route** | NL → report → map overlays → evaluations |
| **Chaos Severance Pivot** | Live monitor + path pivot banner |
| **Decision Audit** | Explain any `link_evaluations` row without reading code |

Script: [`DEMO_SCRIPT.md`](DEMO_SCRIPT.md).

---

## 11. Key numbers (memorise)

| Item | Value |
|------|-------|
| Interplanetary links | 12 |
| Congestion MAE | ~22.7 s (5,743 ticks) |
| Trust precision / recall / F1 | 92% / 70% / 0.80 |
| Spoofed links | Aegis-Elysium, Boreas-Fenix |
| Trust floor | 0.5 |
| Saturation | load_ratio ≥ 0.90 |
| True Cost scales | 80k / 50k / 25k ms |
| Diversification ε | 5% |
| Anomaly conservative | trust 0.3 · targeting 0.85 |
| Tests | 216 unit + 7 E2E |

---

## 12. Project structure (simplified)

```
src/
  lib/relic/           # Phase 1 engine
  lib/chimera/         # Phase 2 Co-Pilot
  lib/api/             # validation, rate limit, logging
  components/telemetry/# dashboard
  app/api/             # health, universe, transmit, route
  app/relic/           # dashboard page
  cli/relic.ts         # terminal demo
challenge/
  INTELLIGENCE_REPORT.md
  DECISION_AUDIT.md
  PHASE2_PROJECT_REPORT.md
challenge p2/          # CSVs + extended universe-config
e2e/                   # Playwright
DEMO_SCRIPT.md
PROJECT_OVERVIEW.md    # this file
QA_SESSION_GUIDE.md
```

---

## 13. End-to-end story (one paragraph)

A judge types *“Send Hello world from Aegis to Caelum.”* The hybrid parser extracts intent (rules; LLM only if needed). The Co-Pilot pulls Chimera `/state`, scores each candidate link with congestion, trust, and targeting models, builds a True Cost, and either runs a global True Cost Dijkstra or walks a baseline path hop-by-hop, blocking saturated/spoofed links and rerouting. It returns a Zod-valid `Phase2RoutingReport`. The dashboard paints the path and scores. **Initiate Void Beam** with Co-Pilot context transmits along that path so the `hop_log` is the receipt. If Chimera saturates a hop mid-session, Live Chaos Monitor re-routes and shows a pivot — zero packet loss on the next send.

---

*Stack Kings — LAUNCH 26 · Relic Ring Protocol + Chimera Co-Pilot*
