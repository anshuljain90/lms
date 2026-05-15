# Phase 2 — Courses, Batches, Enrollment

## Goal

Stand up the core teaching primitives. By the end of Phase 2, an
instructor can create a course with modules and topics, schedule a batch
with sessions, publish it to the public catalog, and approve students
who request enrollment. Students can browse the catalog, request a seat,
see their enrolled batches in their dashboard, and subscribe to an ICS
feed of all their session times rendered in their own timezone. This is
the right slice because it gives the platform a working teaching
backbone before adding the harder layers (LLM planner, tasks, AI
assessment).

## Scope (in)

- `Course`, `Module`, `Topic` CRUD (instructor-owned, soft ownership).
- `Batch` and `Session` CRUD scoped to a course's owner.
- Public catalog listing + course detail page.
- `EnrollmentRequest` + `Enrollment` flows with capacity enforcement.
- Approve / reject with optional remark.
- Withdraw an enrollment request (student) and remove a student from a
  batch with reason (instructor).
- `GET /me/batches` student dashboard listing.
- ICS calendar feed per student, subscribable via signed URL.
- Time zones: store UTC, render via user TZ; `Batch.default_timezone`
  is inherited by sessions unless overridden.
- Admin can perform any instructor action on any owner's resources.
- FE catalog, course detail, instructor course/batch editor, student
  dashboard with batch detail, ICS feed instructions.

## Scope (out / deferred)

- Materials (`Material` table) — Phase 3.
- LLM planning, instructor chat, token tracking — Phase 3.
- Tasks, question pools, assessment — Phases 4 and 5.
- Discussions on tasks/sessions — Phase 4.
- Notifications (in-app or email beyond enrollment confirmation) —
  Phase 6.
- Live video conferencing — out (we only store a meeting URL).
- Payments / receipts — Phase 8.
- Course duplication / template-from-existing — out.
- Search beyond simple `ILIKE` over title + description — Phase 7
  (proper search infra, e.g. Postgres FTS or Meilisearch).

## Prerequisites

- Phase 0 complete (storage, mailer, observability, config).
- Phase 1 complete (auth, role guards, `User` complete, audit log).

## Deliverables

- Migration adding `Course`, `Module`, `Topic`, `Batch`, `Session`,
  `EnrollmentRequest`, `Enrollment` (and any required join indexes).
- NestJS `CoursesModule`, `BatchesModule`, `EnrollmentsModule`,
  `CalendarModule`.
- `OwnerInstructorGuard` decorator that resolves a course / batch and
  asserts `ownerInstructorId === currentUser.id` (or admin override).
- ICS generator service producing valid RFC 5545 output (`ical-generator`
  or hand-rolled with strict tests).
- Signed-URL token for ICS subscription, exposed on `GET /me`.
- Email on enrollment-decided (approved or rejected) — sent through
  `packages/mailer`. Templates live next to the auth ones from Phase 1.
- FE: catalog, course detail, instructor course editor (modules +
  topics), batch editor (sessions), enrollment request review UI,
  student dashboard, batch detail with sessions, profile section
  showing the calendar subscription URL.
- OpenAPI updated; FE types regenerated.

## Data model deltas

Reference CLAUDE.md §5. Phase 2 adds:

