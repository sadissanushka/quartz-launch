# `src/lib/relic/transmission.ts` — Packet Transmission

**Tags:** `#transmission` `#packet` `#phase-1` `#hop-log`  
**See also:** [[relic-router]] · [[relic-codec]] · [[relic-resilience]] · [[api-transmit]]

---

## Purpose

Orchestrates the **full transmission pipeline** — routing + codec pipeline + hop_log construction. This is where a raw string payload becomes a cryptographically-audited packet that proves transit through every planet in the route.

---

## `transmit()` — Main Function

```ts
transmit(
  universe: Universe,
  geometry: GeometryProvider,
  codec: Codec,
  originId: string,
  destinationId: string,
  payload: string,
  options?: RouteOptions,
): TransmissionResult
```

### Step-by-Step Pipeline

```
1. findShortestRoute()          → Route
2. If !deliverable              → Return undeliverable Packet
3. Walk hops (codec pipeline):
     For each hop i (planet[i] → planet[i+1]):
       a. encodeToCodex(ascii[i], nextPlanet.codex)    → digits
       b. serializeToBinary(digits)                     → stream
       c. deserializeFromBinary(stream, nextCodex)      → digits' (should match)
       d. decodeFromCodex(digits')                      → ascii[i+1]
4. Build hop_log[]:
     For each step i:
       - payload_ascii[i]
       - payload_dialect (current planet's codex)
       - next_hop_dialect (next planet's codex)
       - binary_stream (what crosses the void)
       - cumulative_latency_ms (running sum)
5. Return TransmissionResult
```

### `TransmissionResult`

```ts
{
  packet: Packet; // hop_log, status, etc.
  route: Route; // physics route details
  delivered_payload: string | null; // reconstructed at destination
}
```

---

## Codec Pipeline Detail

Each hop runs the full encode → serialize → deserialize → decode cycle:

```
ascii[i]
  ↓ encodeToCodex(ascii[i], nextNode.codex)
EncodedPayload { base: nextCodex, digits: [...] }
  ↓ serializeToBinary
binary_stream (string of 0s and 1s)
  ↓ deserializeFromBinary(stream, nextCodex)
EncodedPayload (same digits reconstructed)
  ↓ decodeFromCodex
ascii[i+1]  ← this is what arrives at the next planet
```

> [!TIP]
> Because the full encode/serialize/deserialize/decode round-trip is executed, `delivered_payload` is a **genuine** end-to-end proof of codec correctness, not a copy of the input.

---

## `hop_log` Construction

Each `HopLogEntry` captures:

- Tower entry/exit indices and ring segments traversed
- `payload_ascii` — payload as ASCII bytes at this planet
- `payload_dialect` — payload encoded in the planet's own codex
- `next_hop_dialect` — payload encoded for the next hop's codex
- `binary_stream` — the actual bit stream that crosses the void
- `cumulative_latency_ms` — running sum of internal + void latencies

The log is the "proof of route" — verifiable that every conversion was done correctly.

---

## Undeliverable Case

When the route is not deliverable:

- Returns a `Packet` with `status: "undeliverable"` and `undeliverable_reason`
- `delivered_payload` is `null`
- `hop_log` is empty `[]`

---

Back to [[00 - Index]]
