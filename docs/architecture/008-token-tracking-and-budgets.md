# ADR-008: Token tracking and per-bucket budgets

- Status: Accepted
- Date: 2026-05-15
- Deciders: Anshul Jain (project owner)
- Phase introduced: Phase 3 (tracking); Phase 6 (warnings); Phase 7 (cost script)

## Context

LLM costs scale with use. Without enforcement, a single instructor (or
runaway agent) could burn through a month's budget in hours. The
platform also needs visibility per feature so we know which surfaces
drive cost.

Constraints from `CLAUDE.md` §6:

- Every LLM call must write a `LlmUsage` row.
- Middleware in `packages/llm` enforces caps **before** the call and
  reconciles after.
- Buckets: `ASSESSMENT`, `PLANNING`, `CHAT`, `QUESTION_GEN`.
- Hard 429 with budget info on cap exceeded.
- Dashboards: instructor-self in Phase 3; admin global in Phase 3.

Per `CLAUDE.md` §2, `InstructorProfile.monthly_token_budget` is set by
admin.

## Decision

Track token usage at call time and enforce per-bucket monthly caps via
middleware in `packages/llm/aiClient`.

### Recording

- `LlmUsage` row written on every call, including failed ones (with
  zero tokens out and the failure reason in metadata).
- Columns: `user_id`, `role`, `feature` (bucket),
  `model`, `input_tokens`, `output_tokens`, `cost_usd_cents`,
  `portkey_trace_id`, `request_id`, `created_at`.
- `cost_usd_cents` is computed at call time using a `model → price`
  table maintained in `packages/llm/pricing.ts`. Refreshed manually via
  `scripts/refresh-llm-pricing.ts` (introduced in Phase 7).

### Enforcement (reservation pattern)

For each call, the middleware:

1. Reads the current user from request context.
2. Looks up the user's per-bucket and overall monthly caps.
3. Sums actual + reserved usage for the current month.
4. Estimates this call's tokens (input from prompt length, output from
   `max_tokens`).
5. If `current + estimate > cap`, **reject** with HTTP 429 plus a body:
   `{ error: { code: "BUDGET_EXCEEDED", bucket, used, cap } }`.
6. Otherwise reserve the estimate.
7. After the call, write the actual `LlmUsage` row and release the
   reserve.

Reservations live in an in-memory map keyed by user with a TTL safety
net; on process restart they are forgotten (acceptable — the next call
re-checks against the DB).

### Buckets

- `ASSESSMENT` — AssessorAgent calls.
- `PLANNING` — PlannerAgent calls.
- `CHAT` — free-form helper chat.
- `QUESTION_GEN` — bulk question pool generation.

Each bucket has a per-instructor monthly cap, plus an overall cap
across buckets. ADMIN sets these on `InstructorProfile`. Students have
implicit caps tied to the instructor whose batch they are taking the
assessment in (assessment cost rolls up to the owning instructor).

### Soft warnings (Phase 6)

When a user crosses 80% of any cap, a `Notification` row is emitted
(kind `BUDGET_WARNING`, payload includes bucket and percentage). This
is a **notification**, not a rejection.

### Dashboards

- **Instructor**: own usage by feature, by day, vs budget. Phase 3.
- **Admin**: all instructors + global totals + cost USD. Phase 3.

Both dashboards are read paths over `LlmUsage` aggregations, surfaced
through the API.

## Consequences

### Positive

- Hard cost ceiling per user — a runaway loop cannot exceed the cap.
- Per-bucket granularity surfaces which feature drives spend.
- Trace IDs on every row connect to Portkey for deep debugging.
- Soft 80% warnings give users (and admin) lead time before hard cuts.
- Cost-USD column makes billing decisions tractable in Phase 8.

### Negative / Trade-offs

- Reservation pattern is approximate — actual output tokens often
  differ from the estimate. We over-reserve and reconcile, which means
  the ceiling is slightly conservative but never breached.
- In-memory reservation state is per-process; multi-replica deploys
  (Phase 7+) need either sticky routing or a shared store (Redis)
  promoted to a future ADR.
- Pricing table drift: prices change; we mitigate by refreshing via
  script and snapshotting the table version in `LlmUsage` metadata.

### Neutral

- Failed calls still write rows for visibility, even with zero output
  tokens.
- Refunds for "model returned an error" cases are credited by writing
  the actual (small) usage; we do not zero out failed calls because
  providers often still bill for input tokens.

## Alternatives considered

- **Per-user daily-only caps**: simpler, less granular per feature, and
  less aligned with the way instructors plan budgets monthly.
- **No enforced limits, just monitoring**: existential cost risk for a
  single-developer project.
- **Per-feature global caps (no per-user)**: any one user could starve
  the rest.
- **Provider-side limits via Portkey only**: Portkey has rate limits
  but not user-aware monthly buckets in the shape we need; combining is
  complementary, not a replacement.

## Related ADRs / docs

- `CLAUDE.md` §6 ("Token tracking & budgets").
- ADR-006 (LLM gateway) — middleware lives in `packages/llm`.
- ADR-007 (agent design) — agents respect the same ceilings.
- `docs/development/phase_3.md` — initial dashboards.
- `docs/development/phase_6.md` — soft warning notifications.
- `docs/development/phase_7.md` — pricing refresh script.
