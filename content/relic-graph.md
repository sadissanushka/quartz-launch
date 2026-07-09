# `src/lib/relic/graph.ts` — Network Graph Builder

**Tags:** `#graph` `#topology` `#phase-1`  
**See also:** [[relic-router]] · [[relic-latency]] · [[relic-types]]

---

## Purpose

Builds the **adjacency graph** of the planet network based on the Lmax constraint. A direct void hop between two planets is only allowed when their void distance `L ≤ Lmax`.

---

## Public API

### `buildNetworkGraph(universe, geometry) → NetworkGraph`

Iterates all `N*(N-1)/2` planet pairs, computes void distance via the geometry provider, and marks each edge as reachable or not.

```ts
interface NetworkGraph {
  adjacency: Map<string, string[]>; // planet id → reachable neighbors
  edges: VoidEdge[]; // all pairs with metadata
}

interface VoidEdge {
  from: string;
  to: string;
  void_distance_km: number;
  within_lmax: boolean;
}
```

### `edgeKey(a, b) → string`

Stable **undirected** key for a planet pair: `a < b ? "${a}|${b}" : "${b}|${a}"`.  
Used throughout the codebase to identify edges in `Set<string>` for the blocked-edge tracking.

> [!NOTE]
> `edgeKey` uses `|` separator (graph layer). `canonicalLinkId` uses `-` separator (Chimera/API layer). Both sort alphabetically — never mix them.

---

## Algorithm

```
for i in [0, nodes.length):
  for j in [i+1, nodes.length):
    L = geometry.voidDistanceKm(nodes[i], nodes[j])
    if L <= Lmax:
      adjacency[i].push(j.id)
      adjacency[j].push(i.id)
    edges.push({ from: i.id, to: j.id, void_distance_km: L, within_lmax: L <= Lmax })
```

Time complexity: **O(N²)** for `N` planets.

---

## Usage in the Router

[[relic-router]] does **not** use the `NetworkGraph` directly for routing — instead it re-queries `geometry.voidDistanceKm` per neighbor during Dijkstra. `NetworkGraph` is primarily useful for:

- The `GET /api/universe` response (`graph.adjacency`, `graph.edges`)
- Dashboard visualization in [[chimera-router-true-cost]] and the SpaceMap component

---

Back to [[00 - Index]]
