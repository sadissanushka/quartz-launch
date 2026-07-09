# `src/lib/version.ts` — App Version

**Tags:** `#utility` `#version`  
**See also:** [[api-health]]

---

## Purpose

Single source of truth for the application version — pulled directly from `package.json` at build time.

---

## Code

```ts
import pkg from "../../package.json";

/** Application semver from package.json (used in UI and /api/health). */
export const APP_VERSION = pkg.version;
```

---

## Usage

- `GET /api/health` → includes `"version": APP_VERSION` in response
- Dashboard UI (if displayed)

Bumping `version` in `package.json` is all that's needed to update the version everywhere.

---

Back to [[00 - Index]]
