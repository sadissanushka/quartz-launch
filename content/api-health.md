# `src/app/api/health/route.ts` — GET /api/health

**Tags:** `#api` `#health` `#observability` `#phase-3`  
**See also:** [[chimera-health]] · [[relic-server-universe]]

---

## Purpose

**Liveness/readiness probe** for container orchestration and deploy verification. Returns the engine status, Chimera reachability, models state, and current simulation tick.

---

## Endpoint

```
GET /api/health
```

No authentication required. Calls are NOT rate-limited (probe traffic).

---

## Successful Response (HTTP 200)

```json
{
  "status": "ok",
  "version": "1.2.0",
  "config_path": "/app/challenge p2/universe-config.json",
  "config_hash": "sha256:abc123...",
  "engine_loaded": true,
  "models_loaded": true,
  "chimera_reachable": true,
  "last_tick": 87
}
```

| Field               | Source                            |
| ------------------- | --------------------------------- |
| `version`           | `APP_VERSION` from `package.json` |
| `config_path`       | `universeConfigPath()`            |
| `config_hash`       | SHA-256 of raw config bytes       |
| `engine_loaded`     | `getEngine()` succeeds            |
| `models_loaded`     | `areModelsLoaded()`               |
| `chimera_reachable` | `probeChimeraReachable()`         |
| `last_tick`         | `getLastChimeraTick()`            |

## Degraded Response (HTTP 503)

```json
{
  "status": "degraded",
  "version": "1.2.0",
  "config_path": "...",
  "engine_loaded": false,
  "models_loaded": false,
  "chimera_reachable": false,
  "last_tick": null,
  "error": "Could not load universe config: ..."
}
```

---

## Implementation Notes

- Uses `export const runtime = "nodejs"` — this is a Node.js route (not Edge runtime), needed for filesystem access.
- Errors are captured to Sentry via `captureApiError`.
- `getEngine()` is called to eagerly warm the engine cache at health-check time.

---

Back to [[00 - Index]]
