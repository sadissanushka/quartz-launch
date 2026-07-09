# `src/lib/relic/server/universe.ts` — Universe Loader & Engine Cache

**Tags:** `#server-only` `#singleton` `#filesystem` `#phase-1`  
**See also:** [[relic-engine]] · [[relic-config]] · [[api-health]] · [[api-universe]]

---

## Purpose

**Node.js-only** helpers for loading `universe-config.json` from disk and caching the engine instance across API route calls. Never import this in client components.

---

## Config File Discovery

### `universeConfigPath() → string`

Priority order:

1. `UNIVERSE_CONFIG_PATH` env var (if set and non-empty)
2. Scan candidate paths in order:
   - `challenge p2/universe-config.json` ← Phase 2 first
   - `challenge p1/universe-config.json`
   - `universe-config.json`
3. Fall back to `universe-config.json` in cwd (even if it doesn't exist)

### `DEFAULT_UNIVERSE_CONFIG_CANDIDATES`

```ts
[
  "challenge p2/universe-config.json",
  "challenge p1/universe-config.json",
  "universe-config.json",
];
```

> [!NOTE]
> Phase 2 config is preferred over Phase 1. Override with `UNIVERSE_CONFIG_PATH` env var for deployment flexibility.

---

## Functions

| Function               | Description                                                                                   |
| ---------------------- | --------------------------------------------------------------------------------------------- |
| `loadUniverseConfig()` | Reads and parses JSON from the discovered path; wraps filesystem errors as `RelicConfigError` |
| `universeConfigHash()` | SHA-256 hash of the raw config bytes — used in `/api/health` response for deploy verification |
| `getEngine()`          | Returns the cached engine, building it on first call                                          |
| `reloadEngine()`       | Force-rebuild from disk and update cache                                                      |
| `clearEngineCache()`   | Nullify the cache (for tests)                                                                 |

---

## Caching Strategy

```ts
let cachedEngine: Engine | undefined;

function getEngine(): Engine {
  if (!cachedEngine) {
    cachedEngine = createEngine(loadUniverseConfig());
  }
  return cachedEngine;
}
```

Simple module-level singleton — works fine for a single Next.js process. In serverless (Vercel) each cold start gets a fresh module; the cache lives only per process lifetime.

> [!WARNING]
> If `universe-config.json` changes at runtime, call `reloadEngine()` explicitly. The cache will not automatically detect file changes.

---

## Error Handling

`loadUniverseConfig()` converts filesystem errors into `RelicConfigError`:

```ts
throw new RelicConfigError(`Could not load universe config from "${path}": ${message}`);
```

API routes catch `RelicConfigError` specifically and return HTTP 500 with `ENGINE_ERROR`.

---

Back to [[00 - Index]]
