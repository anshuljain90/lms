# ADR-013: Testing strategy — Vitest + testcontainers + Playwright

- Status: Accepted
- Date: 2026-05-15
- Deciders: Anshul Jain (project owner)
- Phase introduced: Phase 0 (scaffolded), Phase 1 (first tests land)

## Context

The product mixes deterministic logic (auth, enrollment state machine,
schedule math, RBAC) with non-deterministic LLM behaviour (planner
output, assessor scoring). We need a testing approach that:

- Exercises real Postgres + real Prisma rather than mocks for anything
  involving DB constraints, transactions, and SQL.
- Stays fast enough that a solo developer runs the whole suite on
  every push.
- Treats LLM calls as a side-effect that can be deterministically
  faked, with a separate opt-in path to validate against a live model.
- Doesn't gate every PR on a coverage number while still surfacing
  regressions in coverage.

## Decision

- **Vitest** for unit and integration tests, in both `apps/api` and
  `apps/web`.
- **testcontainers** spins up a real PostgreSQL 16 container per
  integration test suite, with Prisma migrations applied in a
  `globalSetup`.
- **Playwright** for end-to-end tests of named "golden flows" only.
  E2E tests run against `docker compose up --wait` (full stack).
- **Coverage**: ~70% line coverage target across `apps/api` and
  `apps/web` combined, reported via Vitest's c8 integration.
  - Reported on every PR in Phase 7 as a comment.
  - **Not** a hard CI gate — we'd rather have honest tests than
    coverage padding.
  - Excludes generated code: Prisma client output, OpenAPI types,
    config files, dist.

Pyramid shape (intentional):

```
              ┌──────────┐
              │   e2e    │  golden paths only (~10–20 specs)
              └──────────┘
            ┌──────────────┐
            │ integration  │  bulk of confidence (real DB)
            └──────────────┘
          ┌──────────────────┐
          │      unit        │  pure functions, schemas, helpers
          └──────────────────┘
```

LLM testing rules:

- Default in unit/integration: the LLM gateway is mocked with
  deterministic fixtures (`packages/llm/test-fixtures/`). Tools assert
  on call shape, tool args, and emitted DB rows.
- An optional **live smoke test** suite is gated by `LLM_LIVE_TEST=1`.
  Runs a tiny number of representative prompts against the real
  Portkey gateway. Not in the default test command.
- `AssessorAgent` invariants (no answer leakage, no payload mutation,
  cap on follow-ups) are tested as pure code-level guards, **not**
  by relying on prompt obedience.

Per-phase exit criteria (`/docs/development/phase_<N>.md`) list the
named e2e flows that must be green before the phase ships.

## Consequences

### Positive

- Vitest runs everything in-process with esbuild — orders of magnitude
  faster than Jest with ts-jest for our footprint.
- Real Postgres in integration tests catches issues that mock-based
  tests routinely miss: constraint violations, transaction rollbacks,
  cascade behaviour.
- Keeping e2e small means CI stays fast and flakes stay rare; bugs
  found at e2e layer become candidates for new integration tests.
- LLM mocking keeps the test suite hermetic and free; the live opt-in
  preserves the option to catch real-world drift.

### Negative / Trade-offs

- testcontainers adds a Docker dependency to integration runs. Already
  required for local dev, so the cost is zero on developer machines
  and one extra step on CI runners.
- Vitest's Jest-API-compatible mode occasionally diverges in subtle
  ways from Jest examples found online; we standardize on Vitest
  patterns in our own docs.
- A coverage target without a CI gate trusts the developer. Acceptable
  at this team size.

### Neutral

- Playwright traces are uploaded as CI artifacts in Phase 7 for failed
  e2e runs.
- A shared `apps/api/test/integration/setup.ts` provides a fresh DB
  schema per worker via `prisma migrate deploy` against an isolated
  database.
- Snapshot tests are allowed but discouraged for anything other than
  stable rendered HTML; LLM outputs are never snapshotted.

## Alternatives considered

- **Jest**: slower with TS at this point, and Vitest's ESM and
  watch-mode story is better for our stack. No remaining advantage.
- **Mocha + Chai**: more glue, weaker watch story, no built-in
  coverage.
- **Strict TDD across the board**: appropriate for a team and stable
  domain; counterproductive for a solo build phase where shape is
  still moving. We do require tests with every PR, just not test-first.
- **Cypress for e2e**: fine, but Playwright's parallelism, mobile
  emulation, and trace viewer are stronger for our PWA use case.
- **No integration tests, only unit + e2e**: the missing middle layer
  is exactly where most of our regressions would live.

## Related ADRs / docs

- `CLAUDE.md` §3 (Tech stack — Testing row), §9 (Conventions — Testing).
- ADR-003 (NestJS) — testability via DI.
- ADR-004 (Postgres + Prisma) — what we're testing against.
- ADR-005 (LLM gateway) — what we're mocking.
- ADR-019 (CI/CD) — when these tests start running on every PR.
- `docs/development/phase_*.md` — per-phase named e2e flows.
