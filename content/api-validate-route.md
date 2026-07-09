# `src/lib/api/validate-route.ts` — Route Request Validation

**Tags:** `#validation` `#api` `#phase-2`  
**See also:** [[api-route]] · [[api-constants]] · [[api-errors]]

---

## Purpose

Validates and normalises the body of `POST /api/route`. Supports two formats and returns a typed discriminated union.

---

## Types

```ts
interface NlRouteRequest {
  mode: "nl";
  request: string; // NL routing request
}

interface StructuredRouteRequest {
  mode: "structured";
  origin_id: string;
  destination_id: string;
  payload: string;
}

type RouteRequest = NlRouteRequest | StructuredRouteRequest;
```

The `mode` discriminant lets the API route switch on the type:

```ts
if (validated.data.mode === "nl") {
  report = await routeWithCopilot(validated.data.request);
} else {
  report = await routeWithTrueCost(validated.data);
}
```

---

## `validateRouteBody(body: unknown)` — Decision Tree

```
body is not object → 400 VALIDATION_ERROR

body.request is non-empty string:
  → length > MAX_PAYLOAD_CHARS → 413 PAYLOAD_TOO_LARGE
  → ok: { mode: "nl", request: trimmed }

body.origin_id + body.destination_id are strings:
  → either empty → 400 VALIDATION_ERROR
  → origin === destination → 400 VALIDATION_ERROR
  → payload.length > MAX_PAYLOAD_CHARS → 413 PAYLOAD_TOO_LARGE
  → ok: { mode: "structured", origin_id, destination_id, payload }

neither pattern matches → 400 VALIDATION_ERROR with usage hint
```

---

## Design Notes

- `payload` defaults to `""` when not provided in structured mode — makes it optional.
- `origin_id` / `destination_id` are trimmed of whitespace.
- The error for "neither format matched" includes a helpful usage hint in the message.

---

Back to [[00 - Index]]
