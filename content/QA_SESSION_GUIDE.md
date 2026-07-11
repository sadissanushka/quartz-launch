# Stack Kings — QA Session Guide

**Purpose:** Prepare the team for judging-panel Q&A.  
**Companion:** [`PROJECT_OVERVIEW.md`](PROJECT_OVERVIEW.md) · [`challenge/DECISION_AUDIT.md`](challenge/DECISION_AUDIT.md) · [`challenge/INTELLIGENCE_REPORT.md`](challenge/INTELLIGENCE_REPORT.md)

**How to use this file**

1. Memorise the **30-second opener** and **key numbers**.
2. Drill **Tier 1 (Very High)** until answers are automatic.
3. Assign **role owners** so nobody talks over each other.
4. Rehearse **traps** — these are designed to make you uncomfortable.
5. Own the **limitations** section — honesty scores better than bluffing.

---

## 0. Opening statement (memorise)

> "We kept Phase 1 physics routing intact and added an **Analytical Co-Pilot** in front of it. Three CSV-trained models — congestion, trust, and targeting — score each interplanetary link from live Chimera `/state`. We fold those into a single **True Cost** in milliseconds, run Dijkstra, and a **sequential agent** re-evaluates each hop and reroutes if a link saturates or spoofs. Everything emits the Council's mandatory JSON schema, validated with Zod. Generative AI is only an **optional** natural-language fallback — core routing is classical models plus physics. The dashboard proves it end-to-end, including transmitting on the Co-Pilot path via `hop_log`."

---

## 1. Key numbers (flashcard)

| Topic | Value |
|-------|-------|
| Interplanetary links | **12** |
| Congestion MAE | **~22.7 s** (5,743 ticks) |
| Trust P / R / F1 | **92% / 70% / 0.80** |
| Spoofed links | **Aegis-Elysium**, **Boreas-Fenix** |
| Trust hard-block floor | **0.5** |
| Saturation | `load_ratio ≥ **0.90**` |
| True Cost scales | trust **80k**, targeting **50k**, entropy **25k** ms |
| Diversification ε | **5%** |
| Anomaly scores | trust **≤ 0.3**, targeting **≥ 0.85** |
| Tests | **216** unit + **7** E2E |
| Rate limit | **120** req/min |
| Parser LLM threshold | confidence **&lt; 0.6** |

---

## 2. Role split (who answers what)

| Topic | Primary | Backup |
|-------|---------|--------|
| Model training, MAE, spoofed links | **Ruwan** | Inusha |
| True Cost formula, decision audit | **Inusha** | Ruwan |
| Architecture, API, Chimera client, agent | **Inusha** | Ruwan |
| Dashboard, demo, map overlays, E2E | **Anushka** | Inusha |
| GenAI / NL parser | **Inusha** | Anushka |
| Limitations & future work | Anyone — be honest | |

**Rule:** If unsure, hand off: *"Ruwan fitted that — Ruwan?"* Never invent numbers.

---

## 3. Questions by probability

### Tier 1 — Very high probability (almost certain)

These map directly to evaluation trials and the brief. **Must** be fluent.

---

#### Q1. Why didn’t you replace the Phase 1 router?

**A:** The brief says physics foundations stay intact. Chimera attacks **links**, not planet physics or codex. We wrap `findShortestRoute` and only change edge weights via `voidHopSurchargeMs`. Physics `Tv` is always the base; models add penalties.

---

#### Q2. Explain your three models in one minute.

**A:**

1. **Congestion** — power-law `k · load_ratio^p` → extra delay; saturate at 0.90 → hard block.  
2. **Trust** — compare self-reported latency to physics + congestion; under-reporting ⇒ spoof; floor 0.5 blocks. Spoofed: Aegis-Elysium, Boreas-Fenix.  
3. **Targeting** — logistic on `traffic_share` → P(jam); diversify so we don’t paint a target.

---

#### Q3. How do you combine congestion, trust, and targeting?

**A:** Convert everything to milliseconds:

```
combined_cost = physics_void_ms
              + congestion_penalty_ms
              + (1 - trust) × 80,000
              + targeting_risk × 50,000
              - traffic_share × 25,000
```

One Dijkstra minimises the path sum. Within 5% of optimal, pick lowest aggregate targeting risk (entropy).

---

#### Q4. Walk me through one `link_evaluations` row. *(Decision Audit)*

**A:** Point at a live row:

