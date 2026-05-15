# Phase 6 — Notifications & PWA polish

## Goal

Close the user-facing feedback loop. Until now, students and instructors
must keep checking the UI to know that something happened — a task was
assigned, a deadline is approaching, an enrollment was decided, or an
assessment score was overridden. This phase adds a single `Notification`
table, a small set of well-defined notification kinds, server-rendered
email templates over the existing `IMailer` abstraction, and a scheduler
for time-based reminders. It also turns the Next.js app into a proper
installable PWA with manifest, icons, service worker, and an offline
shell. Web Push is intentionally deferred to Phase 7 because it requires
HTTPS provisioning that lands with Caddy in P7.

This is the right slice now because Phases 4 and 5 produce all the events
that need notifying about (task assignments, deadlines, override scores,
discussion replies). Doing it earlier would have meant notifying about
nothing.

---

## Scope (in)

- `Notification` table (CLAUDE.md §5).
- Notification kinds:
  - `TASK_ASSIGNED` — to student.
  - `TASK_DEADLINE_24H` — to student.
  - `TASK_DEADLINE_1H` — to student.
  - `ENROLLMENT_REQUESTED` — to instructor.
  - `ENROLLMENT_APPROVED` — to student.
  - `ENROLLMENT_REJECTED` — to student.
  - `SESSION_REMINDER_24H` — to enrolled student.
  - `ASSESSMENT_OVERRIDE` — to student.
  - `DISCUSSION_REPLY` — to thread author.
- Email templates rendered server-side using `react-email` (chosen for
  TS-native ergonomics; `mjml` rejected to keep one templating runtime).
- Templates dispatched via the existing `IMailer` abstraction
  (`packages/mailer`); Mailpit captures them in dev (CLAUDE.md §3).
- Scheduler: NestJS `@nestjs/schedule` (`@Cron`) for time-based jobs in
  this phase. Decision recorded: `@Cron` is sufficient for current
  cadence; BullMQ is reconsidered in P7 if/when we need retries with
  durability or horizontal workers.
- In-app notification list + unread-count badge in the global nav.
- PWA: `manifest.json`, app icons (multiple resolutions), `next-pwa`
  service worker with offline shell rendering for the last-visited page.
- Avatar upload polish (crop, accepted MIME types tightened, sane size
  limit) building on Phase 1 storage.
- Profile completeness indicator on the user profile page.
- Add Redis to `infra/docker-compose.yml` only **if** we elect BullMQ in
  this phase. Default decision: **no Redis yet** — `@Cron` runs in-process.

## Scope (out / deferred)

- Web Push notifications (deferred to P7; requires HTTPS via Caddy).
- SMS / WhatsApp / Slack channels.
- Notification preferences UI (granular per-kind opt-out). Phase 6 ships
  global on/off only; per-kind preferences may land alongside Web Push
  in P7.
- Capacitor-wrapped native app (Phase 8).
- Background sync of write actions while offline (offline shell renders
  read views only).
- Real-time push of new in-app notifications (poll every 30s for now).

## Prerequisites

- Phase 0 (foundation, docker-compose with Mailpit).
- Phase 1 (auth, user profile, avatar storage).
- Phase 2 (enrollment events).
- Phase 4 (Task, Discussion entities).
- Phase 5 (Assessment override events).
- `IMailer` and `IStorage` abstractions live (per CLAUDE.md §3).

## Deliverables (concrete artifacts)

- Prisma migration adding `Notification` table.
- `NotificationsModule` (NestJS) with:
  - `NotificationsService` (create, list-for-user, mark-read,
    mark-all-read).
  - `NotificationDispatcher` — single point that writes the row, decides
    whether to also email (per kind), and renders + sends the template.
  - Per-kind handlers registered via a small registry pattern; each
    handler has a `kind`, an email template component, and an in-app
    payload Zod schema.
- `SchedulerModule` with `@Cron` jobs:
  - `task-deadline-reminders` (every 5 min) — emits 24h and 1h
    reminders, idempotent via a `Notification` upsert keyed on
    `(userId, kind, taskId)`.
  - `session-reminders` (every 5 min) — emits 24h reminders.
