# Phase 3 — Materials & Planner LLM

## Goal

Bring the LLM into the platform — for instructors only, behind strict
guardrails — and ship the materials layer that lets them attach files,
links, and AI-generated content to courses and batches. By the end of
Phase 3, an instructor can chat with a `PlannerAgent` to draft outlines,
task content, and question-pool proposals; save useful snippets as
`Material` records with attribution; upload PDFs / slide decks; and
watch their token usage against a per-month budget set by the admin at
invite time. Students do **not** get free-form LLM chat in this phase
— their first LLM exposure is the `AssessorAgent` in Phase 5. This
keeps the budget exposure bounded while we tune prompts and observability.

## Scope (in)

- `packages/llm/aiClient` wired to **Portkey** as the default gateway,
  with `LLM_PROVIDER=ollama` swap for local dev (CLAUDE.md §6).
- Token-tracking middleware that wraps every LLM call and writes
  `LlmUsage` rows.
- Per-instructor monthly token budget enforcement (returns 429 with a
  `budget` payload when exceeded).
- `PlannerAgent` with four tools, Zod-validated, allowlisted, turn-cap
  enforced.
- SSE streaming endpoint for the planner chat UI.
- `Material` table + CRUD endpoints scoped to either a course or batch
  (owner-only, admin override).
- Three material sources: `MANUAL_UPLOAD` (storage), `LLM_CHAT_EXPORT`
  (saved from chat), `EXTERNAL_LINK`.
- File upload through `packages/storage` with size/type validation,
  signed download URLs.
- "Save to material" action in the chat UI: turns a chat snippet into
  a `Material` with attribution metadata (model, prompt summary, date).
- Token dashboards: instructor (own usage) and admin (global) using
  Recharts on the FE.
- OTel spans for every LLM call (model, tokens, cost, portkey trace id).
- Audit log entries for material create/update/delete and budget-cap hits.

## Scope (out / deferred)

