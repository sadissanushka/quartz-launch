# 🌌 Stack Kings — Relic Ring Protocol · Code Review Vault

> Full code review of every source file in the `stack-kings` project.  
> Organised for **Obsidian** — all internal links use `[[wikilinks]]`.

---

## 🗺️ Architecture Overview

```
stack-kings/
├── src/
│   ├── app/                   # Next.js App Router (pages + API routes)
│   │   ├── api/
│   │   │   ├── health/        # GET  /api/health
│   │   │   ├── route/         # POST /api/route   (Co-Pilot routing)
│   │   │   ├── transmit/      # POST /api/transmit
│   │   │   ├── universe/      # GET  /api/universe
│   │   │   └── debug/         # Sentry debug trigger
│   │   ├── layout.tsx
│   │   └── page.tsx
│   ├── cli/                   # Terminal demo (milestones M1-M4)
│   ├── components/
│   │   ├── observability/     # Sentry error boundary components
│   │   └── telemetry/         # Dashboard UI components
│   └── lib/
│       ├── api/               # Shared API utilities (rate-limit, validate, errors)
│       ├── chimera/           # Phase 2 — Chimera Co-Pilot engine
│       │   ├── agent/         # Co-Pilot orchestrator (copilot.ts)
│       │   ├── models/        # Analytical sub-models (congestion, trust, targeting)
│       │   ├── parser/        # NL parser (rules + LLM fallback)
│       │   ├── router/        # True Cost router
│       │   └── csv/           # CSV data loaders
│       ├── observability/     # Sentry integration
│       └── relic/             # Phase 1 — core physics engine
│           ├── server/        # Node-only universe loader
│           └── stubs/         # Geometry / codec stubs
```

---

## 📂 File Index

### 🧬 Core Types & Config

- [[relic-types]] — `src/lib/relic/types.ts`
- [[relic-contracts]] — `src/lib/relic/contracts.ts`
- [[relic-config]] — `src/lib/relic/config.ts`
- [[relic-version]] — `src/lib/version.ts`

### ⚛️ Physics Engine (Phase 1 — `src/lib/relic/`)

- [[relic-latency]] — `latency.ts`
- [[relic-graph]] — `graph.ts`
- [[relic-router]] — `router.ts`
- [[relic-codec]] — `codec.ts`
- [[relic-transmission]] — `transmission.ts`
- [[relic-resilience]] — `resilience.ts`
- [[relic-engine]] — `engine.ts`
- [[relic-server-universe]] — `server/universe.ts`

### 🤖 Chimera Co-Pilot (Phase 2 — `src/lib/chimera/`)

- [[chimera-index]] — `index.ts` (public barrel)
- [[chimera-types]] — `types.ts`
- [[chimera-constants]] — `constants.ts`
- [[chimera-client]] — `client.ts`
- [[chimera-health]] — `health.ts`
- [[chimera-link-id]] — `link-id.ts`
- [[chimera-link-evaluation]] — `link-evaluation.ts`
- [[chimera-report-schema]] — `report-schema.ts`
- [[chimera-routing-report]] — `routing-report.ts`
- [[chimera-agent-copilot]] — `agent/copilot.ts`
- [[chimera-router-true-cost]] — `router/true-cost-router.ts`
- [[chimera-models-congestion]] — `models/congestion.ts`
- [[chimera-models-trust]] — `models/trust.ts`
- [[chimera-models-targeting]] — `models/targeting.ts`
- [[chimera-parser-hybrid]] — `parser/hybrid.ts`
- [[chimera-parser-llm]] — `parser/llm.ts`

### 🌐 API Routes (`src/app/api/`)

- [[api-health]] — `GET /api/health`
- [[api-universe]] — `GET /api/universe`
- [[api-route]] — `POST /api/route`
- [[api-transmit]] — `POST /api/transmit`

### 🛡️ API Utilities (`src/lib/api/`)

- [[api-constants]] — `constants.ts`
- [[api-errors]] — `errors.ts`
- [[api-rate-limit]] — `rate-limit.ts`
- [[api-validate-route]] — `validate-route.ts`
- [[api-validate-transmit]] — `validate-transmit.ts`

### 🖥️ App Shell (`src/app/`)

- [[app-layout]] — `layout.tsx`

### 💻 CLI (`src/cli/`)

- [[cli-relic]] — `relic.ts`

---

## 🔗 Key Data Flow

```
POST /api/route (NL)
  └─→ [[chimera-agent-copilot]]
        ├─→ [[chimera-parser-hybrid]]   (NL → origin/destination)
        ├─→ [[chimera-client]]          (fetch live Chimera /state)
        ├─→ [[relic-router]]            (Dijkstra physics path)
        ├─→ [[chimera-link-evaluation]] (score each hop)
        └─→ [[chimera-report-schema]]   (validate output)

POST /api/route (structured)
  └─→ [[chimera-router-true-cost]]
        ├─→ [[chimera-client]]
        ├─→ [[chimera-link-evaluation]]
        └─→ [[chimera-models-congestion]] + [[chimera-models-trust]] + [[chimera-models-targeting]]
```

---

## 🏷️ Tags

`#stack-kings` `#relic-ring-protocol` `#phase-1` `#phase-2` `#chimera` `#code-review`
