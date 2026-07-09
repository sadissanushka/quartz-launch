# `src/lib/chimera/index.ts` — Public Barrel (Chimera Module)

**Tags:** `#barrel` `#exports` `#phase-2`  
**See also:** All chimera files

---

## Purpose

The **public surface of the Chimera Co-Pilot module**. All external code (API routes, dashboard, tests) should import from `@/lib/chimera` via this barrel. Internal details in subdirectories are intentionally **not** re-exported.

---

## What's Exported

| Export                                                                                   | Source                      | Description                    |
| ---------------------------------------------------------------------------------------- | --------------------------- | ------------------------------ |
| Types: `InterplanetaryLink`, `ChimeraLinkState`, `LinkEvaluation`, `Phase2RoutingReport` | `./types`                   | Core domain types              |
| `canonicalLinkId`                                                                        | `./link-id`                 | Alphabetical link ID generator |
| `routeWithCopilot`                                                                       | `./agent/copilot`           | Phase 2 Co-Pilot entry point   |
| `routeWithTrueCost`                                                                      | `./router/true-cost-router` | True Cost router               |
| `ChimeraClient`, `chimeraClient`, `getLastChimeraTick`                                   | `./client`                  | API client & singleton         |
| `areModelsLoaded`, `getChimeraHealthSnapshot`, `probeChimeraReachable`                   | `./health`                  | Health probes                  |
| `evaluateLink`, `neutralLinkState`, `detectLinkAnomaly`                                  | `./link-evaluation`         | Link scoring                   |
| `assertValidRoutingReport`, schemas, `ReportValidationError`                             | `./report-schema`           | Output validation              |
| `predictCongestion`, `scoreTrust`, `scoreTargetingRisk`                                  | `./models/*`                | Sub-model APIs                 |
| `parseRoutingRequest`                                                                    | `./parser/hybrid`           | NL parser                      |
| `createOllamaParser`, `createGeminiParser`, `resolveConfiguredLlm`                       | `./parser/llm`              | LLM factories                  |

---

## Design Notes

- The barrel is the **only stable import path**. API routes and components never import from deep paths like `@/lib/chimera/agent/copilot` — they always use the barrel.
- Test files may import internals directly, but production code should not.
- Phase groupings in comments (Phase 2, Phase 3, Phase 5) correspond to challenge milestones.

---

Back to [[00 - Index]]
