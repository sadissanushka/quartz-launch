# `src/lib/api/constants.ts` — API Constants

**Tags:** `#constants` `#api` `#rate-limit`  
**See also:** [[api-rate-limit]] · [[api-validate-transmit]] · [[api-universe]]

---

## Constants

| Constant                     | Value               | Used In                                    |
| ---------------------------- | ------------------- | ------------------------------------------ |
| `MAX_PAYLOAD_CHARS`          | `10 * 1024` (10 KB) | Transmit + route request validation        |
| `MAX_BLOCKED_NODES`          | `50`                | Transmit request validation                |
| `MAX_BLOCKED_EDGES`          | `100`               | Transmit request validation                |
| `RATE_LIMIT_MAX_REQUESTS`    | `120`               | Rate limiter (120 req/min)                 |
| `RATE_LIMIT_WINDOW_MS`       | `60_000`            | Rate limiter window (1 minute)             |
| `UNIVERSE_CACHE_MAX_AGE_SEC` | `3600`              | Cache-Control for `/api/universe` (1 hour) |

---

## Notes

- Rate limit is **120 requests/minute per client IP** — generous enough for interactive testing but prevents hammering.
- `MAX_BLOCKED_NODES = 50` and `MAX_BLOCKED_EDGES = 100` prevent denial-of-service via huge blocked lists that would slow Dijkstra.
- All limits are returned as structured error responses (not server errors) for easy client handling.

---

Back to [[00 - Index]]