- Event hooks in existing services:
  - `TaskAssignmentService.assign` → `TASK_ASSIGNED`.
  - `EnrollmentService.request` → `ENROLLMENT_REQUESTED`.
  - `EnrollmentService.decide` → `ENROLLMENT_APPROVED` or
    `ENROLLMENT_REJECTED`.
  - `AssessmentAttemptService.overrideScore` → `ASSESSMENT_OVERRIDE`.
  - `DiscussionService.replyCreated` → `DISCUSSION_REPLY` to the parent
    discussion's `authorUserId` (skip if reply author is the same user).
- React-email templates under `packages/mailer/templates/`:
  - `task-assigned.tsx`, `task-deadline-24h.tsx`, `task-deadline-1h.tsx`,
    `enrollment-requested.tsx`, `enrollment-approved.tsx`,
    `enrollment-rejected.tsx`, `session-reminder-24h.tsx`,
    `assessment-override.tsx`, `discussion-reply.tsx`.
  - Each exports a `subject(payload)` + a default React component.
  - Plain-text fallback auto-derived via `react-email/render` text mode.
- FE additions:
  - `NotificationBell` component in nav with unread count.
  - `/me/notifications` route showing chronological list with
    mark-as-read.
  - `manifest.json`, icon set (192, 256, 384, 512 PNG + maskable
    variants).
  - `next-pwa` configured with a runtime caching strategy: documents
    network-first with cache fallback; static assets cache-first.
  - Offline shell page (`app/offline/page.tsx`) rendered by service
    worker on navigation when network is unavailable.
- Avatar uploader v2: tightened to `image/png|jpeg|webp`; max 2 MB; FE
  crop step (square) before upload; existing `IStorage` driver reused.
- Profile completeness indicator (computes a percentage from filled
  fields; displays under user header).
- Scheduler choice (`@Cron` in P6, BullMQ revisit in P7) is recorded
  inline in this phase doc rather than as a standalone ADR. If a
  separate ADR is wanted later, it is the next free slot (ADR-021).

---

## Data model deltas

Reference: CLAUDE.md §5.

```prisma
enum NotificationKind {
  TASK_ASSIGNED
  TASK_DEADLINE_24H
  TASK_DEADLINE_1H
  ENROLLMENT_REQUESTED
  ENROLLMENT_APPROVED
  ENROLLMENT_REJECTED
  SESSION_REMINDER_24H
  ASSESSMENT_OVERRIDE
  DISCUSSION_REPLY
}

model Notification {
  id          String           @id @default(cuid())
  userId      String
  user        User             @relation(fields: [userId], references: [id], onDelete: Cascade)
  kind        NotificationKind
  payloadJson Json             // Per-kind shape; Zod-validated at dispatch.
  readAt      DateTime?
  createdAt   DateTime         @default(now())

  // Idempotency for scheduler-driven kinds: prevents duplicate 24h/1h reminders.
  idempotencyKey String?       @unique

  @@index([userId, readAt])
  @@index([userId, createdAt])
}
```

Per-kind `payloadJson` shapes (Zod schemas in
`packages/shared/notifications/`):

- `TASK_ASSIGNED`: `{ taskId, taskTitle, batchId, batchName, deadlineUtc }`.
- `TASK_DEADLINE_24H` / `TASK_DEADLINE_1H`: `{ taskId, taskTitle,
  deadlineUtc }`.
- `ENROLLMENT_REQUESTED`: `{ enrollmentRequestId, batchId, batchName,
  studentName }`.
- `ENROLLMENT_APPROVED` / `ENROLLMENT_REJECTED`: `{ batchId, batchName,
  remark? }`.
- `SESSION_REMINDER_24H`: `{ sessionId, batchId, batchName,
  scheduledAtUtc, meetingUrl }`.
- `ASSESSMENT_OVERRIDE`: `{ attemptId, taskId, taskTitle, oldScore,
  newScore, reason }`.
- `DISCUSSION_REPLY`: `{ discussionId, replyId, parentScope, parentId,
  replyAuthorName }`.

Idempotency keys (reminders only):

- `TASK_DEADLINE_24H` → `task:${taskId}:user:${userId}:24h`.
- `TASK_DEADLINE_1H` → `task:${taskId}:user:${userId}:1h`.
- `SESSION_REMINDER_24H` → `session:${sessionId}:user:${userId}:24h`.

