# `src/lib/relic/engine.ts` — Composition Root

**Tags:** `#composition` `#dependency-injection` `#phase-1`  
**See also:** [[relic-contracts]] · [[relic-config]] · [[relic-resilience]] · [[relic-server-universe]]

---

## Purpose

The **single wiring point** for the entire engine. All modules are assembled here; to swap teammate implementations, only this file needs to change.

---

## `createEngine(rawConfig: unknown) → Engine`

```ts
interface Engine {
  universe: Universe; // parsed config
  geometry: GeometryProvider; // geometry module
  codec: Codec; // encoding module
  graph: NetworkGraph; // adjacency graph
  network: ResilientNetwork; // stateful failure controller
}
```

### Wiring Sequence

```
rawConfig (unknown JSON)
  ↓ parseUniverseConfig()      → Universe
  ↓ createStubGeometryProvider() → GeometryProvider  ← swap here
  ↓ createRelicCodec()           → Codec             ← swap here
  ↓ buildNetworkGraph()          → NetworkGraph
  ↓ new ResilientNetwork()       → ResilientNetwork
→ Engine { universe, geometry, codec, graph, network }
```

---

## Integration Swap Points

```ts
// ===== Teammate integration swap point =====
const geometry: GeometryProvider = createStubGeometryProvider(universe.metadata);
const codec: Codec = createRelicCodec();
// ============================================
```

- **Geometry stub** — `createStubGeometryProvider` is a complete, working implementation. It can be replaced with the mapping teammate's module when it lands.
- **Production codec** — `createRelicCodec()` is the real codec (not a stub). The stub codec is still available in `stubs/codec.stub.ts` for isolated testing.

> [!IMPORTANT]
> Replace only these two lines at merge time. All other engine behavior — routing, transmission, resilience — automatically uses the new implementations via the interface contracts.

---

## Server Singleton

[[relic-server-universe]] wraps `createEngine` to provide a **cached singleton** for API routes. `createEngine` itself is stateless (no cache) and can be called directly in tests.

---

Back to [[00 - Index]]
