# ADR-016: Data model — entity overview and design rationale

- Status: Accepted
- Date: 2026-05-15
- Deciders: Anshul Jain (project owner)
- Phase introduced: Phase 1 (User, InstructorProfile, AuditLog), evolves through Phase 5

## Context

The data model is the spine of the platform. It is built incrementally
across phases (CLAUDE.md §8) but the shape is decided up front so that
later phases don't require destructive migrations.

This ADR is the canonical narrative reference for the schema. It
explains the **why** behind each non-obvious choice. The authoritative
**what** lives in `prisma/schema.prisma`. CLAUDE.md §5 contains the
flat entity list; this document expands on rationale.

## Decision

Use the entity set listed in CLAUDE.md §5, with the design choices
below.

### User vs InstructorProfile (1:0..1)

- `User` holds auth-essentials: email, password hash, role, name,
  avatar, timezone, bio (short), email verification, suspension.
- `InstructorProfile` holds instructor-only fields: long bio,
  headline, monthly token budget, social links.
- Why split: keeps the User row small and avoids null-heavy columns
  on the (eventually) much larger student population. Joining is cheap
  because we only need the profile on instructor-facing screens.

### Course → Module → Topic

- A three-level structural hierarchy. `Topic.content_md` carries
  authored markdown content.
- `Course.slug` is unique platform-wide for clean public URLs (Udemy
  catalog model, ADR-015).
- `Course.status` is `DRAFT | PUBLISHED | ARCHIVED`. Only `PUBLISHED`
  shows in the catalog.

### Batch → Session

- `Batch` is the cohort instance of a `Course`. `Batch.slug` is unique
  within a course.
- Why batches and not "classes": same vocabulary professional cohort
  programs already use; sets the mental model that this is not
  self-paced.
- `Session.scheduled_at_utc` + `duration_min` are the authoritative
  time fields; `Batch.default_timezone` drives display (ADR-014).

### EnrollmentRequest vs Enrollment

- Approval is a workflow, not a flag. `EnrollmentRequest` carries
  status (`PENDING | APPROVED | REJECTED | WITHDRAWN`), instructor
  remark, and decision audit fields.
- On approval, an `Enrollment` row is created. A student leaving a
  batch sets `removed_at` and `removed_reason` instead of deleting —
  preserves the audit trail and any historical attempts.

### Material with `scope` enum

- A single `Material` table holds both course-level and batch-level
  materials, discriminated by `scope ∈ {COURSE, BATCH}` with
  matching nullable FKs (`course_id?`, `batch_id?`).
- Why one table: cheaper to query for "all materials this user can
  see" without UNIONs; supports a future cross-scope search index.
- `source` enum (`MANUAL_UPLOAD | LLM_CHAT_EXPORT | EXTERNAL_LINK`)
  records provenance, which matters for licensing and for the planner
  agent's "find existing materials" tool.

### Task with `scope` enum

- Same pattern as Material. `scope ∈ {BATCH, COURSE_TEMPLATE}`.
- `COURSE_TEMPLATE` tasks are reusable across batches; the
  `BatchesService.create` flow copies them into the new batch with
  `deadline_utc = NULL`, requiring the instructor to set deadlines
  before assignment.
- `type ∈ {REGULAR, PRE_CLASS_GATED}` — a `PRE_CLASS_GATED` task
  produces a soft "ready / not ready" flag in the batch roster (CLAUDE.md
  §5); needs `gate_session_id` and `pass_threshold_pct`.
- `pass_threshold_pct` (Int 0–100, nullable) is required for
  `PRE_CLASS_GATED` tasks. `ReadinessService` (Phase 5) compares the
  best `GRADED` `AssessmentAttempt` score against this threshold.
- `promoted_from_task_id` (FK to `Task`, nullable, **unique**) makes
  promote-to-template idempotent: a second promotion of the same source
  no-ops on the existing template row.
- `deadline_utc` is nullable to support templates and freshly-inherited
  tasks. Service-layer rules block assignment of tasks with NULL
  deadlines and block publication of `PRE_CLASS_GATED` tasks without
  a deadline.

### TaskAssignment with nullable `student_user_id`

- One row per task represents "assigned to whole batch" when
  `student_user_id` is NULL.
- Per-student override rows (e.g., extension or makeup) carry a
  non-null `student_user_id`.
- Why this and not two tables: one table queries cleanly with
  `WHERE task_id = ? AND (student_user_id IS NULL OR
  student_user_id = ?)`. No UNION, no second table to keep in sync.

### QuestionPool separate from Task

- Pools have their own lifecycle: `DRAFT → REVIEWED → PUBLISHED`.
  Instructors can regenerate pools without touching the parent Task.
- `generation_model`, `generation_prompt_hash`, and `generated_at`
  let us reproduce or compare pools later.
- See ADR-017 for the full lifecycle.

### Question payload as JSON

- `payload_json` carries `{ question, options?, correct_idx?,
  rubric_json }`. JSON keeps MCQ and TEXT polymorphism cheap; we
  validate it with a Zod schema in `packages/shared` on read/write.
