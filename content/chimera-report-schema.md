# `src/lib/chimera/report-schema.ts` — Output Schema Validator

**Tags:** `#validation` `#zod` `#phase-2` `#phase-3`  
**See also:** [[chimera-routing-report]] · [[api-route]] · [[relic-types]]

---

## Purpose

**Runtime validation** of the mandatory Council `Phase2RoutingReport` output using Zod. Every response from `/api/route` is validated before it is returned to prevent Council schema violations that would cause disqualification.

---

## Zod Schemas

### `linkEvaluationSchema`

```ts
z.object({
  link_id: z.string().min(1),
  predicted_congestion_penalty_ms: z.number().finite(),
  trust_score: z.number().min(0).max(1),
  targeting_risk_score: z.number().min(0).max(1),
  combined_cost: z.number().finite(),
}).strict();
```

### `phase2RoutingReportSchema`

```ts
z.object({
  origin_id: z.string().min(1),
  destination_id: z.string().min(1),
  chosen_path: z.array(z.string().min(1)).min(1),
  link_evaluations: z.array(linkEvaluationSchema),
  final_latency_estimate_ms: z.number().finite().nonnegative(),
  explanation: z.string().min(1),
}).strict();
```

Note `.strict()` — extra fields in the output are **rejected**. This prevents accidental schema pollution.

---

## `assertValidRoutingReport(report: unknown) → Phase2RoutingReport`

```ts
const result = phase2RoutingReportSchema.safeParse(report);
if (!result.success) {
  throw new ReportValidationError(
    `Routing report failed Council schema validation: ${issues.join("; ")}`,
    issues,
  );
}
return result.data;
```

Throws `ReportValidationError` with formatted issue paths on any mismatch.

---

## `class ReportValidationError extends Error`

```ts
class ReportValidationError extends Error {
  constructor(message: string, readonly issues: string[]) { ... }
  name = "ReportValidationError";
}
```

The `issues` array contains human-readable paths like `"link_evaluations.0.trust_score: Number must be between 0 and 1"`.

---

## Design Rationale

This is a **defensive guard on our own output** — not just external input. The TypeScript type `Phase2RoutingReport` ensures compile-time correctness, but Zod catches runtime issues like:

- NaN/Infinity leaking through from model outputs
- Empty strings in IDs
- Negative latency estimates

Both layers are needed because the models produce floating-point results that could produce non-finite values despite TypeScript types.

---

Back to [[00 - Index]]
