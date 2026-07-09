# `src/lib/chimera/client.ts` — Chimera API HTTP Client

**Tags:** `#http-client` `#caching` `#server-only` `#phase-2`  
**See also:** [[chimera-health]] · [[chimera-agent-copilot]] · [[chimera-router-true-cost]]

---

## Purpose

HTTP client for the **Chimera external API** with tick-aware in-memory caching. Prevents spamming the API while still providing reasonably fresh data to the routing logic.

---

## Endpoints

| Method       | Endpoint     | Auth                | Description                        |
| ------------ | ------------ | ------------------- | ---------------------------------- |
| `getLinks()` | `GET /links` | None                | Static link topology               |
| `getState()` | `GET /state` | `X-Team-Key` header | Live link states with current tick |

---

## Class: `ChimeraClient`

### Constructor Options

```ts
interface ChimeraClientOptions {
  baseUrl?: string; // default: CHIMERA_API_BASE_URL env or "https://chimera.launch26.space"
  teamKey?: string; // default: CHIMERA_TEAM_KEY env
  fetchFn?: ChimeraFetchFn; // injectable for tests
  pollIntervalMs?: number; // default: CHIMERA_POLL_INTERVAL_MS (1500ms)
}
```

### Cache Behaviour (`getState()`)

```
getState(force = false):
  if !force && cachedState && age < pollIntervalMs → return cachedState
  if !force && inflightState → return inflightState  ← deduplication
  inflightState = fetchStateFromApi()
  await → update cachedState, lastFetchMs, linksById
  inflightState = undefined
  return state
```

Key features:

1. **TTL cache** — reuses state if < 1500ms old
2. **In-flight dedup** — multiple simultaneous callers share one network request
3. **`force=true`** bypasses both for test/debug needs

### Other Methods

| Method                 | Description                                    |
| ---------------------- | ---------------------------------------------- |
| `getLastTick()`        | Last known Chimera simulation tick from cache  |
| `getLinkState(linkId)` | O(1) lookup of live link state by canonical ID |
| `clearCache()`         | Reset cache + linksById map (for tests)        |

---

## Response Parsing (Defensive)

Both `parseLinksResponse` and `parseStateResponse` use `isRecord()` type guards + field-by-field checks:

```ts
function parseStateResponse(body: unknown): ChimeraStateResponse {
  if (!isRecord(body) || typeof body.tick !== "number" || !Array.isArray(body.links)) {
    throw new Error("Malformed Chimera /state response...");
  }
  for (const link of body.links) {
    // validate each link field individually...
  }
  return body as unknown as ChimeraStateResponse;
}
```

This matches the Phase 5 hardening requirement — malformed external data must not crash the routing engine.

---

## Process Singleton

```ts
export const chimeraClient = new ChimeraClient();
```

A **module-level singleton** is exported for use by routing code. Tests can instantiate separate clients with a mock `fetchFn`.

---

## Error Handling

| Condition                  | Error                                                |
| -------------------------- | ---------------------------------------------------- |
| `CHIMERA_TEAM_KEY` not set | `"CHIMERA_TEAM_KEY is required..."`                  |
| HTTP 401                   | `"Chimera /state rejected the team key (HTTP 401)."` |
| Other HTTP error           | `"Chimera /state failed with HTTP N."`               |
| Malformed response         | `"Malformed Chimera /state response: ..."`           |

The API route [[api-route]] catches the `CHIMERA_TEAM_KEY` message and returns HTTP 503.

---

Back to [[00 - Index]]
