# `src/lib/api/rate-limit.ts` — In-Memory Rate Limiter

**Tags:** `#rate-limit` `#api` `#security`  
**See also:** [[api-constants]] · [[api-errors]] · [[api-route]] · [[api-transmit]]

---

## Purpose

Simple **fixed-window rate limiter** keyed by client IP. Applied to mutating endpoints (`/api/route`, `/api/transmit`) to prevent abuse.

---

## Algorithm: Fixed Window Counter

```ts
const buckets = new Map<string, { count: number; resetAt: number }>();
```

```
checkRateLimit(clientKey):
  now = Date.now()
  bucket = buckets.get(clientKey)

  if !bucket || now >= bucket.resetAt:
    → reset to { count: 1, resetAt: now + WINDOW_MS }
    → allow

  if bucket.count >= MAX_REQUESTS:
    → deny (429)

  bucket.count++
  → allow
```

Window: **60 seconds** | Max: **120 requests/window**

---

## Client Key Derivation

```ts
function getClientKey(request: Request): string {
  return (
    request.headers.get("x-forwarded-for")?.split(",")[0]?.trim() ||
    request.headers.get("x-real-ip") ||
    "anonymous"
  );
}
```

Reads from standard proxy headers (Vercel, Nginx, Cloudflare all set these). Falls back to `"anonymous"` when no IP header is present (e.g. local dev).

> [!WARNING]
> All anonymous requests share a single `"anonymous"` bucket. This means local development or test environments that don't set proxy headers count against the same limit. Use `resetRateLimitStore()` in tests.

---

## `enforceRateLimit(request) → Response | null`

```ts
function enforceRateLimit(request: Request): Response | null {
  if (!checkRateLimit(getClientKey(request))) {
    return apiErrorResponse(429, "RATE_LIMITED", "Too many requests...");
  }
  return null; // proceed
}
```

Usage pattern in routes:

```ts
const limited = enforceRateLimit(request);
if (limited) return limited;
```

---

## Limitations

- **Single-instance only** — in-memory, not shared across processes. Multiple Vercel serverless instances each have their own bucket store.
- No **Redis/Upstash** integration — suitable for a challenge demo; a production system would need a distributed store.
- `resetRateLimitStore()` is exported for test cleanup.

---

Back to [[00 - Index]]
