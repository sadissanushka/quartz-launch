# `src/lib/api/errors.ts` — API Error Utilities

**Tags:** `#api` `#errors`  
**See also:** [[api-constants]] · [[api-route]] · [[api-transmit]]

---

## Purpose

Centralised **error response factory** and error code enum for all API routes.

---

## `ApiErrorCode`

```ts
type ApiErrorCode =
  | "INVALID_JSON"
  | "VALIDATION_ERROR"
  | "PAYLOAD_TOO_LARGE"
  | "BLOCKED_LIST_TOO_LARGE"
  | "ENGINE_ERROR"
  | "RATE_LIMITED"
  | "PARSER_ERROR"
  | "ROUTING_ERROR"
  | "CHIMERA_UNAVAILABLE";
```

---

## `apiErrorResponse(status, code, message) → Response`

```ts
function apiErrorResponse(
  status: number,
  code: ApiErrorCode,
  message: string,
): Response {
  return Response.json({ error: message, code }, { status });
}
```

All error responses have the consistent shape:

```json
{ "error": "human-readable message", "code": "MACHINE_READABLE_CODE" }
```

Clients can programmatically switch on `code` while humans can read `error`.

---

## Usage Pattern

```ts
// In API routes:
return apiErrorResponse(400, "VALIDATION_ERROR", "origin_id must be non-empty.");
return apiErrorResponse(503, "CHIMERA_UNAVAILABLE", "CHIMERA_TEAM_KEY is required...");
return apiErrorResponse(429, "RATE_LIMITED", "Too many requests.");
```

---

Back to [[00 - Index]]
