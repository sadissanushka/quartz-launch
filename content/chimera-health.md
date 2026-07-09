# `src/lib/chimera/health.ts` — Chimera Health Probes

**Tags:** `#health` `#observability` `#server-only` `#phase-3`  
**See also:** [[chimera-client]] · [[api-health]]

---

## Purpose

Server-side health probes for the Chimera subsystem, used by `GET /api/health`.

---

## `areModelsLoaded() → boolean`

Checks that all three model JSON artifacts exist and are non-empty:

```ts
function areModelsLoaded(): boolean {
  // congestion.model.json must have at least one key
  // trust.model.json must have a "compromised_links" object
  // targeting.model.json must have at least one key
}
```

Models are JSON files imported at module load time — if the JSON is empty `{}` or missing the `compromised_links` key, models are considered not loaded.

---

## `probeChimeraReachable() → Promise<boolean>`

Non-blocking reachability check: calls `chimeraClient.getLinks()` (no auth) and returns `false` on any error:

```ts
try {
  await chimeraClient.getLinks();
  return true;
} catch {
  return false;
}
```

Uses the public `/links` endpoint specifically to avoid needing `CHIMERA_TEAM_KEY`.

---

## `getChimeraHealthSnapshot() → Promise<ChimeraHealthSnapshot>`

```ts
interface ChimeraHealthSnapshot {
  models_loaded: boolean;
  chimera_reachable: boolean;
  last_tick: number | null;
}
```

Full flow:

```
1. probeChimeraReachable()
2. If reachable: try getState() to get current tick (may fail without team key)
3. Return snapshot from models + reachability + getLastChimeraTick()
```

The `getState()` call is best-effort — it silently falls back to the last known tick (or `null`) if the team key is missing or Chimera is unreachable.

---

## Usage in `/api/health`

```json
{
  "status": "ok",
  "engine_loaded": true,
  "models_loaded": true,
  "chimera_reachable": true,
  "last_tick": 42
}
```

If the engine fails to load, the response is:

```json
{
  "status": "degraded",
  "engine_loaded": false,
  "models_loaded": false,
  "chimera_reachable": false,
  "last_tick": null
}
```

with HTTP 503.

---

Back to [[00 - Index]]
