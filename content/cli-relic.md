# `src/cli/relic.ts` — Terminal Demo CLI

**Tags:** `#cli` `#demo` `#milestone-M1` `#milestone-M2` `#milestone-M3` `#milestone-M4`  
**See also:** [[relic-engine]] · [[relic-transmission]] · [[relic-resilience]] · [[relic-server-universe]]

---

## Purpose

Terminal demo for the Relic Ring Protocol covering milestones **M1 through M4**. Can be run interactively or as a scripted walkthrough.

---

## Usage

```bash
npm run relic                           # scripted M1-M4 demo
npm run relic -- init                   # M1: print universe topology
npm run relic -- send Aegis Caelum "Hi"               # M2/M3: single transmission
npm run relic -- send Aegis Caelum "Hi" --kill Boreas  # M4: with node failure
npm run relic -- send Aegis Caelum "Hi" --cut Aegis-Boreas,Boreas-Caelum  # link failure
```

---

## Commands

### `init` — M1: Universe Initialization

Prints:

- System name
- All planets with their codex base
- All reachable links (L ≤ Lmax) with void distance in millions of km

### `send <origin> <dest> [payload] [--kill ...] [--cut ...]` — M2/M3/M4

Flags:

- `--kill A,B` — comma-separated planet IDs to kill (node failures)
- `--cut A-B,C-D` — comma-separated link pairs to sever (edge failures)

Prints `printResult()`:

- Route path + total latency
- Fiber/tower/atmosphere/void breakdown
- Delivered payload (proof of codec correctness)
- Full `hop_log` with per-hop tower info, dialect, and cumulative latency

### `demo` — Scripted M1-M4 Walkthrough

```
1. Print universe (M1)
2. Send origin → destination (M2/M3)
3. Kill the first intermediate node
4. Re-send to show automatic rerouting (M4)
5. Revive the node
```

---

## Helper Functions

| Function                     | Description                              |
| ---------------------------- | ---------------------------------------- |
| `fmt(value)`                 | Format ms to 3 decimal places            |
| `printUniverse(engine)`      | Print topology                           |
| `printResult(label, result)` | Print full transmission result + hop_log |
| `parseList(value)`           | `"A,B,C"` → `["A", "B", "C"]`            |
| `parseEdges(value)`          | `"A-B,C-D"` → `[["A","B"], ["C","D"]]`   |
| `getFlag(args, name)`        | Extract `--flag value` from argv         |

---

## Design Notes

- The CLI uses `engine.network` ([[relic-resilience]]) for the demo so that `killNode`/`reviveNode` can be called fluently.
- `send` command uses `transmit()` directly (pure, no state) with the failure flags converted to `RouteOptions`.
- Error exit via `process.exit(1)` on unknown commands or missing arguments.

---

Back to [[00 - Index]]
