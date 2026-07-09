# `src/lib/chimera/parser/hybrid.ts` — Hybrid NL Parser

**Tags:** `#nlp` `#parser` `#phase-2` `#inusha`  
**See also:** [[chimera-parser-llm]] · [[chimera-constants]] · [[chimera-agent-copilot]]

---

## Purpose

Two-layer **natural-language parser** that converts free-form routing requests into structured `{ origin_id, destination_id, payload }`. A fast rules layer handles common patterns; an optional LLM fallback handles complex inputs.

**Owner:** Inusha

---

## `parseRoutingRequest(text, nodeIds, options?) → Promise<StructuredIntent>`

```ts
interface StructuredIntent {
  origin_id: string;
  destination_id: string;
  payload: string;
  confidence: number; // 0–1
  source: "rules" | "llm";
}
```

### Layer Priority

```
text → tryRules(text, nodeIds)
  ├─ success (confidence >= 0.6) → return { source: "rules" }
  └─ fail → resolveConfiguredLlm() || options.llm
              ├─ LLM available → llm(text, nodeIds)
              │   ├─ success (confidence >= 0.6) → return { source: "llm" }
              │   └─ fail / low confidence → fall through
              └─ No LLM → throw "Could not confidently parse..."
```

---

## Rules Layer

### `extractEndpoints(text, nodeIds)` — Three Strategies

1. **"from X to Y" pattern:**

   ```regex
   /\bfrom\s+([A-Za-z][\w-]*)\s+to\s+([A-Za-z][\w-]*)/i
   ```

2. **Arrow pattern:**

   ```regex
   /\b([A-Za-z][\w-]*)\s*(?:->|→)\s*([A-Za-z][\w-]*)/
   ```

3. **Planet name scan:** Find all mentioned planet IDs; if ≥ 2, use their text-order as origin/dest (confidence: 0.75)

### `resolvePlanetName(raw, nodeIds)` — Fuzzy Matching

1. Exact case-insensitive match → confidence 0.98
2. Levenshtein distance ≤ 1 (for short names) or ≤ 2 (longer names) → confidence 0.82 / 0.68

```ts
function levenshtein(a, b): number; // O(|a|×|b|) DP implementation
```

### `extractPayload(text)` — Three Strategies

1. Quoted content: `"message"` or `'message'`
2. `payload: ...` field
3. `send <message> from` pattern

---

## LLM Fallback

Disabled by default (`CHIMERA_LLM_PROVIDER` unset). Enabled via:

- `options.llm` — inject a specific `LlmParseFn`
- `CHIMERA_LLM_PROVIDER=ollama|gemini` — use a configured provider
- `options.llm = null` — force rules-only (no LLM even if configured)

---

## Confidence Thresholds

| Confidence | Decision                                               |
| ---------- | ------------------------------------------------------ |
| `>= 0.98`  | Exact planet name match                                |
| `>= 0.82`  | 1-character typo                                       |
| `>= 0.68`  | 2-character typo                                       |
| `>= 0.60`  | Parser threshold — returned; below this, defers to LLM |
| `< 0.60`   | Rejected by rules layer                                |

> [!TIP]
> Pass `options.llm = null` in tests to keep the parser deterministic and offline.

---

## Example Inputs

| Input                               | Detected Strategy   | Origin | Dest   |
| ----------------------------------- | ------------------- | ------ | ------ |
| `"Send hello from Aegis to Caelum"` | from-to             | Aegis  | Caelum |
| `"Boreas -> Fenix"`                 | arrow               | Boreas | Fenix  |
| `"Route via Aegis and Dawn"`        | planet scan         | Aegis  | Dawn   |
| `"aEGIS → cAELUM"`                  | arrow + exact match | Aegis  | Caelum |

---

Back to [[00 - Index]]
