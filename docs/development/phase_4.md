# Phase 4 — Tasks & Discussions

## Goal

Deliver the work-and-conversation layer of the platform: instructors create
tasks for their batches, optionally promote a successful task into a course
template that future batches inherit, and assign tasks either to the whole
batch or to specific students. Students see only their own task list with
deadlines and a per-task discussion thread. This is the right slice now
because Phases 0–3 have established users, batches, materials and the
PlannerAgent, and Phase 5 (AI Assessment) cannot start until Tasks exist as
a first-class entity that question pools and attempts attach to.

This phase intentionally ships **regular tasks and pre-class-gated tasks
without any auto-grading**. Instructors can read student responses (where
applicable) but no AI assessor runs yet — that arrives in Phase 5.

---

## Scope (in)

- `Task`, `TaskAssignment`, `Discussion`, `DiscussionReply` Prisma models
  (CLAUDE.md §5).
- Instructor CRUD on tasks scoped to their own batches.
- Promote a batch task to a `COURSE_TEMPLATE` so future batches of that
  course inherit it at batch-creation time (snapshot copy, not reference).
- Assign tasks to whole batch (`student_user_id = NULL`) or to specific
  students.
- `PRE_CLASS_GATED` tasks with `gate_session_id` and a deadline validation
  rule: deadline must be ≥ 1 hour before `gate_session.scheduled_at_utc`.
- Student endpoint `GET /me/tasks` returning only their assignments with a
  countdown to deadline.
- Discussion threads attached to either a `Task` or a `Session`, with
  threaded replies, markdown body, current user as author.
- Read/post permission: any user enrolled in the relevant batch (or the
  owning instructor / any admin) may read and post.
- Markdown rendering pipeline: server-side parser with raw HTML disabled;
  client-side `rehype-sanitize` (or DOMPurify for non-rehype paths) to
  block script tags, event handlers, `javascript:` URLs.
- E2E flows listed below.

## Scope (out / deferred)

- Any AI grading / assessor flow (Phase 5).
- `QuestionPool`, `Question`, `AssessmentAttempt`, `AttemptTurn` tables
  (Phase 5).
- Notifications when a task is assigned or a deadline approaches (Phase 6).
  We **write** the data but do not push email/in-app yet.
- Hard gating of session attendance (still soft signal only; Phase 5
  introduces the readiness flag once attempts exist).
- Discussion forums per course (Phase 8).
- Real-time updates on discussion threads (polling is fine for now).
- File attachments on tasks or replies (deferred; markdown links only).

## Prerequisites

- Phase 0 (foundation, monorepo, docker-compose, config loader).
- Phase 1 (auth, JWT, role guards).
- Phase 2 (Course, Batch, Enrollment — task scope and ownership rely on
  these).
- Phase 3 (Materials & PlannerAgent — instructors will reference materials
  from task descriptions; PlannerAgent's `draftTaskContent` tool is
  available).

## Deliverables (concrete artifacts)

- Prisma migration adding `Task`, `TaskAssignment`, `Discussion`,
  `DiscussionReply` tables and required enums.
- NestJS modules: `TasksModule`, `DiscussionsModule`.
- Zod schemas in `packages/shared`:
  - `TaskCreateInput`, `TaskUpdateInput`, `TaskAssignInput`,
    `TaskPromoteToTemplateInput`.
  - `DiscussionCreateInput`, `DiscussionReplyCreateInput`.
- Frontend routes (App Router):
  - `/instructor/batches/:id/tasks` (list + create).
  - `/instructor/batches/:id/tasks/:taskId` (detail, edit, assign, promote).
  - `/me/tasks` (student dashboard).
  - `/tasks/:id/discussion` (any enrolled user; instructor + admin too).
- shadcn/ui components: `TaskFormDialog`, `AssignmentPicker`,
  `DeadlineCountdown`, `MarkdownEditor`, `MarkdownView`, `DiscussionThread`.