- Student-facing LLM chat — never (replaced by `AssessorAgent` in P5).
- `AssessorAgent` and assessment flows — Phase 5.
- Tasks, question pools as persisted entities — Phase 4 (the Planner
  can *propose* a question pool, but the pool isn't persisted as a
  `QuestionPool` row in P3; it's a draft returned to the chat).
- RAG / vector search over uploaded materials — Phase 8.
- Multi-modal LLM (image / audio inputs) — out.
- Material moderation pipeline — out (basic input sanitization only).
- Prompt library / saved prompts — out.
- Cost-allocation across team budgets — out (single instructor budget
  only; aggregate visible to admin).

## Prerequisites

- Phase 0 complete (`packages/llm` skeleton, `packages/storage`,
  `packages/observability`, `packages/config`).
- Phase 1 complete (auth, roles, audit log, instructor invites carry
  `monthly_token_budget`).
- Phase 2 complete (`Course`, `Batch` exist for material scoping).

## Deliverables

- Real implementation of `aiClient.streamText({...})`,
  `aiClient.generateObject({...})` against Portkey/Ollama.
- `LlmUsageInterceptor` (NestJS interceptor) capturing token + cost on
  every call.
- `BudgetGuard` middleware that pre-flights and rejects at the cap.
- `PlannerAgent` class with: system prompt, tool registry, turn loop,
  output sanitizer (strip `<script>`, etc.).
- Migration adding `Material` and `LlmUsage`.
- NestJS `MaterialsModule`, `PlannerModule`, `LlmUsageModule`.
- FE: planner chat page, materials manager (per course / per batch),
  material upload modal, "save snippet" flow, token dashboards.
- Grafana dashboard panel for global LLM cost & latency (provisioned in
  `infra/grafana/`).
- Portkey config (virtual key, primary + fallback model) documented in
  `docs/architecture/`.
- OpenAPI updated; FE types regenerated.

## Data model deltas

Reference CLAUDE.md §5. Phase 3 adds:

```prisma
enum MaterialScope {
  COURSE
  BATCH
}

enum MaterialSource {
  MANUAL_UPLOAD
  LLM_CHAT_EXPORT
  EXTERNAL_LINK
}

model Material {
  id                  String          @id @default(cuid())
  scope               MaterialScope
  courseId            String?         @map("course_id")
  batchId             String?         @map("batch_id")
  ownerInstructorId   String          @map("owner_instructor_id")
  source              MaterialSource
  title               String
  description         String?         @db.Text
  fileRef             String?         @map("file_ref")          // storage key, when MANUAL_UPLOAD
  fileMime            String?         @map("file_mime")
  fileSizeBytes       Int?            @map("file_size_bytes")
  contentMd           String?         @map("content_md") @db.Text // when LLM_CHAT_EXPORT
  externalUrl         String?         @map("external_url")        // when EXTERNAL_LINK
  attributionJson     Json?           @map("attribution_json")    // see "Attribution shape" below
  createdAt           DateTime        @default(now()) @map("created_at")
  updatedAt           DateTime        @updatedAt      @map("updated_at")

  owner   User    @relation(fields: [ownerInstructorId], references: [id])
  course  Course? @relation(fields: [courseId],          references: [id], onDelete: Cascade)
  batch   Batch?  @relation(fields: [batchId],           references: [id], onDelete: Cascade)

  @@index([courseId])
  @@index([batchId])
  @@index([ownerInstructorId])
  @@map("materials")
}

enum LlmFeature {
  ASSESSMENT     // P5
  PLANNING       // P3 — Planner chat
  CHAT           // reserved for future surfaces
  QUESTION_GEN   // Planner proposing pools (P3) and gen flows (P4+)
}

model LlmUsage {
  id              String      @id @default(cuid())
  userId          String      @map("user_id")
  role            UserRole
  feature         LlmFeature
  model           String                                  // e.g. "openai/gpt-4o-mini"
  inputTokens     Int         @map("input_tokens")
  outputTokens   Int          @map("output_tokens")
  costUsdCents    Int         @map("cost_usd_cents")     // local-dev: 0
  portkeyTraceId  String?     @map("portkey_trace_id")
  requestId       String      @map("request_id")
  createdAt       DateTime    @default(now()) @map("created_at")

  user User @relation(fields: [userId], references: [id])

  @@index([userId, createdAt])
  @@index([feature, createdAt])
  @@index([portkeyTraceId])
  @@map("llm_usage")
}
```

Migration SQL additions (raw):

```sql
-- Enforce that Material has exactly the fields its source requires.
ALTER TABLE materials ADD CONSTRAINT materials_source_payload_chk
  CHECK (
    (source = 'MANUAL_UPLOAD'   AND file_ref IS NOT NULL AND external_url IS NULL AND content_md IS NULL)
    OR (source = 'LLM_CHAT_EXPORT' AND content_md IS NOT NULL AND file_ref IS NULL AND external_url IS NULL)
    OR (source = 'EXTERNAL_LINK'   AND external_url IS NOT NULL AND file_ref IS NULL AND content_md IS NULL)
  );

-- Enforce scope/foreign-key consistency.
ALTER TABLE materials ADD CONSTRAINT materials_scope_chk
  CHECK (
    (scope = 'COURSE' AND course_id IS NOT NULL AND batch_id IS NULL)
    OR (scope = 'BATCH' AND batch_id IS NOT NULL AND course_id IS NULL)
  );
```

**Attribution shape** (resolved per CLAUDE.md §5 "Visibility & lifecycle
notes"): when `source = LLM_CHAT_EXPORT`, `attribution_json` stores the
**full** originating prompt — not a summary — so we have the audit trail
for prompt-engineering review and Phase 8 RAG provenance.

```ts
type LlmChatExportAttribution = {
  model: string;             // e.g. "openai/gpt-4o-mini"
  fullPrompt: string;        // verbatim user message that produced this snippet
  generatedAt: string;       // ISO-8601 UTC
  portkeyTraceId: string | null;
  plannerSessionId: string;  // back-reference to PlannerSession
  plannerMessageId: string;  // back-reference to the assistant message
};
```

The `InstructorProfile.monthlyTokenBudget` from Phase 1 is now actively
read by the budget enforcement middleware.

**Material visibility for batches** (resolved per CLAUDE.md §5):
`GET /batches/:id/materials` returns the union of (a) batch-scope
materials directly attached to that batch and (b) course-scope materials
attached to the batch's parent course. Course-scope materials are
inherited by every batch automatically — no per-batch attach action is
required. Implementation: a single SQL query with `WHERE
(scope = 'BATCH' AND batch_id = :id) OR (scope = 'COURSE' AND course_id = :batch.course_id)`.

## API surface

### Materials — course-scope

| Method | Path                                        | Purpose                              |
| ------ | ------------------------------------------- | ------------------------------------ |
| POST   | `/courses/:id/materials`                    | Create (any source)                  |
| GET    | `/courses/:id/materials`                    | List for the course                  |
| GET    | `/courses/:id/materials/:materialId`        | Detail (incl. signed download URL if MANUAL_UPLOAD) |
| PATCH  | `/courses/:id/materials/:materialId`        | Update title/description/external URL|
| DELETE | `/courses/:id/materials/:materialId`        | Delete (cascade removes storage obj) |
| POST   | `/courses/:id/materials/upload-url`         | Issue pre-signed upload URL (MinIO/GCS); local driver uses direct multipart |

### Materials — batch-scope

Same endpoints under `/batches/:id/materials/...` with the same semantics
and owner-only auth.

### Planner chat

| Method | Path                                            | Purpose                                          |
| ------ | ----------------------------------------------- | ------------------------------------------------ |
| POST   | `/planner/sessions`                             | Create a chat session (returns id)               |
| GET    | `/planner/sessions`                             | List own chat sessions                           |
| GET    | `/planner/sessions/:id`                         | Read a session's transcript                      |
| POST   | `/planner/sessions/:id/messages`                | Send a user message; SSE stream of assistant + tool events |
| POST   | `/planner/sessions/:id/save-snippet`            | Save a transcript range as a `Material` (LLM_CHAT_EXPORT) |
| DELETE | `/planner/sessions/:id`                         | Delete a session                                 |

SSE event shapes:
- `event: token` — partial text chunk.
- `event: tool_call` — `{ tool, args }` (Zod-validated).
- `event: tool_result` — `{ tool, result }`.
- `event: usage` — `{ inputTokens, outputTokens, costUsdCents, model }`.
- `event: done` — terminal.
- `event: error` — `{ code, message }`.

Note: chat sessions and messages are persisted in lightweight tables
inside the planner module (`PlannerSession`, `PlannerMessage`) — kept
out of CLAUDE.md §5 because they're surface-internal. Schema:

```prisma
model PlannerSession {
  id                  String           @id @default(cuid())
  ownerInstructorId   String           @map("owner_instructor_id")
  scopeCourseId       String?          @map("scope_course_id")
  scopeBatchId        String?          @map("scope_batch_id")
  title               String
  createdAt           DateTime         @default(now()) @map("created_at")
  updatedAt           DateTime         @updatedAt      @map("updated_at")

  owner    User             @relation(fields: [ownerInstructorId], references: [id])
  messages PlannerMessage[]

  @@index([ownerInstructorId, updatedAt])
  @@map("planner_sessions")
}

model PlannerMessage {
  id            String   @id @default(cuid())
  sessionId     String   @map("session_id")
  turnIdx       Int      @map("turn_idx")
  role          String                                  // "user" | "assistant" | "tool"
  content       String   @db.Text
  toolCallsJson Json?    @map("tool_calls_json")
  usageJson     Json?    @map("usage_json")
  createdAt     DateTime @default(now()) @map("created_at")

  session PlannerSession @relation(fields: [sessionId], references: [id], onDelete: Cascade)

  @@unique([sessionId, turnIdx])
  @@index([sessionId])
  @@map("planner_messages")
}
```

### Token usage

| Method | Path                       | Purpose                                |
| ------ | -------------------------- | -------------------------------------- |
| GET    | `/me/usage`                | Instructor's own usage (filters: from, to, feature) |
| GET    | `/me/usage/summary`        | Current period vs budget               |
| GET    | `/admin/usage`             | Global usage feed                      |
| GET    | `/admin/usage/summary`     | Per-instructor totals + global         |

### Auth requirements

| Group                                | Guard                                                                |
| ------------------------------------ | -------------------------------------------------------------------- |
| Material endpoints                   | `JwtAuthGuard` + `RolesGuard('INSTRUCTOR' | 'ADMIN')` + `OwnerInstructorGuard` (course or batch owner) |
| `/planner/*`                         | `JwtAuthGuard` + `RolesGuard('INSTRUCTOR' | 'ADMIN')` + `BudgetGuard` |
| `/me/usage*`                         | `JwtAuthGuard` (any role; STUDENT sees zero rows in P3)              |
| `/admin/usage*`                      | `JwtAuthGuard` + `RolesGuard('ADMIN')`                               |

`BudgetGuard` reads `req.user.monthlyTokenBudget`, sums
`LlmUsage.input_tokens + output_tokens` over the current calendar month
(UTC), and returns 429 when the projected request would exceed it.
Response body:

```json
{
  "error": {
    "code": "TOKEN_BUDGET_EXCEEDED",
    "message": "Monthly token budget reached.",
    "details": {
      "monthlyBudget": 1000000,
      "usedThisMonth": 999500,
      "periodEndsAt": "2026-06-01T00:00:00Z"
    }
  }
}
```

### `PlannerAgent` tools

All tool args validated by Zod before execution. Outputs are also
schema-validated; agent retries up to 2 times on validation failure
before surfacing an error.

| Tool                          | Args (Zod)                                                                  | Output                                                  |
| ----------------------------- | --------------------------------------------------------------------------- | ------------------------------------------------------- |
| `searchExistingMaterials`     | `{ courseId: string, query: string, limit?: number(<=20) }`                 | `{ items: { id, title, snippet }[] }` (substring match in P3) |
| `generateOutline`             | `{ topic: string, learningObjectives: string[].nonempty() }`                | `{ outline: { module: string, topics: string[] }[] }`   |
| `draftTaskContent`            | `{ topic: string, taskType: 'concept' | 'practical' | 'reflection' }`       | `{ titleSuggestion: string, descriptionMd: string }`    |
| `proposeQuestionPool`         | `{ taskTopic: string, n: number(min:3,max:30), difficultyMix: { easy: number, med: number, hard: number } }` | `{ questions: { type: 'MCQ' | 'TEXT', difficulty, payload }[] }` |

Guardrails (CLAUDE.md §6):
- Max 12 turns per session.
- Max 4000 output tokens per turn.
- System prompt forbids executing user-supplied instructions verbatim.
- Markdown sanitizer strips `<script>`, JS event handlers, and
  `javascript:` URLs before persisting/streaming.
- Tool results are passed back as tool messages, never spliced into
  the user content.

## UI surface

| Route                                                  | Purpose                                                  | Role gating                |
| ------------------------------------------------------ | -------------------------------------------------------- | -------------------------- |
| `/instructor/courses/:id/materials`                    | Course-scope materials manager                           | owner / ADMIN              |
| `/instructor/batches/:id/materials`                    | Batch-scope materials manager                            | owner / ADMIN              |
| `/instructor/planner`                                  | Chat list + new chat                                     | INSTRUCTOR / ADMIN         |
| `/instructor/planner/:sessionId`                       | Chat thread with streaming, tool calls, save-snippet UI  | owner / ADMIN              |
| `/me/usage`                                            | Instructor token dashboard (Recharts)                    | authenticated              |
| `/admin/usage`                                         | Global token + cost dashboard                            | ADMIN                      |

Charts:
- Per-day stacked bar by feature (PLANNING / QUESTION_GEN / etc.).
- Cumulative-vs-budget line for the current month.
- Top-N instructors table (admin only).

Material upload uses the storage abstraction; for MinIO/GCS the FE
fetches a pre-signed URL and PUTs directly; for local driver the FE
posts multipart to the API.

## Configuration

New env vars on top of Phase 2:

| Var                              | Required          | Default                              | Notes                                         |
| -------------------------------- | ----------------- | ------------------------------------ | --------------------------------------------- |
| `LLM_PROVIDER`                   | yes (declared P0) | `ollama` (dev), `portkey` (prod)     |                                               |
| `LLM_PRIMARY_MODEL`              | yes               | `llama3.1:8b` (dev) / e.g. `openai/gpt-4o-mini` (prod) |                                  |
| `LLM_FALLBACK_MODEL`             | no (prod-recommended) | —                                | Portkey routing rule fires on primary error   |
| `PORTKEY_BASE_URL`               | if portkey        | `https://api.portkey.ai/v1`          |                                               |
| `PORTKEY_API_KEY`                | if portkey        | —                                    | Portkey master key                            |
| `PORTKEY_VIRTUAL_KEY`            | if portkey        | —                                    | Per-tenant virtual key                        |
| `OLLAMA_BASE_URL`                | if ollama         | `http://localhost:11434/v1`          |                                               |
| `LLM_MAX_TURNS_PER_SESSION`      | yes               | `12`                                 |                                               |
| `LLM_MAX_OUTPUT_TOKENS_PER_TURN` | yes               | `4000`                               |                                               |
| `LLM_BUDGET_RESERVE_TOKENS`      | yes               | `2000`                               | Pre-call reserve, refunded after actual usage |
| `LLM_DEFAULT_PRICE_PER_1K_INPUT_USD_CENTS`  | yes    | `15`                                 | Used when Portkey doesn't return cost         |
| `LLM_DEFAULT_PRICE_PER_1K_OUTPUT_USD_CENTS` | yes    | `60`                                 |                                               |
| `MATERIAL_MAX_FILE_MB`           | yes               | `50`                                 |                                               |
| `MATERIAL_ALLOWED_MIMES`         | yes               | `application/pdf,application/vnd.openxmlformats-officedocument.presentationml.presentation,image/png,image/jpeg,text/markdown` | CSV |

Boot fails loud if `LLM_PROVIDER=portkey` and any portkey-prefixed var
is missing, or if `LLM_PROVIDER=ollama` and `OLLAMA_BASE_URL` is
unreachable (warn-only in dev, fatal in prod).

## Testing plan

### Unit tests

- `aiClient` request shape: ensures Portkey headers
  (`x-portkey-virtual-key`) are present in portkey mode and absent in
  ollama mode.
- Token estimator (used by `BudgetGuard` for the pre-call reserve)
  bounds within ±20% on a fixture corpus.
- Markdown sanitizer strips `<script>`, `javascript:`,
  `on{event}=` handlers; preserves valid markdown.
- Tool arg validators reject malformed input and return a structured
  error the agent can recover from.
- Material constraint logic: cannot create COURSE-scope with a batchId,
  and source ↔ payload combinations are enforced.

### Integration tests (testcontainers)

- End-to-end planner turn against a **mocked LLM provider** (small
  in-process server returning deterministic SSE chunks): tokens
  recorded, `LlmUsage` row written, audit log row written.
- Budget hit: set instructor budget to 100 tokens; second call returns
  429 with the structured payload.
- Save-snippet: a chat range with assistant + tool turns becomes a
  single `Material` (LLM_CHAT_EXPORT) with attribution metadata.
- File upload via local storage: PDF round-trips; signed URL works;
  delete removes the underlying file.
- Fallback on primary error: mock provider raises 500; Portkey
  fallback (simulated) succeeds; usage row records the model that
  actually served the request.

### E2E (Playwright)

Named golden flows that must pass:

1. `instructor-chats-with-planner-and-streams`: instructor opens
   `/instructor/planner`, starts a session scoped to a course, asks
   for an outline; tokens stream into the UI; tool call panels render.
2. `instructor-saves-snippet-to-material`: from the chat, instructor
   highlights a section, clicks "Save to material", picks course
   scope; a new MATERIAL appears under that course with attribution.
3. `instructor-uploads-pdf-material`: instructor uploads a PDF on a
   course; download URL works from the materials list.
4. `token-budget-blocks-when-exceeded`: admin sets a low budget on
   an instructor; instructor's next planner message returns the
   429 payload and the UI renders a friendly upgrade-budget message.
5. `fallback-model-engages-on-primary-error` (mocked at the gateway):
   primary fails; fallback succeeds; the assistant message renders
   normally; the `usage` event reports the fallback model.
6. `student-cannot-access-planner`: student tries
   `POST /planner/sessions` and gets 403; the `/instructor/planner`
   route is not in their nav.

## Exit criteria

- [ ] Every LLM call writes exactly one `LlmUsage` row with non-null
      `request_id` and `portkey_trace_id` (in portkey mode).
- [ ] Zero direct provider SDK imports outside `packages/llm/` (CI
      guard from Phase 0 still green with new code added).
- [ ] `BudgetGuard` blocks calls deterministically; refund logic
      proven by a test where reserved > actual.
- [ ] All Playwright flows above pass.
- [ ] Material constraints (scope, source/payload) enforced at DB
      level — proven by negative integration tests.
- [ ] Coverage on `packages/llm` ≥ 75% lines (the agent loop is the
      single most testworthy piece of the codebase).
- [ ] Token dashboards render correct numbers for a seeded usage
      fixture (instructor view + admin view).
- [ ] Grafana panel for global LLM cost & latency provisioned and
      populated by the OTel pipeline.
- [ ] Switching `LLM_PROVIDER=ollama → portkey` requires only env
      changes; no code changes.

## Risks & open questions

- **Cost accuracy**: Portkey returns cost in many cases but not all.
  The fallback per-1K pricing constants must be reviewed monthly; flag
  drift by alerting if Portkey-reported cost diverges from the
  estimator by >25%.
- **Prompt injection in saved materials**: a `LLM_CHAT_EXPORT` material
  becomes part of future planner context (in Phase 8 RAG). Sanitize at
  save time and again at retrieve time. P3 only sanitizes at save.
- **Streaming over SSE in Next.js 15 App Router**: confirm that `fetch`
  with `ReadableStream` works through the route handler proxy or that
  we need a direct browser-to-API stream; either is acceptable, but
  it's a known papercut.
- **Local Ollama latency**: `llama3.1:8b` planner sessions can be slow
  on developer laptops without a GPU. Document expected latency in
  the README; consider a `LLM_PROVIDER=stub` mode that returns canned
  outlines, useful for E2E speed.
- **Resolved**: saved chat snippets store the **full** originating
  prompt in `attribution_json` (alongside model, generatedAt,
  portkey_trace_id, plannerSessionId, plannerMessageId). Required for
  audit and for Phase 8 RAG provenance.
- **Resolved**: course-scope materials are auto-visible to every batch
  under that course; batch-scope materials are visible only to their
  batch. `GET /batches/:id/materials` returns the union (batch ∪
  parent course) in one query.