- `instructor_edited` is a boolean we flip on any edit. Useful signal
  for analytics ("how often does the instructor rewrite generated
  questions?") and for the UI badge.

### AssessmentAttempt → AttemptTurn

- Each turn (system, assessor, student) is its own row keyed by
  `attempt_id` + `turn_idx` for ordered replay.
- `tool_calls_json` captures the structured tool calls and their
  results; without this, debugging an assessor outcome from logs
  alone is impractical.
- `portkey_trace_id` links the turn to a Portkey trace for cost +
  prompt + fallback inspection (ADR-012).
- `tokens_in` / `tokens_out` / `model` denormalize for fast attempt-
  level aggregations without joining `LlmUsage`.

### LlmUsage as a separate table

- Why not a column on AttemptTurn: planning, chat, and question
  generation calls also need usage tracking. A central table
  powers the role-aware budget enforcement and dashboards described
  in CLAUDE.md §6.
- `feature` enum (`ASSESSMENT | PLANNING | CHAT | QUESTION_GEN`)
  is the bucket for budgets and dashboards.

### AuditLog as a generic table

- Generic shape: `actor_user_id, action, target_type, target_id,
  metadata_json, created_at`.
- Avoids N per-feature audit tables; keeps a single pane of glass for
  the admin "history" view.
- `action` strings follow `dot.case` (`enrollment.approve`,
  `instructor.invite`).

### Discussion → DiscussionReply

- Two-level threading only (parent + replies). Three-level threading
  is rarely useful and complicates UI and queries. Mentions and
  reactions are not in scope yet.
- `scope ∈ {TASK, SESSION}` with matching FKs, same pattern as
  Material/Task.

### Notification

- Polymorphic `payload_json` + a `kind` enum. Read state is per-row
  via `read_at`. Push delivery is opt-in and added when web push
  ships (P6+).

### Indexes (called out for hot paths)

- `User(email)` unique.
- `Course(slug)` unique.
- `Batch(course_id, slug)` composite unique.
- `Enrollment(batch_id, student_user_id)` composite for fast
  membership checks.
- `LlmUsage(user_id, created_at)` composite for budget lookups
  bounded by month/day.
- `AttemptTurn(attempt_id, turn_idx)` composite unique for ordered
  replay.
- `AuditLog(target_type, target_id, created_at)` composite for
  history pages.
- `Task(promoted_from_task_id)` unique — enforces promote-to-template
  idempotency.

### Cascade rules

- Deleting a `Batch` should **not** cascade to `Enrollment`; instead,
  set `Batch.status = CANCELLED` and `Enrollment.removed_at` so
  history survives.
- Deleting a `Course` is **blocked** if it has any `Batch`; instructor
  must archive instead.
- Deleting a `Task` cascades to its `QuestionPool` and `Question`s
  (a task without a pool is meaningless), but is **blocked** if any
  `AssessmentAttempt` exists.
- Deleting a `User` is not supported through the API; suspension is
  the only path. Hard delete only via DB script (GDPR).

## Consequences

### Positive

- Polymorphism via `scope` enums keeps related concepts in one table
  and queries simple.
- Lifecycle separation (Task vs QuestionPool, EnrollmentRequest vs
  Enrollment) lets instructors iterate on AI-generated content
  without touching downstream data.
- Generic `AuditLog` and central `LlmUsage` mean new features get
  history and budget for free.
- Schema is incrementally addable: each phase adds tables/columns,
  none requires renaming or rebuilding earlier ones.

### Negative / Trade-offs

- Polymorphic FKs with nullable columns require app-level guards to
  enforce "exactly one of (`course_id`, `batch_id`) is set". We add
  Postgres `CHECK` constraints and a Prisma extension for safety.
- JSON columns (`payload_json`, `metadata_json`, `tool_calls_json`)
  bypass Prisma's static typing; we re-validate with Zod at every
  boundary.
- Soft-delete patterns (`removed_at`, `suspended_at`) require every
  query to filter unless explicitly opt-in. We codify this in
  repository-layer helpers.

### Neutral

- All `*_at` columns are `timestamptz` UTC (ADR-014).
- Enum values are `SCREAMING_SNAKE_CASE` strings in DB and
  TypeScript.
- Prisma migration files are committed; `prisma db push` is forbidden
  outside throwaway dev DBs.

## Alternatives considered

- **Strict normalization (separate tables for course/batch
  materials)**: cleaner shape but worse for cross-scope queries and
  doubles the surface area.
- **Single attempts table without per-turn rows**: loses replay and
  per-turn token attribution.
- **Per-feature audit tables**: cleaner schema per feature, but a
  unified history view becomes a UNION of N tables that grows with
  every release.
- **Hard deletes everywhere**: simpler queries, but destroys evidence
  needed for instructor / admin disputes.

## Related ADRs / docs

- `CLAUDE.md` §5 (Data model — flat entity list), §6 (LLM strategy
  — `LlmUsage` semantics).
- ADR-004 (Postgres + Prisma).
- ADR-014 (Time zones) — `timestamptz` rule.
- ADR-015 (Role model) — `User.role`, `owner_instructor_id`.
- ADR-017 (Question pool & scoring) — pool lifecycle deep-dive.
- `prisma/schema.prisma` — authoritative source.