1. **Congestion penalty** — extra ms from current load (power-law).  
2. **Trust** — do we believe self-report? Low = spoofing.  
3. **Targeting** — jam probability from traffic share.  
4. **Combined cost** — the number Dijkstra minimised for this hop.

Formulas: `challenge/DECISION_AUDIT.md`.

---

#### Q5. What are your three intelligence findings?

**A:**

1. Congestion is **non-linear** — reroute before the 0.90 cliff.  
2. **Two links spoof** — Aegis-Elysium (~78 s mean delta), Boreas-Fenix (~65 s).  
3. **Predictable routes get jammed** — entropy bonus + 5% tie-break diversify.

---

#### Q6. How does the sequential Co-Pilot agent work?

**A:** Parse NL → baseline physics path → **for each hop**: fetch live state → three model tools → combined cost → if blocked, **reroute from current node** with updated blocklist → emit final report. Matches the Council’s “localized loop,” not a one-shot solve.

---

#### Q7. Where do you use AI / GenAI?

**A:** Two senses:

- **Classical ML:** the three CSV-trained models (core of Phase 2).  
- **Generative AI:** **only** optional NL parser fallback (Ollama/Gemini) when rules confidence &lt; 0.6. Default is rules-only. Core routing does **not** call an LLM.

On the dashboard, GenAI lives only in **Co-Pilot Natural Language** — and for clean utterances like “Send Hello world from Aegis to Caelum,” rules handle it with **no LLM call**.

---

#### Q8. How do you prove the Co-Pilot path is real, not just UI?

**A:** `POST /api/transmit` with `use_copilot: true` constrains the physics router to `chosen_path`. The `hop_log` is the receipt. Flag: `transmitted_on_copilot_path`.

---

#### Q9. What happens when a link saturates mid-session?

**A:** Live Chaos Monitor re-polls `/state`, sequential evaluation blocks the hop, picks a detour, dashboard shows a **pivot banner**. Next send uses the new path — zero packet loss on that send.

---

#### Q10. Did you train on live `/state`?

**A:** **No.** Brief warned early `/state` was scrambled. We train only on CSVs (`npm run train:models`). Live `/state` is **inference input** only.

---

### Tier 2 — High probability (very likely)

---

#### Q11. Why is congestion MAE ~23 seconds? Isn’t that bad?

**A:** Magnitude can be imprecise per link. Routing needs **direction** and the **saturation guard** more than perfect ms. We hard-block at 0.90 / saturated / null latency so we don’t treat failure as “fast.” Future: logistic slow→fail boundary + online calibration.

---

#### Q12. Trust recall is only 70% — why?

**A:** We bias toward **precision (92%)** — false spoof flags block good links. Floor + reroute limit damage when we miss a spoof tick. Known chronic spoofers are still caught via history.

---

#### Q13. Targeting P(jam) is only ~8.7% vs 8.2% — is the signal useless?

**A:** Absolute calibration is subtle. We use **relative** share ranking for diversification. We don’t need perfect probability — we need to avoid always picking the same predictable path.

---

#### Q14. Why those scale constants (80k / 50k / 25k)?

**A:** Policy knobs, not physical constants. They encode risk appetite: “how many ms of delay we’d accept to avoid distrust or jam risk.” Tunable in `constants.ts` without retraining models.

---

#### Q15. What’s the difference between True Cost router and sequential agent?

**A:** **True Cost router** = global Dijkstra with dynamic weights + diversification. **Sequential agent** = hop-by-hop tool loop with mid-path reroute (chaos pivot narrative). Both share `evaluateLink`.

---

#### Q16. How is the Chimera API key secured?

**A:** `CHIMERA_TEAM_KEY` only in server `.env` / Vercel env. Never in client bundle or Git. Browser talks to **our** `/api/route`, not Chimera directly.

---

#### Q17. What if Chimera is down during the demo?

**A:** Graceful degradation: **neutral link state**, explanation notes it, physics routing still works. No crash. Health shows `chimera_reachable: false`.

---

#### Q18. How do you avoid schema disqualification?

**A:** Zod validates every `Phase2RoutingReport` field before HTTP response. Integration tests assert the schema. Anomaly sanitisation prevents NaN scores.

---

#### Q19. `/api/route` vs `/api/transmit`?

**A:** `/api/route` → Council report from NL/structured intent. `/api/transmit` → Phase 1 packet + hop_log; optional `use_copilot` attaches report and sends on Co-Pilot path.

---

#### Q20. Who built what?