Event-driven kinds (`TASK_ASSIGNED`, `ENROLLMENT_*`,
`ASSESSMENT_OVERRIDE`, `DISCUSSION_REPLY`) leave `idempotencyKey` null.

---

## API surface

| Method | Path                                      | Purpose                                                          | Auth gate            |
| ------ | ----------------------------------------- | ---------------------------------------------------------------- | -------------------- |
| GET    | `/me/notifications`                       | Paginated list of own notifications. Query: `unreadOnly?=true`.  | Authenticated user.  |
| GET    | `/me/notifications/unread-count`          | Cheap count for the bell badge.                                  | Authenticated user.  |
| POST   | `/me/notifications/:id/read`              | Mark one notification read.                                      | Owner only.          |
| POST   | `/me/notifications/read-all`              | Mark all unread as read.                                         | Authenticated user.  |
| GET    | `/me/profile-completeness`                | Returns `{ percent, missing: string[] }`.                        | Authenticated user.  |

No new admin or instructor endpoints in this phase. Email is sent
side-effect-style by the dispatcher; no API to "trigger send manually" in
P6 (deferred to test/admin tooling later).

Errors:

- `NOTIFICATION_NOT_FOUND` (404).
- `NOTIFICATION_FORBIDDEN` (403) when marking another user's row.

---

## UI surface

| Route / Component             | Purpose                                                                               | Role gating               |
| ----------------------------- | ------------------------------------------------------------------------------------- | ------------------------- |
| `NotificationBell` (in nav)   | Shows unread count; click opens dropdown of last 10 with mark-read affordance.        | Authenticated user.       |
| `/me/notifications`           | Full chronological list, filter (all / unread), bulk mark-all-read button.            | Authenticated user.       |
| `/me/profile`                 | Existing route; adds completeness indicator + avatar crop dialog.                     | Authenticated user.       |
| `/offline`                    | Static fallback page rendered by service worker when offline + no cache hit.          | Public.                   |
| `manifest.json`               | PWA manifest (name, short_name, icons, theme_color, background_color, display).       | Public asset.             |

PWA behavior:

- Add-to-home-screen prompt fires after the user has authenticated and
  visited at least 3 distinct routes (suppressed otherwise to avoid
  nagging).
- Service worker precaches the app shell (HTML, CSS, font subset, icons).
- Runtime cache for API GETs of `/me/notifications` and `/me/tasks`
  uses `stale-while-revalidate` with a 30s TTL so the app feels alive
  but does not show wildly stale state.
- All write requests are network-only — no offline write queue in this
  phase.

---

## Configuration (new env vars)

Add to `.env.example`:

- `MAILER_FROM_NAME` (default `LMS`).
- `MAILER_FROM_EMAIL` (already present from Phase 1; reused).
- `APP_PUBLIC_URL` (e.g. `http://localhost:3000`) — used to build
  deep links in email templates.
- `SCHEDULER_ENABLED` (default `true`) — turn off in tests / read replicas.
- `SCHEDULER_TICK_MINUTES` (default `5`) — frequency of reminder cron.
- `PWA_ENABLED` (default `true`) — `next-pwa` is disabled when `false`
  for easier dev hot-reload.
- `AVATAR_MAX_BYTES` (default `2_097_152`).

Validate at boot via `packages/config` (CLAUDE.md §7). No new
infrastructure containers added unless we escalate to BullMQ (defer).

---

## Testing plan

### Unit (Vitest)

- Each per-kind dispatcher: payload Zod validation rejects malformed
  shapes; idempotency keys are constructed correctly.
- React-email templates render to non-empty HTML for representative
  payloads (snapshot tests).
- `NotificationsService.markRead` is a no-op (returns 200) when the row
  is already read; preserves original `readAt`.
- Profile completeness percentage is computed from a known fixture and
  matches expected (e.g. 60% when 3 of 5 weighted fields filled).

### Integration (testcontainers + Mailpit)

- `TaskAssignmentService.assign` to a single student triggers exactly one
  `Notification` row and one Mailpit email.
- Whole-batch assignment to a 5-student batch emits 5 notifications and 5
  emails.
- Scheduler tick: with two tasks at deadline -24h05m and -23h55m, only
  the latter generates a `TASK_DEADLINE_24H`. Run the tick again 6 minutes
  later: still no duplicate (idempotency holds).