- Service: `TaskTemplateInheritanceService` — invoked from the existing
  `BatchesService.create()` (Phase 2) to copy any course-level
  `COURSE_TEMPLATE` tasks into the new batch as `BATCH`-scoped tasks.
- Markdown sanitization wrapper `packages/shared/markdown/sanitize.ts`.
- Updated OpenAPI spec + regenerated FE types via `openapi-typescript`.
- Playwright specs for the listed E2E flows.

---

## Data model deltas

Reference: CLAUDE.md §5.

```prisma
enum TaskScope {
  BATCH
  COURSE_TEMPLATE
}

enum TaskType {
  REGULAR
  PRE_CLASS_GATED
}

enum DiscussionScope {
  TASK
  SESSION
}

model Task {
  id                       String       @id @default(cuid())
  scope                    TaskScope
  batchId                  String?
  batch                    Batch?       @relation(fields: [batchId], references: [id], onDelete: Cascade)
  courseId                 String?
  course                   Course?      @relation(fields: [courseId], references: [id], onDelete: Cascade)
  ownerInstructorId        String
  ownerInstructor          User         @relation("TaskOwner", fields: [ownerInstructorId], references: [id])
  title                    String
  descriptionMd            String       @db.Text
  learningObjectivesJson   Json
  type                     TaskType     @default(REGULAR)
  deadlineUtc              DateTime?    // nullable; required before publishing the parent batch
  gateSessionId            String?
  gateSession              Session?     @relation(fields: [gateSessionId], references: [id])
  mandatoryForAll          Boolean      @default(false)
  passThresholdPct         Int?         // 0-100; required for PRE_CLASS_GATED (P5 ReadinessService reads this)
  promotedFromTaskId       String?      // FK → Task.id; set on COURSE_TEMPLATE rows produced via promote
  promotedFrom             Task?        @relation("TaskPromotion", fields: [promotedFromTaskId], references: [id])
  promotedTemplates        Task[]       @relation("TaskPromotion")
  createdAt                DateTime     @default(now())
  updatedAt                DateTime     @updatedAt

  assignments              TaskAssignment[]
  discussions              Discussion[]

  @@index([batchId])
  @@index([courseId])
  @@index([ownerInstructorId])
  @@unique([promotedFromTaskId])  // one template per source task; promote is idempotent
  // Exactly one of (batchId, courseId) must be non-null; enforced in service layer.
  // gateSessionId required iff type = PRE_CLASS_GATED; enforced in service layer.
  // passThresholdPct required iff type = PRE_CLASS_GATED; enforced in service layer.
  // deadlineUtc must be non-null on BATCH-scope tasks before parent batch transitions to PUBLISHED status (or before assignments accrue work).
}

model TaskAssignment {
  id            String   @id @default(cuid())
  taskId        String
  task          Task     @relation(fields: [taskId], references: [id], onDelete: Cascade)
  studentUserId String?  // NULL = whole batch
  student       User?    @relation("StudentAssignments", fields: [studentUserId], references: [id])
  createdAt     DateTime @default(now())

  @@unique([taskId, studentUserId])
  @@index([studentUserId])
}

model Discussion {
  id            String          @id @default(cuid())
  scope         DiscussionScope
  taskId        String?
  task          Task?           @relation(fields: [taskId], references: [id], onDelete: Cascade)
  sessionId     String?
  session       Session?        @relation(fields: [sessionId], references: [id], onDelete: Cascade)
  authorUserId  String
  author        User            @relation("DiscussionAuthor", fields: [authorUserId], references: [id])
  bodyMd        String          @db.Text
  createdAt     DateTime        @default(now())
  editedAt      DateTime?

  replies       DiscussionReply[]

  @@index([taskId])
  @@index([sessionId])
  @@index([authorUserId])
}

model DiscussionReply {
  id            String     @id @default(cuid())
  discussionId  String
  discussion    Discussion @relation(fields: [discussionId], references: [id], onDelete: Cascade)
  authorUserId  String
  author        User       @relation("ReplyAuthor", fields: [authorUserId], references: [id])
  bodyMd        String     @db.Text
  createdAt     DateTime   @default(now())

  @@index([discussionId])
  @@index([authorUserId])
}
```