**A:** Ruwan = models + CSVs + intelligence report. Inusha = client, agent, True Cost, parser, APIs, schema. Anushka = dashboard, overlays, E2E, demo UX. Models are pure functions; UI consumes JSON only.

---

### Tier 3 — Medium probability (expect a few)

---

#### Q21. Why power-law for congestion, not a neural net?

**A:** Interpretable, fast, reproducible in TypeScript, no Python at runtime. Judges can audit coefficients. Fits the Decision Audit trial.

---

#### Q22. Why logistic regression for targeting?

**A:** Binary jam outcome + continuous `traffic_share` is a classic logistic setup. L2 regularisation reduces overfit. Coefficients ship as JSON.

---

#### Q23. Explain unseen-vector / anomaly handling.

**A:** Sanitize OOD fields → models never emit NaN. Conservative scores (trust ≤ 0.3, targeting ≥ 0.85). Explanation flags anomaly. Inflated cost steers away; last-resort still deliverable.

---

#### Q24. How does the hybrid NL parser work?

**A:** Rules first: `from X to Y`, arrows, fuzzy Levenshtein planet match, quoted/`send …` payload. If confidence &lt; 0.6 and LLM configured → Ollama/Gemini. Else throw a clear error. Dashboard shows **preview before route**.

---

#### Q25. What does `/api/health` prove?

**A:** Engine loaded, config path/hash, `models_loaded`, `chimera_reachable`, `last_tick` (refreshed from `/state` when reachable). System Init trial.

---

#### Q26. How many tests? What’s covered?

**A:** ~216 unit (engine, models, agent, schema, API helpers) + 7 Playwright E2E (dashboard, chaos, Co-Pilot route, live monitor pivot). Chimera mocked in CI.

---

#### Q27. How does route diversification work?

**A:** Among paths within **5%** of best True Cost, pick lowest sum of targeting risk. Stops always painting the same target.

---

#### Q28. What is `canonicalLinkId`?

**A:** Alphabetical `planet_a-planet_b` (e.g. Aegis-Boreas). Matches config, CSVs, and undirected physics edges.

---

#### Q29. Can you show baseline vs Co-Pilot path?

**A:** Yes — map toggle. Explanation also embeds baseline physics path. When Chimera is offline/neutral, paths may look similar; with live spoof/congestion they diverge.

---

#### Q30. Rate limiting?

**A:** 120 req/min per client on API routes. Structured 429. Protects demo from accidental spam / Live Monitor abuse.

---

### Tier 4 — Lower probability (still prepare)

---

#### Q31. Latency equations — void vs internal?

**A:** Void `Tv` from distance/atmosphere/c; internal `Tp` from fiber arc + tower delay. Constants from `universe_metadata`. Phase 2 adds Chimera penalty **on top of** void, not instead of it.

---

#### Q32. Why Dijkstra over `(planet, entryTower)`?

**A:** Internal cost depends on which tower you enter and leave. Node-only shortest path would be wrong.

---

#### Q33. Codex / hop_log — why next-hop encoding?

**A:** Protocol: encode into **next** planet’s dialect before the void. hop_log shows local dialect + next-hop digits + binary stream — M2 proof.

---

#### Q34. Docker / Vercel / CI?

**A:** Standalone Next output for Docker; Vercel Git deploy; Actions run lint/typecheck/test/build/E2E. Sentry source maps optional via `SENTRY_AUTH_TOKEN`.

---

#### Q35. What would you do with one more week?

**A:** Online calibration on unscrambled live data; logistic saturation boundary; temporal trust across ticks; adaptive True Cost scales; stronger targeting features (rolling share).

---

#### Q36. Is the LLM required for the demo?

**A:** No. Scripted utterances are rules-only. LLM is optional for ambiguous Q&A phrasing.

---

#### Q37. How do you handle undeliverable routes?

**A:** Phase 1 returns `undeliverable` + reason. Co-Pilot constrained transmit falls back to physics baseline if Co-Pilot path is severed.

---

#### Q38. Security headers / observability?

**A:** CSP, HSTS, X-Frame-Options in `next.config`. Structured JSON API logs. Optional Sentry. Rate limits.

---

## 4. Common traps (designed to make you uncomfortable)

Judges often use these. **Recognise the trap → answer the real question → don’t panic.**

---

### Trap A — “So you’re just using ChatGPT for routing?”

**Intent:** Collapse your system into “LLM hype.”  
**Response:**  
> “No. Routing is physics Dijkstra plus three classical models. The LLM is an **optional** parser fallback for messy English. Default and CI are rules-only. You can turn GenAI off and the Co-Pilot still routes.”