```prisma
enum CourseStatus {
  DRAFT
  PUBLISHED
  ARCHIVED
}

enum BatchStatus {
  UPCOMING
  ONGOING
  COMPLETED
  CANCELLED
}

enum EnrollmentRequestStatus {
  PENDING
  APPROVED
  REJECTED
  WITHDRAWN
}

model Course {
  id                     String        @id @default(cuid())
  ownerInstructorId      String        @map("owner_instructor_id")
  slug                   String        @unique
  title                  String
  description            String        @db.Text
  learningObjectivesJson Json          @map("learning_objectives_json") // string[]
  status                 CourseStatus  @default(DRAFT)
  publishedAt            DateTime?     @map("published_at")
  createdAt              DateTime      @default(now()) @map("created_at")
  updatedAt              DateTime      @updatedAt      @map("updated_at")

  owner    User     @relation(fields: [ownerInstructorId], references: [id])
  modules  Module[]
  batches  Batch[]

  @@index([ownerInstructorId])
  @@index([status])
  @@map("courses")
}

model Module {
  id          String  @id @default(cuid())
  courseId    String  @map("course_id")
  order       Int
  title       String
  description String? @db.Text

  course Course  @relation(fields: [courseId], references: [id], onDelete: Cascade)
  topics Topic[]

  @@unique([courseId, order])
  @@index([courseId])
  @@map("modules")
}

model Topic {
  id          String  @id @default(cuid())
  moduleId    String  @map("module_id")
  order       Int
  title       String
  description String? @db.Text
  contentMd   String? @map("content_md") @db.Text

  module Module @relation(fields: [moduleId], references: [id], onDelete: Cascade)

  @@unique([moduleId, order])
  @@index([moduleId])
  @@map("topics")
}

model Batch {
  id                  String       @id @default(cuid())
  courseId            String       @map("course_id")
  ownerInstructorId   String       @map("owner_instructor_id")
  name                String
  slug                String
  startDate           DateTime     @map("start_date")     // date-only at midnight UTC
  endDate             DateTime     @map("end_date")
  defaultMeetingUrl   String?      @map("default_meeting_url")
  defaultTimezone     String       @map("default_timezone") // IANA tz, validated
  capacity            Int                                    // > 0
  status              BatchStatus  @default(UPCOMING)
  createdAt           DateTime     @default(now()) @map("created_at")
  updatedAt           DateTime     @updatedAt      @map("updated_at")

  course             Course               @relation(fields: [courseId], references: [id])
  owner              User                 @relation(fields: [ownerInstructorId], references: [id])
  sessions           Session[]
  enrollmentRequests EnrollmentRequest[]
  enrollments        Enrollment[]

  @@unique([courseId, slug])
  @@index([ownerInstructorId])
  @@index([status, startDate])
  @@map("batches")
}

model Session {
  id                  String    @id @default(cuid())
  batchId             String    @map("batch_id")
  scheduledAtUtc      DateTime  @map("scheduled_at_utc")
  durationMin         Int       @map("duration_min")
  agendaMd            String?   @map("agenda_md") @db.Text
  meetingUrlOverride  String?   @map("meeting_url_override")

  batch Batch @relation(fields: [batchId], references: [id], onDelete: Cascade)

  @@index([batchId, scheduledAtUtc])
  @@map("sessions")
}

model EnrollmentRequest {
  id               String                  @id @default(cuid())
  batchId          String                  @map("batch_id")
  studentUserId    String                  @map("student_user_id")
  message          String?
  status           EnrollmentRequestStatus @default(PENDING)
  instructorRemark String?                 @map("instructor_remark")
  decidedByUserId  String?                 @map("decided_by_user_id")
  decidedAt        DateTime?               @map("decided_at")
  createdAt        DateTime                @default(now()) @map("created_at")
  updatedAt        DateTime                @updatedAt      @map("updated_at")

  batch     Batch @relation(fields: [batchId], references: [id], onDelete: Cascade)
  student   User  @relation("StudentRequest", fields: [studentUserId], references: [id])
  decidedBy User? @relation("DecidedBy",     fields: [decidedByUserId], references: [id])

  // Only one PENDING request per (batch, student); enforced via partial index in migration SQL.
  @@index([batchId, status])
  @@index([studentUserId, status])
  @@map("enrollment_requests")
}

model Enrollment {
  id             String    @id @default(cuid())
  batchId        String    @map("batch_id")
  studentUserId  String    @map("student_user_id")
  joinedAt       DateTime  @default(now()) @map("joined_at")
  removedAt      DateTime? @map("removed_at")
  removedReason  String?   @map("removed_reason")

  batch   Batch @relation(fields: [batchId], references: [id], onDelete: Cascade)
  student User  @relation("Enrollment", fields: [studentUserId], references: [id])

  // Active enrollment uniqueness via partial index in migration SQL: WHERE removed_at IS NULL.
  @@index([batchId])
  @@index([studentUserId])
  @@map("enrollments")
}
```

Migration SQL additions (raw, after Prisma generate):

```sql
-- Only one PENDING request per (batch, student)
CREATE UNIQUE INDEX enrollment_requests_pending_uniq
  ON enrollment_requests (batch_id, student_user_id)
  WHERE status = 'PENDING';

-- Only one ACTIVE enrollment per (batch, student)
CREATE UNIQUE INDEX enrollments_active_uniq
  ON enrollments (batch_id, student_user_id)
  WHERE removed_at IS NULL;
```

