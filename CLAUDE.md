# CLAUDE.md

> Canonical spec and working agreement for this repository.
> Read this file before making any non-trivial change. All other docs
> (`/docs/architecture/*.md`, `/docs/development/phase_*.md`) elaborate on
> sections of this file — they never contradict it.

---

## 1. Project mission

A **personalised learning management platform** for live-cohort teaching of
Generative AI / Agentic AI to working professionals from mixed backgrounds
(development, QA, networking, finance, security).

The differentiating capability is **per-student AI assessment**: an LLM-driven
chat assessor evaluates each student against instructor-defined learning
objectives and rubrics, so the instructor can give attention to people across
different domains and experience levels — even when the same syllabus is
running for everyone.

This is **not** a self-paced MOOC. It is a cohort-based, instructor-led tool
with batches, sessions, deadlines, and assessment-gated progression.

---

## 2. Platform model

### Roles (final)

| Role         | How created                                           | Capabilities                                                                                                                 |
| ------------ | ----------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------- |
| `ADMIN`      | Seeded on first boot from env (`ADMIN_EMAIL`/`ADMIN_PASSWORD`) | Manage instructors, override anything platform-wide, view global token + cost dashboards, suspend users.                     |
| `INSTRUCTOR` | **Admin invite only** (signed link → set password → log in) | Create/edit own courses, batches, sessions, tasks, materials. Approve/reject enrollment requests for own batches. Has admin-set monthly token budget. |
| `STUDENT`    | Self-register **OR** instructor invite                | Browse public catalog, request enrollment, view assigned tasks, take AI assessments, post in discussions.                    |

There are **no organizations / tenants**. Single platform, soft ownership via
`owner_instructor_id` columns on `Course`, `Batch`, `Material`, `Task`.

When an admin changes a user's role or suspends them, all of that user's
active refresh-token rows are revoked in the same transaction. Subsequent
`/auth/refresh` calls return 401 and the user must re-login. The 15-minute
access-token lifetime is the upper bound on stale-role privilege exposure.

### Visibility model (Udemy-like)

- **Public catalog**: every published course is discoverable on the landing
  page. Course detail page lists the instructor and available batches.
- **Edit scope**: instructors can edit only the courses, batches, etc. they
  own. Admin can edit anything.
- **Enrollment**: the owning instructor approves/rejects enrollment requests
  for their batches with an optional remark. Admin can override.
- **Students**: globally identified by email. One student account can be
  enrolled in batches across many instructors.

---

## 3. Tech stack (locked)

| Layer              | Choice                                                                                            |
| ------------------ | ------------------------------------------------------------------------------------------------- |
| Monorepo           | **pnpm workspaces** + **Turborepo**                                                               |
| Frontend           | **Next.js 15** (App Router) + TypeScript + Tailwind + shadcn/ui                                   |
| Backend            | **NestJS** + TypeScript                                                                           |
| API contract       | OpenAPI generated from NestJS Swagger; FE types generated via `openapi-typescript`                |
| Database           | **PostgreSQL 16** + **Prisma** ORM                                                                |
| Migrations         | Prisma Migrate                                                                                    |
| Auth (Phase 1)     | Email + password + **JWT** (short access, rotating refresh, HTTP-only refresh cookie)             |
| Auth (Phase 8)     | Keycloak with Google OAuth, behind same `/auth` API                                               |
| Realtime           | **Server-Sent Events (SSE)** for LLM streaming + REST for state                                   |
| Storage            | Interface-driven: `LocalFsStorage` / `MinioStorage` / `GcsStorage` (env-selectable)               |
| Email              | Interface-driven: `GmailSmtpMailer` / `ProviderApiMailer` (env-selectable); **Mailpit** in dev    |
| LLM SDK            | **Vercel AI SDK** (`ai` package) — provider-agnostic                                              |
| LLM gateway        | **Portkey** (mandatory) — handles routing, primary + fallback model, retries, observability       |
| Agents             | `PlannerAgent`, `AssessorAgent` — strict system prompts, Zod-validated tool calls, allowlisted tools, prompt-injection defenses, hard token budgets |
| Local LLM          | **Ollama** in `docker-compose` (default models: `llama3.1:8b`, `nomic-embed-text`)                |
| Observability      | OpenTelemetry SDK → **Grafana + Loki + Tempo + Prometheus** + Portkey UI                          |
| Mobile             | **PWA-first** (manifest + service worker). Capacitor wrapper later.                               |
| Time zones         | Store UTC. Render in user's TZ from profile. Batch has a default TZ.                              |
| Testing            | **Vitest** (unit + integration with **testcontainers** for real Postgres) + **Playwright** (e2e on golden paths). ~70% coverage target. |
| CI/CD              | Local-only in P1 (`pnpm test`, `docker compose up`). **GitHub Actions** in Phase 7.               |
| Deploy             | `docker-compose` for now. Production target deferred.                                             |