**Follow-up they may use:** “Then why mention AI at all?”  
> “Phase 2 requires analytical models — that’s ML. GenAI is a thin UX layer for natural language, not the decision engine.”

---

### Trap B — “Your MAE is 23 seconds — your model is useless.”

**Intent:** Force you to defend a weak metric.  
**Response:**  
> “We report MAE honestly. For routing, **rank and saturation** matter more than exact ms. The hard block at 0.90 prevents treating a dead link as free. We’re not claiming oracle latency prediction.”

**Don’t:** Invent a lower MAE or claim “it’s fine.”

---

### Trap C — “Show me the Co-Pilot choosing a different path than physics — right now.”

**Intent:** Live demo may show identical paths if Chimera is quiet/offline.  
**Response:**  
> “When telemetry is neutral or Chimera is unreachable, True Cost ≈ physics — that’s correct behaviour. With live congestion or spoofed links, costs diverge. We can also force a pivot with Live Chaos Monitor / Hyper-Flare.”

**Prep:** Know how to trigger a scenario and turn on Live Monitor.

---

### Trap D — “You trained on live data / you cheated the scrambled period.”

**Intent:** Integrity check.  
**Response:**  
> “We did not. Training is CSV-only via `npm run train:models`. Live `/state` is inference. Coefficients are in repo JSON — reproducible offline.”

---

### Trap E — “Pick a random link and explain every score without looking at docs.”

**Intent:** Decision Audit stress.  
**Response pattern:**  
1. Read the four numbers from the **dashboard row** (allowed — that’s the report).  
2. Explain each in plain English (congestion / trust / targeting / combined).  
3. If blocked: say whether saturation or trust floor.  
**Don’t:** Open source code. Use `DECISION_AUDIT` talking points from memory.

---

### Trap F — “Isn’t sequential evaluation just theatre? One Dijkstra is enough.”

**Intent:** Attack the agent narrative.  
**Response:**  
> “Global Dijkstra optimises one tick. The sequential loop handles **mid-path** blocks and matches the required per-hop tool-call flow. Live Chaos Monitor depends on re-evaluation, not a frozen path.”

---

### Trap G — “Your targeting model barely separates jam vs clean.”

**Intent:** Push on weak calibration.  
**Response:**  
> “Agreed — absolute probabilities are close. We use the signal for **relative** diversification, not as a precise jam forecast. Entropy bonus + 5% tie-break encode that policy. It’s a known limitation.”

---

### Trap H — “What if we feed NaN / garbage telemetry?”

**Intent:** Unseen-vector / robustness.  
**Response:**  
> “`detectLinkAnomaly` sanitises inputs, applies conservative trust/targeting, flags the explanation. Schema stays valid. We steer away when alternatives exist.”

---

### Trap I — “Who wrote this code — did AI write everything?”

**Intent:** Ownership / authenticity.  
**Response:**  
> “We used AI assistants as tools, like any modern team. Architecture, model choices, formulas, and demo narrative are ours. We can explain every score and limitation without reading code — that’s the bar we prepared for.”

---

### Trap J — “Your dashboard GenAI section is fake — it’s just regex.”

**Intent:** Catch overclaiming GenAI.  
**Response:**  
> “For this utterance, yes — rules parse it. That’s intentional. GenAI is the **fallback** for ambiguous input when enabled. We don’t claim every NL request hits an LLM.”

**This is a good trap to walk into calmly** — it shows honesty.

---

### Trap K — “Why should we trust your trust model?”

**Intent:** Wordplay + methodology.  
**Response:**  
> “Trust is measured against physics + congestion expectation, evaluated on held-out telemetry ticks: 92% precision on spoof flags. Chronic spoofers match the intelligence report. It’s not circular — physics is the ground reference.”

---

### Trap L — “Silence / stare after a hard question.”

**Intent:** Pressure.  
**Response habit:**  
> Pause 2 seconds → restate the question → answer in one claim + one proof.  
> If you don’t know: “I don’t want to invent that number — Ruwan / our report says X.”

---

### Trap M — “Compare yourselves to another team / why are you better?”

**Intent:** Ego bait.  
**Response:**  
> “We can’t speak for others. Our bar was: physics intact, three auditable models, schema-valid reports, live pivot, and hop_log proof. Happy to defend those choices.”

---

### Trap N — “Production is broken / health shows chimera_reachable false.”