User profile (Phase 1) gains a non-migrating addition: a derived
"calendar token" stored as a hash on the user (rotatable from `/me`).

```prisma
// extension on User
calendarTokenHash String? @map("calendar_token_hash")
```

## API surface

### Public

| Method | Path                          | Purpose                                       |
| ------ | ----------------------------- | --------------------------------------------- |
| GET    | `/courses`                    | Catalog: list PUBLISHED courses (+ owner's own DRAFTs), search/filter |
| GET    | `/courses/:slug`              | Course detail incl. instructor + open batches |
| GET    | `/courses/:slug/batches/:slug`| Batch detail (read-only public view)          |

Catalog query params: `q`, `instructorId`, `page`, `pageSize` (caps at
50). Sorting: newest first by default; `?sort=title|recent`.

**Catalog row visibility** (CLAUDE.md §5 "Visibility & lifecycle notes"):
`GET /courses` returns rows where `status = 'PUBLISHED'` OR (`status =
'DRAFT'` AND `owner_instructor_id = current_user_id`). DRAFT rows owned by
the caller are returned with a `draft: true` flag for the FE to render a
"DRAFT" badge. Anonymous catalog requests see PUBLISHED only.

### Authenticated student

| Method | Path                                                | Purpose                                        |
| ------ | --------------------------------------------------- | ---------------------------------------------- |
| POST   | `/batches/:id/enrollment-requests`                  | Request enrollment with optional message       |
| DELETE | `/me/enrollment-requests/:id`                       | Withdraw own pending request                   |
| GET    | `/me/enrollment-requests`                           | List own requests                              |
| GET    | `/me/batches`                                       | List active enrollments                        |
| GET    | `/me/batches/:id`                                   | Batch detail with sessions (student view)      |
| POST   | `/me/calendar-token/rotate`                         | Rotate ICS subscription token                  |
| GET    | `/calendar/:userId/:token.ics`                      | Subscribed calendar feed (no cookie auth)      |

### Authenticated instructor (owner-only unless admin)

| Method | Path                                                       | Purpose                                |
| ------ | ---------------------------------------------------------- | -------------------------------------- |
| POST   | `/courses`                                                 | Create course (becomes owner)          |
| GET    | `/courses/me`                                              | List courses I own                     |
| GET    | `/courses/:id`                                             | Course detail (full, incl. DRAFT)      |
| PATCH  | `/courses/:id`                                             | Update title, description, objectives  |
| POST   | `/courses/:id/publish`                                     | DRAFT → PUBLISHED                      |
| POST   | `/courses/:id/archive`                                     | → ARCHIVED                             |
| DELETE | `/courses/:id`                                             | Allowed only if no batches exist       |
| POST   | `/courses/:id/modules`                                     | Add module                             |
| PATCH  | `/courses/:id/modules/:moduleId`                           | Update module                          |
| DELETE | `/courses/:id/modules/:moduleId`                           | Delete module (cascade to topics)      |
| POST   | `/courses/:id/modules/:moduleId/topics`                    | Add topic                              |
| PATCH  | `/courses/:id/modules/:moduleId/topics/:topicId`           | Update topic                           |
| DELETE | `/courses/:id/modules/:moduleId/topics/:topicId`           | Delete topic                           |
| POST   | `/courses/:id/modules/reorder`                             | Reorder modules (array of IDs)         |
| POST   | `/courses/:id/modules/:moduleId/topics/reorder`            | Reorder topics                         |
| POST   | `/courses/:id/batches`                                     | Create batch                           |
| GET    | `/batches/:id`                                             | Batch detail (instructor view)         |
| PATCH  | `/batches/:id`                                             | Update batch                           |
| POST   | `/batches/:id/cancel`                                      | → CANCELLED (sessions retained; see below) |
| POST   | `/batches/:id/sessions`                                    | Add session                            |
| PATCH  | `/batches/:id/sessions/:sessionId`                         | Update session                         |
| DELETE | `/batches/:id/sessions/:sessionId`                         | Delete session                         |
| GET    | `/batches/:id/enrollment-requests`                         | List requests (filter by status)       |
| POST   | `/batches/:id/enrollment-requests/:reqId/approve`          | Approve with remark; capacity-checked  |
| POST   | `/batches/:id/enrollment-requests/:reqId/reject`           | Reject with remark                     |
| GET    | `/batches/:id/enrollments`                                 | Roster                                 |
| DELETE | `/batches/:id/enrollments/:enrollmentId`                   | Remove with required reason            |

### Auth requirements

| Group                                       | Guard                                                         |
| ------------------------------------------- | ------------------------------------------------------------- |
| `GET /courses`, `GET /courses/:slug`        | Public                                                        |
| `/calendar/:userId/:token.ics`              | Token-only (no JWT)                                           |
| Student endpoints                           | `JwtAuthGuard` + `RolesGuard('STUDENT' | 'ADMIN')`            |
| Instructor endpoints (owner-scoped)         | `JwtAuthGuard` + `RolesGuard('INSTRUCTOR' | 'ADMIN')` + `OwnerInstructorGuard` |
| Admin override                              | `RolesGuard('ADMIN')` bypasses owner check                    |

Validation rules:
- `Batch.startDate < Batch.endDate`.
- `Session.scheduledAtUtc` must fall within `[Batch.startDate, Batch.endDate + 1d]`.
- `Batch.defaultTimezone` must be in `Intl.supportedValuesOf('timeZone')`.
- Capacity strictly enforced **at approval time** with a transactional
  count of active enrollments under `SERIALIZABLE` isolation (or a
  `SELECT ... FOR UPDATE` on the batch row).
- Approving a request flips status to `APPROVED` and creates an
  `Enrollment` in the same transaction.

**Cancellation behavior** (CLAUDE.md §5 "Visibility & lifecycle notes"):
`POST /batches/:id/cancel` sets `status = CANCELLED` and soft-archives
active enrollments (`removed_at = now()`, `removed_reason = 'BATCH_CANCELLED'`)
but does **not** delete `Session` rows. Sessions remain in the ICS feed with
`STATUS:CANCELLED` (RFC 5545) until their `scheduledAtUtc` passes; after
that they are hidden from feeds and dashboards. Cancelled batches are
hidden from the public catalog immediately but remain visible (with a
"CANCELLED" badge) in `GET /me/batches` for affected students.

## UI surface

| Route                                        | Purpose                                    | Role gating                |
| -------------------------------------------- | ------------------------------------------ | -------------------------- |
| `/`                                          | Public catalog (replaces P0 placeholder)   | public                     |
| `/courses/:slug`                             | Course detail with batch list              | public                     |
| `/courses/:slug/batches/:batchSlug`          | Batch detail (read-only) + "Request to enroll" CTA | public; CTA gates to login |
| `/dashboard`                                 | Student: list of my batches; Instructor: list of my courses | authenticated              |
| `/me/calendar`                               | Profile section: ICS URL + rotate button   | authenticated              |
| `/me/batches/:id`                            | Student batch detail with sessions in their TZ | STUDENT                |
| `/me/enrollment-requests`                    | Student: pending/approved/rejected list    | STUDENT                    |
| `/instructor/courses`                        | My courses                                 | INSTRUCTOR / ADMIN         |
| `/instructor/courses/new`                    | Create course                              | INSTRUCTOR / ADMIN         |
| `/instructor/courses/:id`                    | Course editor (modules + topics)           | owner / ADMIN              |
| `/instructor/courses/:id/batches/new`        | Create batch                               | owner / ADMIN              |
| `/instructor/batches/:id`                    | Batch editor (sessions + roster + requests)| owner / ADMIN              |
| `/instructor/batches/:id/requests`           | Approve/reject UI                          | owner / ADMIN              |

Time-zone rendering: every datetime is shown in the viewer's profile
timezone with a tooltip showing UTC and (for sessions) the batch's
default timezone.

## Configuration

New env vars on top of Phase 1:

| Var                            | Required | Default                | Notes                                        |
| ------------------------------ | -------- | ---------------------- | -------------------------------------------- |
| `CATALOG_PAGE_SIZE_DEFAULT`    | yes      | `20`                   |                                              |
| `CATALOG_PAGE_SIZE_MAX`        | yes      | `50`                   |                                              |
| `CALENDAR_TOKEN_BYTES`         | yes      | `32`                   | Random byte length before base64url encoding |
| `BATCH_DEFAULT_CAPACITY`       | yes      | `30`                   | Used in the batch-create form pre-fill        |

No new external services in Phase 2. Storage is unused; mailer is reused
for enrollment-decision notifications.

## Testing plan

### Unit tests

- ICS generator produces valid output (vendored test corpus + an
  `ical.js` round-trip parse).
- TZ helpers: instructor enters a wall-clock time + tz; backend stores
  UTC; FE reformats to viewer TZ.
- Capacity counter: counts only `removed_at IS NULL` enrollments.
- Slug generator: stable, lowercased, length-bounded, collision suffix.
- Reorder service: validates that submitted IDs are exactly the current
  set under that parent.

### Integration tests (testcontainers)

- Concurrent approval race: two admin/instructor approvals at exactly
  the same time on the last seat → exactly one becomes `APPROVED`,
  the other returns 409 `BATCH_FULL`.
- Withdrawing a `PENDING` request frees the partial-unique constraint.
- Rejecting and re-applying: a previously-rejected student can submit
  a new request.
- Removing a student then re-approving: a new active enrollment row is
  created; the partial-unique index allows it because the prior is
  soft-deleted.
- Cascade: deleting a module deletes its topics; deleting a batch
  deletes its sessions and requests but rejects if active enrollments
  exist (require `--force` flag explicit in the API).
- ICS feed: a student with sessions across two batches sees both, with
  TZ-correct DTSTART/DTEND and `UID` stable across regenerations.

### E2E (Playwright)

Named golden flows that must pass:

1. `instructor-creates-course-with-batch`: instructor logs in, creates a
   DRAFT course, adds a module + topic, creates a batch with three
   sessions, publishes the course.
2. `student-browses-and-requests-enrollment`: anonymous user opens
   catalog, opens a course, clicks request → redirected to login →
   logs in → submits request with message.
3. `instructor-approves-with-remark`: instructor sees the pending
   request and approves with a remark; student receives the email
   (Mailpit) and the request appears as APPROVED.
4. `student-sees-batch-in-dashboard`: after approval, the batch shows
   on the student dashboard with sessions in their TZ.
5. `ics-feed-renders`: student copies their calendar URL, fetches it,
   parses with `ical.js` — events match.
6. `capacity-enforced`: instructor sets capacity to 1; two students
   request; first approval succeeds, second returns a friendly
   "BATCH_FULL" message in the UI.

## Exit criteria

- [ ] All migrations applied and idempotent.
- [ ] All Playwright flows above pass.
- [ ] Concurrent approval test green under load (10 parallel attempts).
- [ ] ICS feed validates against `ical.js` round-trip in CI.
- [ ] All instructor endpoints return 403 (not 404) when called by a
      non-owner instructor; admin succeeds in those same calls.
- [ ] Catalog endpoints respond in <200ms p95 on a seeded dataset of
      200 courses (local docker compose).
- [ ] FE renders all session datetimes in the viewer's profile TZ; no
      raw UTC strings shown without explicit "UTC" badge / tooltip.
- [ ] OpenAPI regenerated; FE types regenerated; CI re-run shows no
      drift.

## Risks & open questions

- **Concurrent approvals** are the trickiest correctness surface. The
  `SERIALIZABLE` transaction approach is simple but throws conflict
  errors that must be retried. Confirm the retry policy and surface a
  user-friendly error rather than a raw 500.
- **ICS auth** uses an unguessable token in the URL because most
  calendar clients don't support headers. That makes the URL itself
  the secret; document "rotate if shared" prominently in `/me/calendar`.
- **Catalog search** with `ILIKE` is fine to ~10k courses; Phase 7 will
  swap in proper FTS. Make sure search code lives behind a small
  service interface so the swap is contained.
- **Module/topic ordering**: chosen integer `order` field with unique
  index. Reordering rewrites the field in a single transaction.
  Alternative (fractional indexing) deferred to avoid premature
  optimization.
- **Resolved**: DRAFT courses are visible to their owner in the catalog
  with a "DRAFT" badge; nobody else sees them. Anonymous and other
  authenticated users see PUBLISHED only. Implementation: add
  `(status = 'PUBLISHED' OR owner_instructor_id = current_user_id)`
  predicate to the catalog query.
- **Resolved**: cancelled batches retain `Session` rows; ICS feed emits
  `STATUS:CANCELLED` until the original `scheduledAtUtc` passes; after
  that the session is filtered out of feeds and dashboards. Enrollments
  are soft-archived with `removed_reason = 'BATCH_CANCELLED'`.