Constraints enforced in service layer (cannot be expressed in Prisma DSL
directly):

- `Task` requires exactly one of `batchId` or `courseId` non-null based on
  `scope`.
- `Task` of type `PRE_CLASS_GATED` requires `gateSessionId`,
  `passThresholdPct`, and a non-null `deadlineUtc`. Validates
  `deadlineUtc <= gateSession.scheduledAtUtc - 1 hour`.
- `Discussion` requires exactly one of `taskId` or `sessionId` non-null
  based on `scope`.
- A `Task` of scope `COURSE_TEMPLATE` cannot have `TaskAssignment` rows
  (template tasks are not directly assigned; their copies into batches are).
- A `Task` of scope `COURSE_TEMPLATE` may have `deadlineUtc = NULL`
  (templates carry no schedule of their own).
- A `BATCH`-scope task with `deadlineUtc = NULL` may exist transiently
  (e.g., freshly inherited from a template); it is rejected from being
  assigned to students and rejected from being published as a gated task
  until a deadline is set.

---

## API surface

All routes are prefixed `/api/v1`. Auth via existing JWT bearer.

| Method | Path                                                       | Purpose                                                                           | Auth gate                                                |
| ------ | ---------------------------------------------------------- | --------------------------------------------------------------------------------- | -------------------------------------------------------- |
| POST   | `/batches/:batchId/tasks`                                  | Create batch-scoped task (REGULAR or PRE_CLASS_GATED).                            | INSTRUCTOR (must own batch) or ADMIN.                    |
| GET    | `/batches/:batchId/tasks`                                  | List tasks in a batch.                                                            | Owning instructor, ADMIN, or any enrolled student.       |
| GET    | `/tasks/:id`                                               | Get a task's full detail (description, assignments, attached discussion count).   | Owning instructor, ADMIN, or any enrolled student in its batch. |
| PATCH  | `/tasks/:id`                                               | Edit task fields. Re-validates deadline rule for gated tasks.                     | Owning instructor or ADMIN.                              |
| DELETE | `/tasks/:id`                                               | Delete a task and its assignments + discussions (cascades).                       | Owning instructor or ADMIN.                              |
| POST   | `/batches/:batchId/tasks/:taskId/promote-to-template`      | Snapshot the task into a `COURSE_TEMPLATE` task on the parent course.             | Owning instructor or ADMIN.                              |
| POST   | `/batches/:batchId/tasks/:taskId/assign`                   | Body: `{ wholeBatch: bool }` or `{ studentUserIds: string[] }`. Idempotent.       | Owning instructor or ADMIN.                              |
| DELETE | `/batches/:batchId/tasks/:taskId/assign/:studentUserId`    | Unassign a specific student.                                                      | Owning instructor or ADMIN.                              |
| GET    | `/me/tasks`                                                | Student view: only tasks assigned to me, with `deadlineUtc` and time remaining.   | Authenticated student.                                   |
| POST   | `/tasks/:id/discussions`                                   | Create a new discussion thread on a task.                                         | Any user enrolled in the batch + owning instructor + ADMIN. |
| GET    | `/tasks/:id/discussions`                                   | List discussion threads on a task (paginated).                                    | Same as above.                                           |
| POST   | `/sessions/:id/discussions`                                | Create a discussion thread on a session.                                          | Any user enrolled in the batch + owning instructor + ADMIN. |
| GET    | `/sessions/:id/discussions`                                | List discussion threads on a session.                                             | Same.                                                    |
| GET    | `/discussions/:id`                                         | Get a single discussion thread with its replies.                                  | Permitted readers per parent.                            |
| POST   | `/discussions/:id/replies`                                 | Append a reply to a thread. Body: `{ bodyMd }`.                                   | Permitted readers per parent.                            |
| PATCH  | `/discussions/:id`                                         | Edit own discussion (sets `editedAt`).                                            | Author or ADMIN.                                         |
| PATCH  | `/discussions/replies/:id`                                 | Edit own reply.                                                                   | Author or ADMIN.                                         |

