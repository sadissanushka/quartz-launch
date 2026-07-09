# `src/lib/api/validate-transmit.ts` — Transmit Request Validation

**Tags:** `#validation` `#api`  
**See also:** [[api-transmit]] · [[api-constants]] · [[api-errors]]

---

## Purpose

Validates and normalises the body of `POST /api/transmit`. Handles optional failure lists (M4 chaos) and the `use_copilot` flag.

---

## `TransmitRequest` Type

```ts
interface TransmitRequest {
  origin: string;
  destination: string;
  payload: string;
  options: RouteOptions; // { blockedNodes, blockedEdges }
  useCopilot: boolean;
}
```

---

## `validateTransmitBody(body: unknown)` — Validation Steps

```
1. body must be an object
2. origin, destination, payload must be strings
3. payload.length <= MAX_PAYLOAD_CHARS (10 KB)
4. blockedNodes = asStringArray(body.blockedNodes)
   → length <= MAX_BLOCKED_NODES (50)
5. blockedEdges = asEdgeList(body.blockedEdges)  ← [[a, b], ...]
   → length <= MAX_BLOCKED_EDGES (100)
6. useCopilot = (body.use_copilot === true)     ← strict equality
```

---

## Helper: `asStringArray(value)`

Returns only string items from an array; ignores non-strings silently. Safe for malformed input.

## Helper: `asEdgeList(value)`

Accepts `[[string, string], ...]` — filters out malformed items. An edge `[a, b]` is only added when both items are strings.

---

## Key Design: Lenient on Optional Fields

- Missing `blockedNodes` / `blockedEdges` → empty arrays (not an error)
- Missing `use_copilot` → `false`
- Non-string items in `blockedNodes` → silently filtered
- Missing `payload` → error (required field)

This makes it easy for clients to gradually add optional fields without breaking validation.

---

Back to [[00 - Index]]