**Provider-agnostic discipline**: every external integration lives behind an
interface in the BE. We never `import { OpenAI } from "openai"` in business
logic — only `aiClient.streamText({...})` from the abstraction.

---

## 4. Repository layout

```
lms/
├── apps/
│   ├── web/                     # Next.js 15 (FE)
│   └── api/                     # NestJS (BE)
├── packages/
│   ├── shared/                  # Zod schemas, DTOs, enums shared FE/BE
│   ├── llm/                     # Vercel AI SDK + Portkey wrapper, agents, tools, guardrails
│   ├── storage/                 # IStorage + 3 impls (local, minio, gcs)
│   ├── mailer/                  # IMailer + 2 impls (gmail-smtp, provider-api)
│   ├── observability/           # OTel setup, structured logger
│   ├── config/                  # Zod-validated env loader
│   └── eslint-config/           # Shared lint config
├── prisma/                      # schema.prisma, migrations, seed
├── infra/
│   ├── docker-compose.yml       # full stack (postgres, mailpit, minio, ollama, lgtm, api, web)
│   ├── docker-compose.dev.yml   # dev override
│   ├── grafana/                 # provisioned dashboards & datasources
│   ├── otel-collector.yaml
│   └── caddy/                   # added in Phase 7
├── docs/
│   ├── architecture/            # ADRs (numbered, MADR format)
│   └── development/             # phase_0.md … phase_8.md
├── scripts/                     # one-off ops scripts (seed, backup, etc.)
├── .env.example
├── pnpm-workspace.yaml
├── turbo.json
├── package.json
└── CLAUDE.md                    # this file
```

---

## 5. Data model (high-level)

Detailed schema is in `/docs/architecture/016-data-model.md` and the
authoritative source is `prisma/schema.prisma`.

