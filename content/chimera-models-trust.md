# `src/lib/chimera/models/trust.ts` — Trust / Reliability Model

**Tags:** `#model` `#trust` `#spoofing` `#phase-2` `#ruwan`  
**See also:** [[chimera-models-congestion]] · [[chimera-link-evaluation]] · [[chimera-constants]]

---

## Purpose

Scores how **trustworthy** a link's self-reported latency is. Detects when Chimera is under-reporting its own latency (spoofing). Trained from `link_telemetry.csv`.

**Owner:** Ruwan (Phase 1, section 1.3)

---

## Algorithm: `scoreTrust(linkState) → number [0, 1]`

```
1 = fully trustworthy
0 = confirmed Chimera spoofing footprint
```

### Step 1: Saturation Check

```ts
if (status === "saturated" || self_reported_latency_ms === null) return 0.0;
```

### Step 2: Compute Expected Latency

```
Tv = getTvForLink(planet_a, planet_b)   // physics void travel time
congestion = predictCongestion(linkState, Tv)
expectedLatency = Tv + congestion.penalty_ms
liveDelta = expectedLatency - self_reported_latency_ms
```

A positive `liveDelta` means the link claims it's faster than it should be → potential spoofing.

### Step 3: Score Based on Known Compromised Links

```ts
if (compromisedInfo) {
  // Known bad link: check if live delta matches historical spoofing pattern
  const ratio = liveDelta / compromisedInfo.mean_delta_ms;
  return clamp(0, 1, 1 - ratio);
  // ratio ≈ 1 → matches spoofing → trust ≈ 0
  // ratio ≤ 0 → behaving honestly → trust = 1
} else {
  // Historically honest: check for "Unseen Vector" attack
  if (liveDelta > 15_000) {
    const anomalyFactor = clamp(0, 1, (liveDelta - 15_000) / 30_000);
    return 1 - anomalyFactor; // degrades trust as delta exceeds 15s
  }
  return 1.0;
}
```

---

## Model Data: `trust.model.json`

```json
{
  "compromised_links": {
    "Aegis-Boreas": { "mean_delta_ms": 50000, "std_delta_ms": 5000 },
    ...
  }
}
```

Links not in `compromised_links` are assumed historically honest.

---

## `getTvForLink(planetA, planetB) → number`

Reads `universe-config.json` to compute the physics void travel time `Tv` for a planet pair. Results are cached in `tvCache: Map<string, number>`.

**Fallback chain:**

1. `UNIVERSE_CONFIG_PATH` env var
2. `challenge p2/universe-config.json`
3. `challenge p1/universe-config.json`
4. `universe-config.json`
5. Hardcoded fallback table for the 12 known link pairs

> [!WARNING]
> The hardcoded fallback table (for emergency use) mirrors the specific Zeta-26 planet positions. It will give wrong results on a different universe config. Always prefer the config-based path.

---

## The "Unseen Vector" Detection

Honest links get trust score 1.0 unless `liveDelta > 15,000ms`. This threshold catches novel spoofing attacks on links that aren't in the compromised_links list yet. The trust degrades linearly from 1.0 to 0.0 as liveDelta goes from 15,000 to 45,000ms.

---

Back to [[00 - Index]]
