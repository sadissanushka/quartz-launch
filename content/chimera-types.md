# `src/lib/chimera/types.ts` — Chimera Type Re-exports

**Tags:** `#types` `#barrel` `#phase-2`  
**See also:** [[relic-types]] · [[chimera-index]]

---

## Purpose

A local type barrel inside `src/lib/chimera/` so that code within the Chimera subdirectory can import types from a short, local path without depending directly on the relic internals.

---

## Code

```ts
export type {
  InterplanetaryLink,
  ChimeraLinkState,
  LinkEvaluation,
  Phase2RoutingReport,
} from "../relic/types";

export { canonicalLinkId } from "./link-id";
```

---

## Why This Exists

Without this barrel, chimera files would need to write:

```ts
import type { ChimeraLinkState } from "../../relic/types"; // from deep inside chimera/
```

With this barrel:

```ts
import type { ChimeraLinkState } from "../types"; // cleaner, no upward coupling
```

This also means the canonical source of truth (`relic/types.ts`) stays unchanged — types are only re-exported here.

---

Back to [[00 - Index]]
