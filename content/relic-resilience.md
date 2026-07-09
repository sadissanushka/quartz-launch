# `src/lib/relic/resilience.ts` — Failure-Aware Network Controller

**Tags:** `#resilience` `#chaos` `#phase-1` `#milestone-M4`  
**See also:** [[relic-router]] · [[relic-transmission]] · [[relic-engine]]

---

## Purpose

Stateful wrapper over the universe that tracks **node and link failures**. Routes are recomputed on every `send()` against the current failure set, enabling live rerouting around dead zones. Powers the M4 chaos demonstration.

---

## Class: `ResilientNetwork`

```ts
new ResilientNetwork(universe, geometry, codec);
```

### State

- `failedNodes: Set<string>` — planets that are offline
- `failedLinks: Map<edgeKey, [a, b]>` — severed links

### Methods

| Method                        | Description                                         |
| ----------------------------- | --------------------------------------------------- |
| `killNode(id)`                | Mark planet offline; all subsequent routes avoid it |
| `reviveNode(id)`              | Bring planet back online                            |
| `killLink(a, b)`              | Sever the undirected link between two planets       |
| `reviveLink(a, b)`            | Restore a severed link                              |
| `reset()`                     | Clear all failures                                  |
| `isNodeFailed(id)`            | Check if a planet is offline                        |
| `isLinkFailed(a, b)`          | Check if a link is severed                          |
| `status()`                    | Snapshot: `{ failed_nodes, failed_links }`          |
| `route(origin, dest)`         | Compute route with current failures                 |
| `send(origin, dest, payload)` | Transmit with current failures                      |

---

## Design Notes

### Immutable Universe

The `ResilientNetwork` wraps a frozen `Universe` object — it never mutates the universe. Failures are tracked separately in the `failedNodes`/`failedLinks` sets.

### Failure-as-Options

Failures are passed to [[relic-router]]'s `RouteOptions` on every call:

```ts
private currentOptions(): RouteOptions {
  return {
    blockedNodes: [...this.failedNodes],
    blockedEdges: [...this.failedLinks.values()],
  };
}
```

This means `ResilientNetwork` is just a thin stateful adapter over the pure functions in `router.ts` and `transmission.ts`.

### Automatic Rerouting

Because `route()` and `send()` use the current failure set on every call, a `killNode` immediately affects the next route computation — no explicit "invalidate cache" step needed.

---

## M4 Demo Flow

```
1. engine.network.send(A, C, "Hello")       → route A → B → C
2. engine.network.killNode("B")             → B is now offline
3. engine.network.send(A, C, "Hello")       → automatically routes A → D → C (or undeliverable)
4. engine.network.reviveNode("B")           → restored
```

---

Back to [[00 - Index]]