- 1h reminder fires correctly even if the 24h reminder was missed (e.g.
  task created 30m before deadline produces only the 1h reminder).
- `ENROLLMENT_REQUESTED` notification is delivered to the **owning
  instructor** of the batch, not all instructors.
- `DISCUSSION_REPLY` is suppressed when the reply author equals the
  parent author.
- `ASSESSMENT_OVERRIDE` carries old + new scores in payload.

### E2E (Playwright)

1. **assigned-task-notifies-student** — instructor assigns a task; within
   5s the student's notification bell shows unread badge `1` and
   `/me/notifications` lists the kind.
2. **deadline-reminder-fires-1h-before** — Playwright fast-forwards
   "now" via a test-only endpoint or seed script to put a task ~58
   minutes from deadline; tick the scheduler; assert 1h notification
   appears and Mailpit received the email.
3. **install-as-pwa-on-android** — Chrome Android emulation: run
   Lighthouse PWA audit programmatically; assert installability score
   passes; also verify `manifest.json` is reachable and icon set
   resolves.
4. **in-app-mark-as-read** — student clicks a notification; row's
   `readAt` populates; bell count decrements; reload preserves state.
5. **enrollment-decision-roundtrip** — student requests enrollment →
   instructor sees in-app notification → instructor approves → student
   gets approval notification + email.
6. **assessment-override-notifies-student** — instructor overrides a
   student's score; student receives `ASSESSMENT_OVERRIDE` with old/new
   score; clicking opens the result page.
7. **offline-shell-renders** — DevTools offline mode after first load;
   navigation to a previously visited route shows cached content; a
   never-visited route shows `/offline`.

---

## Exit criteria (checklist)

- [ ] `Notification` table migrated; idempotency unique index in place.
- [ ] All nine `NotificationKind` values dispatched at least once across
      tests.
- [ ] All seven E2E flows green.
- [ ] Email templates render correctly in Mailpit (visual smoke during
      review; snapshots in unit tests).
- [ ] PWA Lighthouse score: Installable = pass; PWA Optimized = pass.
- [ ] `next-pwa` does not break dev hot reload (`PWA_ENABLED=false` in
      dev `.env.example`).
- [ ] Profile completeness shown on `/me/profile`.
- [ ] Avatar upload limited to declared MIME types and size; rejection
      shows a helpful error.
- [ ] No new container in `infra/docker-compose.yml` unless we adopted
      BullMQ (current decision: no).
- [ ] CLAUDE.md §5 still matches `prisma/schema.prisma`.

---

## Risks & open questions

- **`@Cron` durability.** A scheduler tick missed during downtime is
  silently lost. The 5-minute frequency limits the blast radius (most
  reminders still fire on the next tick), but a long outage straddling
  a 1h deadline could miss the `TASK_DEADLINE_1H`. Acceptable for
  current cohort sizes; revisit in P7 if BullMQ is adopted.
- **Email deliverability.** With `gmail-smtp` driver, daily caps and
  spam reputation will bite once cohort sizes grow. The `IMailer`
  abstraction lets us swap to a provider API in P7 without code
  changes; document the migration in operational runbook.
- **Notification fatigue.** Whole-batch assignments × 30 students × 9
  kinds gets noisy. Consider digesting reminder kinds (e.g. one daily
  digest of all 24h-out tasks) — propose for P7 if user feedback
  demands it.
- **PWA caching of stale auth.** Service worker caching of API
  responses can leak between users on shared devices. Strategy:
  document that the SW only caches static assets + non-PII GETs; do
  not cache `/me/*` responses across logins (clear cache on logout).
- **Scheduler in multi-instance deploy.** `@Cron` runs per process; if
  we ever scale the API horizontally, two schedulers will fire the
  same job twice. The `Notification.idempotencyKey` unique constraint
  catches duplicates at write time, but the wasted work is real.
  **Resolved approach (deferred to Phase 7)**: wrap each tick in a
  Postgres advisory lock (`pg_try_advisory_lock(<hash(jobName)>)`); the
  process that fails to acquire the lock skips that tick. No new
  infrastructure required.
- **Push permission UX.** Phase 6 stops short of asking for
  notification permission. P7's Web Push implementation should treat
  the prompt as opt-in and gate it behind an explicit user action
  (settings toggle) rather than auto-prompting on app load.
