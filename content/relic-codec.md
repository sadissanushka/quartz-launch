# `src/lib/relic/codec.ts` — Production Codec

**Tags:** `#encoding` `#codec` `#phase-1`  
**See also:** [[relic-contracts]] · [[relic-transmission]] · [[relic-engine]]

---

## Purpose

The production implementation of the [[relic-contracts#Codec|Codec interface]]. Handles the full `raw payload → codex digits → binary stream → void → destination` pipeline.

---

## Class: `RelicCodec`

Implements `Codec` with 6 methods:

### ASCII Helpers

```ts
toAscii(payload: string): number[]    // charCodeAt each character
fromAscii(bytes: number[]): string    // String.fromCharCode each byte
```

### Codex Encoding

```ts
encodeToCodex(asciiBytes: number[], base: number): EncodedPayload
// Each byte.toString(base).toUpperCase()
// Example: ASCII 72 in base 5 → "242"

decodeFromCodex(encoded: EncodedPayload): number[]
// parseInt(digit, base) per digit
```

### Binary Serialization

```ts
serializeToBinary(encoded: EncodedPayload): string
// 1. Join digit strings with DIGIT_DELIMITER (" ")
// 2. Emit each character of the joined string as 8-bit group
// e.g. "242 401" → each char → binary string

deserializeFromBinary(stream: string, base: number): EncodedPayload
// 1. Parse 8-bit groups back to characters
// 2. Split on DIGIT_DELIMITER to recover digit strings
// 3. Validate each digit is valid in base
```

---

## Binary Stream Protocol

```
Codex digits:  ["242", "401", ...]
               ↓ join(" ")
String repr:   "242 401 ..."
               ↓ charCode each char → 8-bit binary
Binary stream: "00110010 00110100 00110010 ..."
```

The space `" "` is an **unambiguous delimiter** since codex digits only use `[0-9A-Z]`. This makes the stream fully reversible.

> [!NOTE]
> This matches the encoding teammate's `toBinaryStream` convention on `pasindu-dev` — both use space-separated digit strings encoded as 8-bit ASCII groups.

---

## Validation

- `assertBase(base)` — base must be integer in [2, 36] (JavaScript's `parseInt`/`toString` range).
- `parseDigit(digit, base)` — `parseInt` must return a non-NaN value; otherwise throws.
- `deserializeFromBinary` — stream length must be a multiple of 8.

---

## Design Notes

- Digits are uppercased (`toUpperCase()`) for consistency — e.g. base-16 uses `A-F` not `a-f`.
- The codec is stateless — `createRelicCodec()` just `new RelicCodec()`.
- The production codec replaced the stub (`stubs/codec.stub.ts`) which was the placeholder during development.

---

Back to [[00 - Index]]