Validation:

- All inputs Zod-validated at the controller layer using shared schemas.
- Markdown body length capped (`descriptionMd` 50 KB; reply 10 KB).
- `learningObjectivesJson` shape: `{ items: string[] }`, items 1–20.

Error contract per CLAUDE.md §9: `4xx` returns `{ error: { code, message,
details? } }`. Specific codes: `TASK_DEADLINE_TOO_LATE`,
`TASK_GATE_SESSION_REQUIRED`, `TASK_TEMPLATE_CANNOT_BE_ASSIGNED`,
`DISCUSSION_NOT_PERMITTED`.

---

## UI surface

| Route                                              | Purpose                                                               | Role gating                                  |
| -------------------------------------------------- | --------------------------------------------------------------------- | -------------------------------------------- |
| `/instructor/batches/:id/tasks`                    | List + create batch tasks. Shows assignment counts.                   | INSTRUCTOR (owner) or ADMIN.                 |
| `/instructor/batches/:id/tasks/:taskId`            | Edit task; `Assign…` button opens `AssignmentPicker`; `Promote to course template` button. | INSTRUCTOR (owner) or ADMIN.                 |
| `/instructor/courses/:id/tasks`                    | Read-only list of `COURSE_TEMPLATE` tasks; allows delete.             | INSTRUCTOR (owner) or ADMIN.                 |
| `/me/tasks`                                        | Student dashboard with cards: title, deadline countdown, batch label, "Open discussion". | STUDENT (assignments filtered server-side).  |
| `/tasks/:id`                                       | Read-only task view for students; description renders sanitized MD.   | Enrolled student, owning INSTRUCTOR, ADMIN.  |
| `/tasks/:id/discussion`                            | Discussion thread list + composer.                                    | Enrolled student, owning INSTRUCTOR, ADMIN.  |
| `/sessions/:id/discussion`                         | Same component, scoped to session.                                    | Same.                                        |

Components:

- `MarkdownEditor` — uses `react-markdown` preview with sanitize.
- `MarkdownView` — render-only, sanitized.
- `DeadlineCountdown` — client component that ticks once per minute; shows
  "Overdue by Nh" in red when negative.
- `AssignmentPicker` — toggle "Whole batch" or pick from enrolled students
  (typeahead).

---

## Configuration (new env vars)

None required for Phase 4. Re-uses existing storage / mailer / DB config
from prior phases.

Defaults that may be tuned (no env yet, hard-coded constants in
`packages/config/constants.ts`):

- `TASK_DESCRIPTION_MAX_BYTES = 50_000`.
- `DISCUSSION_BODY_MAX_BYTES = 10_000`.
- `DISCUSSION_REPLY_MAX_BYTES = 10_000`.
- `GATED_TASK_LEAD_TIME_MIN = 60`.

---

## Testing plan

### Unit (Vitest)

- `TaskService.create` rejects `PRE_CLASS_GATED` without `gateSessionId`,
  `passThresholdPct`, or non-null `deadlineUtc`.
- `TaskService.create` rejects deadline less than 1h before session.
- `TaskService.promoteToTemplate` produces a `COURSE_TEMPLATE` task with
  `batchId = null`, copies title/description/objectives/type/
  passThresholdPct, drops assignments, `gateSessionId`, and `deadlineUtc`.
  Sets `promotedFromTaskId` to the source task id.
- `TaskService.promoteToTemplate` is idempotent: re-promoting the same
  source task no-ops and returns the existing template (enforced by the
  unique index on `promotedFromTaskId`).
- `TaskAssignmentService.assign` is idempotent on whole-batch and on
  specific-student inputs (no duplicate rows).
- `TaskAssignmentService.assign` rejects with `TASK_DEADLINE_REQUIRED`
  if the task has `deadlineUtc = NULL`.