```
User (id, email, password_hash, role[ADMIN|INSTRUCTOR|STUDENT],
      name, avatar_url, timezone, bio, email_verified_at,
      created_at, updated_at, suspended_at?)

InstructorProfile (user_id PK/FK, headline, monthly_token_budget,
                   bio_long, social_links_json)

Course (id, owner_instructor_id, slug, title, description,
        learning_objectives_json, status[DRAFT|PUBLISHED|ARCHIVED],
        created_at, updated_at)
  └── Module (id, course_id, order, title, description)
        └── Topic (id, module_id, order, title, description, content_md)

Batch (id, course_id, owner_instructor_id, name, slug, start_date, end_date,
       default_meeting_url, default_timezone, capacity,
       status[UPCOMING|ONGOING|COMPLETED|CANCELLED], created_at)
  └── Session (id, batch_id, scheduled_at_utc, duration_min, agenda_md,
                meeting_url_override?)

EnrollmentRequest (id, batch_id, student_user_id, message?,
                   status[PENDING|APPROVED|REJECTED|WITHDRAWN],
                   instructor_remark?, decided_by_user_id?, decided_at?)

Enrollment (id, batch_id, student_user_id, joined_at, removed_at?,
            removed_reason?)

Material (id, scope[COURSE|BATCH], course_id?, batch_id?,
          owner_instructor_id, source[MANUAL_UPLOAD|LLM_CHAT_EXPORT|EXTERNAL_LINK],
          title, file_ref?, content_md?, external_url?, created_at)

Task (id, scope[BATCH|COURSE_TEMPLATE], batch_id?, course_id?,
      owner_instructor_id, title, description_md, learning_objectives_json,
      type[REGULAR|PRE_CLASS_GATED], deadline_utc?,
      gate_session_id?, mandatory_for_all, pass_threshold_pct?,
      promoted_from_task_id?,
      created_at, updated_at)

TaskAssignment (id, task_id, student_user_id?  -- NULL = whole batch)

QuestionPool (id, task_id, generated_at, generation_model,
              generation_prompt_hash, status[DRAFT|REVIEWED|PUBLISHED])
  └── Question (id, pool_id, type[MCQ|TEXT], difficulty[EASY|MED|HARD],
                payload_json {question, options?, correct_idx?, rubric_json},
                instructor_edited)

AssessmentAttempt (id, task_id, student_user_id, started_at, completed_at?,
                   final_score?, instructor_override_score?, status)
  └── AttemptTurn (id, attempt_id, turn_idx, question_id?,
                   role[SYSTEM|ASSESSOR|STUDENT], content,
                   tool_calls_json?, tokens_in, tokens_out,
                   model, portkey_trace_id, created_at)

Discussion (id, scope[TASK|SESSION], task_id?, session_id?,
            author_user_id, body_md, created_at, edited_at?)
  └── DiscussionReply (id, discussion_id, author_user_id, body_md, created_at)

Notification (id, user_id, kind, payload_json, read_at?, created_at)

LlmUsage (id, user_id, role, feature[ASSESSMENT|PLANNING|CHAT|QUESTION_GEN],
          model, input_tokens, output_tokens, cost_usd_cents,
          portkey_trace_id, request_id, created_at)

AuditLog (id, actor_user_id, action, target_type, target_id,
          metadata_json, created_at)
```

All `*_at` columns are `timestamptz` storing UTC. Times are displayed in the
user's TZ on the FE. See ADR-014.

### Visibility & lifecycle notes

- **Catalog visibility**: `GET /courses` returns rows where
  `status = 'PUBLISHED'` OR (`status = 'DRAFT'` AND
  `owner_instructor_id = current_user_id`). Drafts are visible to their
  owner only, rendered with a "DRAFT" badge in the FE.
- **Course-scope materials are inherited by every batch** of that course
  automatically. `GET /batches/:id/materials` returns batch-scope rows ∪
  parent course-scope rows. Batch-scope materials are visible only to that
  batch.
- **Cancelled batches**: setting `Batch.status = CANCELLED` does not delete
  `Session` rows. They remain in the ICS feed (with `STATUS:CANCELLED`)
  until the originally-scheduled time passes; only after that point are
  they hidden from dashboards and feeds. Enrollments are soft-archived.
- **Promoted task lineage**: `Task.promoted_from_task_id` (nullable FK to
  `Task`) is set on a `COURSE_TEMPLATE` task when an instructor promotes a
  batch task. Re-promoting the same source task no-ops on the existing
  template. Inherited copies into a new batch reset `deadline_utc` to
  `NULL`; the instructor must set it before the batch is published.
- **Pass threshold**: `Task.pass_threshold_pct` (nullable, 0–100) is
  required for `PRE_CLASS_GATED` tasks. `ReadinessService` (Phase 5) uses
  it to compute "ready / not ready" flags.
- **Question subset selection**: `AssessmentAttempt.start` selects a
  stratified random subset by difficulty (configurable per task; default
  for `subset_size = 5`: 2 EASY / 2 MED / 1 HARD).
- **Saved chat snippet attribution**: `Material.attribution_json`
  (when `source = LLM_CHAT_EXPORT`) contains the full originating prompt,
  model, generated_at, and portkey_trace_id — enables future audit and
  Phase 8 RAG provenance.

