# `src/lib/chimera/link-id.ts` — Canonical Link ID

**Tags:** `#utility` `#phase-2`  
**See also:** [[relic-graph]] · [[chimera-link-evaluation]] · [[relic-config]]

---

## Purpose

Single-function module that produces **canonical, order-independent** interplanetary link identifiers.

---

## `canonicalLinkId(a, b) → string`

```ts
export function canonicalLinkId(a: string, b: string): string {
  return a < b ? `${a}-${b}` : `${b}-${a}`;
}
```

### Examples

```ts
canonicalLinkId("Boreas", "Aegis"); // → "Aegis-Boreas"
canonicalLinkId("Aegis", "Boreas"); // → "Aegis-Boreas"
```

The two inputs are **lexicographically sorted** before joining with `-`. This guarantees the same ID regardless of argument order.

---

## Why Two Separators?

| Convention                  | Separator   | Used In                                               |
| --------------------------- | ----------- | ----------------------------------------------------- |
| `canonicalLinkId`           | `-` (dash)  | Chimera API, `universe-config.json`, Council schema   |
| `edgeKey` ([[relic-graph]]) | `\|` (pipe) | Internal graph layer, `Set<string>` for blocked edges |

> [!IMPORTANT]
> Never mix these. The Chimera API's `link_id` field uses dashes. The router's `edgeKey` uses pipes for the blocked edge set. Using the wrong one will cause silent lookup misses.

---

## Validation

[[relic-config]] enforces during parsing that every `interplanetary_link.link_id` in the config equals `canonicalLinkId(planet_a, planet_b)`. Links with non-alphabetical IDs are rejected.

---

Back to [[00 - Index]]