**Intent:** Ops stress.  
**Response:**  
> “That’s degraded mode by design. Co-Pilot uses neutral state; Phase 1 still delivers. We’ll note it in the explanation. Root cause is usually key/env or network to chimera.launch26.space.”

---

## 5. Limitations (say these before they force you)

Owning limitations builds credibility. Use this language.

### 5.1 Model limitations

| Limitation | Impact | Mitigation today | Future |
|------------|--------|------------------|--------|
| Congestion MAE ~23 s | Imprecise penalty magnitude | Saturation hard-block; directional signal | Logistic boundary; online fit |
| Trust recall 70% | Some spoof ticks missed | High precision; floor 0.5; known-spoof priors | Temporal consistency across ticks |
| Targeting jam vs clean close | Weak absolute calibration | Relative share + entropy + ε tie-break | Rolling share / path popularity features |
| Static coefficients | Live-day drift possible | CSV priors; anomaly conservatism | Online calibration post-unscramble |

### 5.2 System limitations

| Limitation | Impact | Mitigation |
|------------|--------|------------|
| Chimera dependency | Offline ⇒ neutral scores | Graceful fallback; health flags |
| Rules-first NL | Ambiguous English may fail | Optional LLM; clear error message |
| LLM latency/cost (if enabled) | Demo lag | Off by default; rules cover script |
| Single-region deploy | No multi-region poll redundancy | Vercel + documented env |
| True Cost scales are hand-tuned | Policy, not learned | Documented constants; tunable |
| Live Monitor polling | Extra API load | Rate limit; 4 s interval; in-flight guard |

### 5.3 Scope limitations (honest boundaries)

- We do **not** replace planet-level physics or invent new void equations.  
- We do **not** use deep learning for routing.  
- We do **not** claim GenAI drives path choice.  
- We do **not** online-learn during the scrambled `/state` window.  
- We do **not** guarantee Co-Pilot ≠ baseline when Chimera is quiet — that’s correct.

### 5.4 One-sentence limitation closer

> “Our biggest honest gaps are congestion magnitude error, targeting absolute calibration, and static coefficients — we mitigate with hard blocks, diversification policy, and anomaly conservatism, and we’d next add online calibration.”

---

## 6. Demo failure cheat sheet

| Symptom | What to say / do |
|---------|------------------|
| Chimera unreachable | “Degraded mode — neutral state.” Continue NL route. |
| Path equals baseline | “Expected if telemetry neutral.” Trigger Hyper-Flare + Live Monitor. |
| NL parse error | Retry: `Send "Hello world" from Aegis to Caelum`. |
| Rate limited 429 | Pause; explain 120/min; turn off Live Monitor briefly. |
| Transmit undeliverable | Clear dead nodes/links; Baseline scenario; retry. |
| Judge asks obscure link | Expand that row in Link Evaluations — explain live numbers. |

---

## 7. 15-minute rehearsal plan

| Time | Drill |
|------|-------|
| 0–2 min | Opener + three findings (no notes) |
| 2–5 min | Decision Audit on one live row |
| 5–8 min | True Cost formula on whiteboard/paper |
| 8–11 min | Trap A + B + C + J (GenAI / MAE / live path / regex) |
| 11–13 min | Limitations closer (Section 5.4) |
| 13–15 min | Chaos Monitor pivot once |

---

## 8. Quick “do / don’t”

| Do | Don’t |
|----|-------|
| Hand off to the owner | Talk over Ruwan on model metrics |
| Admit limitations early | Invent MAE / precision numbers |
| Point at live report rows | Open source code in Decision Audit |
| Separate GenAI vs classical ML | Claim “AI routes everything” |
| Restate hard questions | Fill silence with speculation |

---

## 9. Cross-reference index

| Need | Open |
|------|------|
| Full system picture | [`PROJECT_OVERVIEW.md`](PROJECT_OVERVIEW.md) |
| Score formulas | [`challenge/DECISION_AUDIT.md`](challenge/DECISION_AUDIT.md) |
| Model metrics | [`challenge/INTELLIGENCE_REPORT.md`](challenge/INTELLIGENCE_REPORT.md) |
| Demo flow | [`DEMO_SCRIPT.md`](DEMO_SCRIPT.md) |
| Formal report | [`challenge/PHASE2_PROJECT_REPORT.md`](challenge/PHASE2_PROJECT_REPORT.md) |

---

*Stack Kings — LAUNCH 26 · Prepare hard questions harder than the easy ones.*