---

## 6. LLM strategy

### Gateway

All LLM calls — every single one — go through:

```
NestJS service
  └─ packages/llm/aiClient
       └─ Vercel AI SDK
            └─ baseURL = Portkey gateway
                 └─ primary model (configured via Portkey virtual key)
                      ↳ fallback model on error (Portkey routing rule)
```

There is **no direct provider import** outside `packages/llm/`. Lint rule
enforces this (no-restricted-imports).

### Agents

Two agents, both implemented in `packages/llm/agents/`:

**`PlannerAgent`** — used by instructors (Phase 3+) for course/lesson planning.
- Tools (allowlisted, Zod-validated):
  - `searchExistingMaterials(courseId, query)`
  - `generateOutline(topic, learningObjectives)`
  - `draftTaskContent(topic, taskType)`
  - `proposeQuestionPool(taskId, n, difficultyMix)`
- Output schema: every tool returns Zod-validated structured data.
- Guardrails: max turns per session, max tokens per turn, system prompt
  forbids executing user-supplied instructions verbatim, output sanitized
  before display (no script tags in MD).

**`AssessorAgent`** — used during student assessments (Phase 5).
- Tools:
  - `getNextQuestion(attemptId)`
  - `scoreAnswer(questionId, answer, rubric)`
  - `askFollowUp(reason)` — capped at 2 per text question
  - `finalizeAttempt(attemptId)`
- Strict invariants enforced in code (not prompt):
  - Cannot reveal correct answers to MCQ.
  - Cannot change the question payload.
  - Cannot grant a passing score without going through `scoreAnswer`.
- Prompt injection defenses:
  - Student input is wrapped in `<student_input>` and the system prompt
    instructs the agent to treat anything inside as data, never as
    instructions.
  - All tool arguments validated against Zod schemas before execution.
  - Rate-limited turn loop.

### Token tracking & budgets

Every call writes a row to `LlmUsage`. Middleware in the LLM client:

1. Reads the current user from request context.
2. Looks up their role + per-feature daily/monthly cap.
3. Reserves an estimated token amount.
4. After the call, writes actual usage and refunds the reserve.
5. Hard rejects further calls when cap reached (returns 429 with budget info).

Buckets: `ASSESSMENT`, `PLANNING`, `CHAT`, `QUESTION_GEN`.

Dashboards:
- Instructor view: own usage by feature, by day, vs budget.
- Admin view: all instructors + global totals + cost USD.

### Local-dev mode

`LLM_PROVIDER=ollama` in `.env` swaps the Portkey gateway URL for the local
Ollama OpenAI-compatible endpoint. Same `aiClient` surface; everything else
unchanged. Token tracking still runs (local cost = $0).

---

## 7. Configuration & secrets

- All env vars validated by Zod at boot via `packages/config`.
- Boot fails loud if required vars are missing — never silently default.
- `.env.example` is the source of truth for what's available.
- Local secrets in `.env` (gitignored). Prod secrets via deploy target's
  secret manager (TBD in Phase 7).

Key flags:
- `STORAGE_DRIVER` ∈ `local | minio | gcs`
- `MAILER_DRIVER` ∈ `gmail-smtp | provider-api`
- `LLM_PROVIDER` ∈ `portkey | ollama` (portkey is prod default)
- `LLM_PRIMARY_MODEL`, `LLM_FALLBACK_MODEL`
- `OTEL_EXPORTER_OTLP_ENDPOINT`

---

## 8. Phasing & docs

The project ships in phases. Each phase has a doc in
`/docs/development/phase_<N>.md` with: goals, deliverables, data model deltas,
API surface, test plan, exit criteria. **Do not start Phase N+1 until Phase N
exit criteria are green.**

