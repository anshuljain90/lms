# ADR-004: Database — PostgreSQL 16 with Prisma ORM

- Status: Accepted
- Date: 2026-05-15
- Deciders: Anshul Jain (project owner)
- Phase introduced: Phase 0

## Context

The data model (see `CLAUDE.md` §5) has strong relational structure:

- Users → InstructorProfile / Enrollments / TaskAssignments / etc.
- Course → Module → Topic.
- Batch → Session, EnrollmentRequest, Enrollment.
- Task → QuestionPool → Question; Task → AssessmentAttempt → AttemptTurn.
- Cross-cutting tables: `LlmUsage`, `AuditLog`, `Notification`.

It also has flexible payloads:

- `learning_objectives_json` on Course/Task.
- `payload_json` and `rubric_json` on Question.
- `tool_calls_json` on AttemptTurn.
- `metadata_json` on AuditLog; `payload_json` on Notification.

Future phases need:

- Vector search for RAG (`CLAUDE.md` §8 lists this in Phase 8).
- Typed queries from a single TypeScript codebase.
- Migrations checked into git, applied deterministically.
- Single-DB operations posture (one container in dev, one managed
  Postgres in prod).

## Decision

Use **PostgreSQL 16** as the only operational database, accessed via
**Prisma ORM** with **Prisma Migrate** for schema evolution.

- Schema lives in `prisma/schema.prisma`; migrations under
  `prisma/migrations/`.
- Flexible payloads use `Json` (mapped to PG `jsonb`).
- All `*_at` columns are `timestamptz` and store UTC.
- pgvector extension is **not** enabled in P1 but is a known Phase 8
  add (RAG); chosen now so we don't have to migrate databases for it.
- Prisma client is instantiated once per Node process and injected via
  NestJS's DI container.
- Raw SQL is forbidden in business logic; exceptions only in migrations
  or rare perf-critical aggregates, with a `// reason:` comment.

## Consequences

### Positive

- Relational integrity (FKs, unique constraints, cascades) for the
  Users/Batches/Tasks/Attempts core; Postgres handles it natively.
- `jsonb` covers the flexible payloads without forcing a second store.
- Prisma's generated types flow into shared types via `@prisma/client`;
  combined with Zod schemas in `packages/shared`, the TypeScript boundary
  is checked end-to-end.
- Prisma Migrate produces deterministic, reviewable SQL migrations.
- pgvector is a one-extension upgrade away when Phase 8 RAG lands —
  no second datastore to operate.
- Single DB to back up, monitor, tune.

### Negative / Trade-offs

- Prisma's query layer is opinionated; complex aggregations sometimes
  need `$queryRaw` (allowed with comment) or a SQL view.
- Long-running migrations are awkward in Prisma vs hand-rolled tools;
  for any large data backfill we wrap with explicit scripts under
  `scripts/` rather than in-line in a migration.
- Prisma's connection pooling is per-process; under heavy load we may
  need to add PgBouncer in Phase 7. Not a P1 concern.

### Neutral

- We use `cuid()` (Prisma default) for primary keys; predictable string
  IDs that travel cleanly through APIs. UUIDs only where another system
  demands it.
- Prisma's default soft-delete pattern is not used; `removed_at`-style
  columns are explicit on Enrollment.

## Alternatives considered

- **Drizzle ORM**: lighter, SQL-first, growing fast. Migration ergonomics
  and ecosystem are still less mature than Prisma's; we revisit later if
  Prisma becomes a constraint.
- **Raw SQL + a query builder (Kysely)**: maximum control, more toil; we
  would still rebuild the type-flow Prisma gives us for free.
- **TypeORM / MikroORM**: comparable feature set, smaller community
  momentum vs Prisma.
- **MongoDB (document)**: would suit `payload_json` but loses the
  relational guarantees we rely on for Users/Batches/Attempts. A split
  Postgres+Mongo posture doubles ops complexity.
- **SQLite for dev, Postgres for prod**: divergence in JSON semantics
  and pgvector availability; testcontainers makes real Postgres in dev
  cheap enough that the divergence isn't worth it.

## Related ADRs / docs

- `CLAUDE.md` §5, §9.
- ADR-003 (backend) — Prisma client lives in `apps/api`.
- ADR-007 (agents) — `AttemptTurn` writes happen on every agent turn.
- ADR-008 (token tracking) — `LlmUsage` rows.
- `docs/development/phase_0.md` — initial schema + seed.