- `TaskTemplateInheritanceService` copies all `COURSE_TEMPLATE` tasks
  attached to the parent course into a new batch as `BATCH`-scope tasks
  with **`deadlineUtc = NULL`**. The instructor must set deadlines
  before assigning the tasks. The instructor batch editor surfaces a
  "needs review" badge on every inherited task with a NULL deadline.
- `markdown/sanitize` strips `<script>`, `onerror=`, `javascript:` URIs.

### Integration (testcontainers, real Postgres)

- Full CRUD on tasks under an owned batch; non-owner instructor gets 403.
- Whole-batch assignment then specific unassign behaviour.
- `GET /me/tasks` returns:
  - tasks where `studentUserId = me`.
  - tasks where any assignment row has `studentUserId = NULL` AND I am
    enrolled in that batch.
  - excludes tasks of any batch I am not enrolled in.
- Discussion permissions: non-enrolled student gets 403 on POST; enrolled
  student succeeds; reply permissions equal read permissions.
- Promote-to-template creates the row on the parent course; subsequent
  `BatchesService.create` for the same course copies it in.

### E2E (Playwright)

1. **instructor-creates-task** — instructor creates a REGULAR task in their
   batch with a 7-day deadline; appears in their list.
2. **instructor-assigns-to-specific-student** — instructor assigns the task
   to one of three enrolled students; only that student sees it on
   `/me/tasks`; the other two do not.
3. **student-sees-only-their-tasks** — student in batch A and batch B sees
   tasks from both, none from batches they are not enrolled in.
4. **deadline-validation-for-gated-tasks** — UI rejects deadline <1h before
   session start with the API error message; deadline ≥1h succeeds.
5. **discussion-roundtrip** — student opens task discussion, posts a
   thread, instructor replies, student sees the reply on refresh.
6. **promote-to-template-then-new-batch-inherits** — instructor promotes
   task; creates a new batch on the same course; the template appears as a
   batch task ready to assign.
7. **markdown-script-tag-blocked** — student attempts to post a reply
   containing `<script>alert(1)</script>`; the rendered HTML contains no
   script element and no alert fires.

---

## Exit criteria (checklist)

- [ ] Prisma migration applied; tables and indexes created.
- [ ] All listed endpoints return Swagger-documented schemas.
- [ ] `openapi-typescript` regen committed under `packages/shared`.
- [ ] All unit + integration tests pass; coverage on new code ≥ 70%.
- [ ] All seven E2E flows green in Playwright on `docker compose up
      --wait`.
- [ ] No new `any` introduced; `pnpm lint` and `pnpm typecheck` clean.
- [ ] Markdown sanitization covered by an explicit test that fails if
      `<script>` is allowed through.
- [ ] `BatchesService.create` (from Phase 2) updated and re-tested to
      invoke template inheritance without regressing existing tests.
- [ ] CLAUDE.md §5 still matches `prisma/schema.prisma`; no drift.
- [ ] Phase doc updated with any decisions changed during implementation.

---

## Risks & open questions

- **Resolved**: inherited template tasks land in the new batch with
  `deadlineUtc = NULL` and a "needs review" badge. The instructor must
  set the deadline before assigning the task to students. Avoids stale
  `+Nd` defaults that don't fit the new batch's schedule.
- **Resolved**: `Task.promotedFromTaskId` (FK to `Task`) makes
  promote-to-template idempotent. Re-promoting the same source returns
  the existing template (unique index enforces).
- **Discussion permissions on archived courses / completed batches.** Do
  we still allow new posts? Proposal: read-only after batch status
  `COMPLETED`. Confirm with product before P5 builds on this.
- **`mandatoryForAll` semantics.** Phase 4 stores the flag but does not
  act on it. It will gain meaning in Phase 5 (counts toward readiness)
  and Phase 6 (deadline reminder routing). Document the deferred
  behaviour in code comments to avoid future confusion.
- **Markdown editor footprint.** `react-markdown` + `rehype-sanitize` +
  highlight.js can balloon the FE bundle. Lazy-load the editor route to
  keep `/me/tasks` light.
