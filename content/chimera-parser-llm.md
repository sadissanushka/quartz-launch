# `src/lib/chimera/parser/llm.ts` — LLM Parser Fallback

**Tags:** `#llm` `#parser` `#ollama` `#gemini` `#server-only`  
**See also:** [[chimera-parser-hybrid]] · [[chimera-constants]]

---

## Purpose

**LLM-backed fallback** for the hybrid parser. Used only when the deterministic rules layer cannot confidently extract origin/destination. Supports two providers: local **Ollama** and **Google Gemini** REST API.

Server-side only — never import from client components.

---

## Provider Selection

```
CHIMERA_LLM_PROVIDER=ollama  → createOllamaParser()
CHIMERA_LLM_PROVIDER=gemini  → createGeminiParser()  (needs GEMINI_API_KEY)
(unset / "off")              → undefined (no LLM fallback, CI-safe)
```

```ts
export function resolveConfiguredLlm(): LlmParseFn | undefined;
```

---

## `LlmParseFn` Type

```ts
type LlmParseFn = (
  text: string,
  nodeIds: string[],
) => Promise<Omit<StructuredIntent, "source"> | null>;
```

Returns `null` when the LLM cannot parse (both providers handle this gracefully).

---

## Prompt Engineering

```ts
function buildPrompt(text, nodeIds): string {
  return [
    "You extract interplanetary routing intent from a user's message.",
    `Valid planet IDs (use these EXACT strings): ${nodeIds.join(", ")}.`,
    "Return ONLY a JSON object with keys:",
    '  "origin_id"      - the planet the message starts from',
    '  "destination_id" - the planet the message is sent to',
    '  "payload"        - the message content to transmit (empty string if none)',
    '  "confidence"     - your confidence 0..1',
    "If you cannot identify both planets, set confidence to 0.",
    `User message: ${JSON.stringify(text)}`,
  ].join("\n");
}
```

The valid planet IDs are injected explicitly — the LLM is constrained to exact ID strings, not free-form names.

---

## `createOllamaParser(options?)`

```ts
{
  baseUrl: OLLAMA_BASE_URL env || "http://localhost:11434",
  model: CHIMERA_LLM_MODEL env || "qwen3:4b",
  fetchFn: fetch
}
```

Calls `POST /api/chat` with `{ stream: false, think: false, format: "json" }`.

---

## `createGeminiParser(options?)`

```ts
{
  apiKey: GEMINI_API_KEY env (required),
  model: CHIMERA_LLM_MODEL env || "gemini-2.0-flash",
  fetchFn: fetch
}
```

Calls `POST generateContent` with `{ responseMimeType: "application/json", temperature: 0 }`.

---

## `normalizeIntent(raw, nodeIds)`

Validates the LLM JSON output:

- `origin_id` / `destination_id` must be in `nodeIds` (case-insensitive, trimmed)
- `payload` defaults to `""`
- `confidence` clamped to [0, 1], defaults to 0.7
- Returns `null` if origin = destination or either is missing

---

## Error Handling

Both providers:

- Throw on non-OK HTTP responses
- Return `null` on empty/unparseable JSON (not an exception — just "can't parse")
- The hybrid parser catches all exceptions and falls through to the "cannot parse" error

---

Back to [[00 - Index]]