| #  | Phase                      | Doc                                     |
| -- | -------------------------- | --------------------------------------- |
| 0  | Foundation                 | `docs/development/phase_0.md`           |
| 1  | Auth & Users               | `docs/development/phase_1.md`           |
| 2  | Courses, Batches, Enrollment | `docs/development/phase_2.md`         |
| 3  | Materials & Planner LLM    | `docs/development/phase_3.md`           |
| 4  | Tasks & Discussions        | `docs/development/phase_4.md`           |
| 5  | AI Assessment              | `docs/development/phase_5.md`           |
| 6  | Notifications & PWA polish | `docs/development/phase_6.md`           |
| 7  | Hardening & CI/CD          | `docs/development/phase_7.md`           |
| 8  | Future (Keycloak, RAG, app stores, payments, ELK) | `docs/development/phase_8.md` |

Architecture decision records (ADRs) live in `/docs/architecture/`, numbered
`001-*.md` through `020-*.md`. Each is short, MADR-format, and linked from the
relevant section here.

---

## 9. Conventions

### Code style
- **TypeScript strict mode everywhere.** No `any` without an inline justification.
- **Zod schemas** are the single source of truth for shapes shared across the
  FE/BE boundary, validated input, and LLM tool args.
- Prisma client is the only path to the DB. No raw SQL except in migrations
  or rare perf-critical aggregates (must be commented).
- API responses follow a small set of conventions: 2xx on success with the
  resource body; 4xx with `{ error: { code, message, details? } }`.
- UTC in DB. Localized in the FE.

### Folder/file naming
- `kebab-case` for files (`enrollment-request.controller.ts`).
- `PascalCase` for classes, types, components.
- One exported entity per file when reasonable; co-locate small helpers.

### Testing
- Unit tests next to source (`*.spec.ts`).
- Integration tests under `apps/api/test/integration/` use testcontainers
  for real Postgres.
- e2e tests under `apps/web/e2e/` use Playwright against
  `docker compose up --wait`.
- Every phase doc lists the e2e flows that must pass before the phase is
  considered shippable.

### Git
- Trunk-based on `main`. Feature branches → PR → review → squash merge.
- Conventional commits (`feat:`, `fix:`, `docs:`, `chore:`, `test:`, `refactor:`).
- Never commit `.env`. Never commit real LLM keys (Portkey virtual keys are
  preferred so a leak is revocable without rotating provider keys).

### Comments
- Default to **no comments**. Names should explain the what.
- Comment **why** when behaviour is non-obvious, when there's a workaround,
  or when an invariant is enforced in code that a reader could mistake for
  optional.

---

## 10. Working agreement (for Claude / contributors)

- Read `CLAUDE.md` and the relevant phase doc before any non-trivial change.
- Don't introduce dependencies the spec doesn't already approve. If
  something seems missing, propose it as an ADR addition first.
- Don't break the provider-agnostic boundary. If you reach for a vendor
  SDK in business logic, stop and route through the abstraction.
- Don't mix phases. If Phase 2 work touches Phase 4 surfaces, raise it
  rather than silently building ahead.
- Keep `.env.example` in sync with config. CI will eventually enforce this.
- Keep ADRs in sync with reality. If you change a decision, update or
  supersede the ADR — don't leave it lying.

---

## 11. Quickstart (will be completed in Phase 0)

```bash
pnpm install
cp .env.example .env
docker compose up -d        # postgres, mailpit, minio, ollama, lgtm
pnpm db:migrate
pnpm db:seed                # creates ADMIN from env
pnpm dev                    # runs api + web with turbo
```

Then:
- Web: http://localhost:3000
- API: http://localhost:4000 (Swagger at `/api/docs`)
- Mailpit: http://localhost:8025
- MinIO: http://localhost:9001
- Grafana: http://localhost:3001
- Portkey UI: cloud (or self-hosted, configured in Phase 3)

---

## 12. Out of scope (for now)

- Multi-tenancy / organizations.
- Live video conferencing (we store a meeting URL only).
- Payments / billing (receipts on enrollment moved to Phase 8).
- Native mobile apps (PWA only until Capacitor wrapper in Phase 8).
- Marketplace / instructor payouts.
- Content moderation pipeline beyond basic input sanitization.
