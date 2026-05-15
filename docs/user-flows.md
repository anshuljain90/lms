# User Flows

> Linear textual walk-throughs of the platform's primary user journeys.
> Each flow notes the phase that delivers it (see `/docs/development/phase_*.md`).
> Cross-references: `CLAUDE.md` §2 (roles), §5 (data model), §6 (LLM strategy).

---

## 1. Admin onboards a new instructor (P1)

`Admin logs in` → `Admin Dashboard` → `Users / Invite Instructor` → `Enters email + name + monthly_token_budget` → `System sends signed invite link via Mailpit/Gmail` → `Instructor receives email` → `Clicks link` → `Set Password page` → `Submits` → `Email auto-verified` → `Auto-logged-in` → `Lands on Instructor Dashboard (empty courses list)`

---

## 2. Student self-registers (P1)

`Student visits /` → `Sign Up` → `Enters email + password + name` → `Account created (status=UNVERIFIED)` → `Verification email sent` → `Student clicks link` → `Email verified` → `Profile setup (timezone, bio, avatar)` → `Lands on public Catalog`

---

## 3. Student forgets and resets password (P1)

`Login page` → `Forgot password?` → `Enters email` → `Reset link mailed (signed token)` → `Clicks link` → `New Password page` → `Submits` → `password_reset_nonce bumped (link single-use)` → `Auto-logged-in` → `Dashboard`

---

## 4. Instructor creates a course (P2)

`Instructor Dashboard` → `+ New Course` → `Fills title / description / learning objectives` → `Adds modules (ordered)` → `Adds topics under each module (with content_md)` → `Saves as DRAFT` → `Previews course in catalog (DRAFT badge visible only to owner)` → `Clicks Publish` → `status=PUBLISHED` → `Course visible to all in /courses`

---

## 5. Instructor creates a batch with sessions (P2)

`Course detail page (own)` → `+ New Batch` → `Enters name / start_date / end_date / capacity / default_meeting_url / default_timezone` → `Saves Batch (status=UPCOMING)` → `Adds Session 1 (scheduled_at, agenda, optional URL override)` → `... Session N` → `Batch appears on the course's public detail page` → `Inherited COURSE_TEMPLATE tasks copied in with NULL deadlines` → `Instructor sets deadlines on each` → `Batch ready for enrollment`

---

## 6. Student discovers and requests enrollment (P2)

`Anonymous user → /` → `Searches catalog ("Generative AI")` → `Opens Course detail` → `Picks a Batch (sees schedule in their TZ)` → `Clicks Request to Enroll` → `Redirected to login (if not logged in)` → `Logs in` → `Returns to batch with optional message field` → `Submits` → `EnrollmentRequest row created (status=PENDING)` → `Instructor receives ENROLLMENT_REQUESTED notification (email + in-app)`

---

## 7. Instructor approves enrollment request (P2)

`Instructor Dashboard` → `Batch → Enrollment Requests` → `Sees pending request with student message` → `Clicks Approve, adds remark` → `Transactional check: capacity available?` → `EnrollmentRequest=APPROVED + Enrollment row created` → `Student receives ENROLLMENT_APPROVED email + in-app notification` → `Student appears in batch roster` → `Student's /me/batches now lists this batch`

---

## 8. Instructor plans content with Planner Agent and saves a snippet (P3)

`Instructor → /instructor/planner` → `+ New Chat (scope: Course X)` → `Asks: "Draft an outline on Prompt Engineering for QA professionals"` → `SSE stream: PlannerAgent calls generateOutline tool → tool result rendered → assistant response streams in tokens` → `LlmUsage row written (feature=PLANNING)` → `Instructor highlights the outline` → `Save to Material → picks scope (Course X)` → `Material created (source=LLM_CHAT_EXPORT, attribution_json with full prompt + model + trace_id)` → `Auto-visible to every batch under Course X`

---

## 9. Instructor uploads a file material (P3)

`Instructor → Course → Materials` → `+ Upload` → `Picks PDF (under 50MB, allowed MIME)` → `FE requests pre-signed URL from API (storage driver: minio/gcs) OR posts multipart (local FS driver)` → `File uploaded` → `Material row created (source=MANUAL_UPLOAD, file_ref=storage_key)` → `Appears in course material list` → `All batches under that course show it in their materials`

---

## 10. Instructor creates a task and assigns it (P4)

`Instructor → Batch → Tasks` → `+ New Task` → `Fills title / description_md / learning_objectives / type (REGULAR or PRE_CLASS_GATED) / deadline_utc` → `If PRE_CLASS_GATED: picks gate_session, sets pass_threshold_pct (deadline auto-validated as ≥1h before session)` → `Saves Task` → `Clicks Assign → Whole Batch (or picks specific students)` → `TaskAssignment rows created` → `Each assigned student receives TASK_ASSIGNED notification` → `Task appears on /me/tasks for those students with countdown`

---

## 11. Instructor promotes a task to course template (P4)

`Instructor → Batch → Tasks → Task X` → `Clicks Promote to Course Template` → `System creates COURSE_TEMPLATE Task Y on parent course (promoted_from_task_id=X, deadline=NULL, no assignments)` → `Re-clicking promote no-ops (unique index returns existing template)` → `Next time instructor creates a Batch under same Course → TaskTemplateInheritanceService copies Task Y in as a BATCH-scope task with NULL deadline + "needs review" badge` → `Instructor sets deadline → assigns`

---

## 12. Instructor generates and publishes a question pool (P5)

`Instructor → Task X → Question Pool` → `Clicks Generate (size=20)` → `PlannerAgent.proposeQuestionPool tool runs (LlmUsage feature=QUESTION_GEN)` → `20 DRAFT questions appear (mix MCQ + TEXT, stratified by difficulty)` → `Instructor edits Q3's wording (instructor_edited=true)` → `Regenerates Q5 (single-question reroll)` → `Sets rubric_json on each TEXT question (criteria + weights + max points)` → `Clicks Publish` → `Validation: every question has non-empty rubric` → `Pool status=PUBLISHED, immutable` → `Students can now attempt`

---

## 13. Student takes AI assessment (P5)

`Student → /me/tasks → Task X → Take Assessment` → `POST /tasks/:id/attempts → random stratified subset selected (e.g., 2 EASY+2 MED+1 HARD), persisted in selected_question_ids` → `SSE chat opens` → `AssessorAgent presents Q1 (MCQ, click-only buttons rendered, text input hidden)` → `Student clicks option → POST answer → AssessorAgent.scoreAnswer (exact-match)` → `Q2 (TEXT, free input)` → `Student answers vaguely → AssessorAgent.askFollowUp (1 of 2 max) → student clarifies → AssessorAgent.scoreAnswer (per-rubric criteria + justification)` → `... continues through subset` → `AssessorAgent.finalizeAttempt → final_score = weighted sum` → `Status=GRADED` → `Student sees score breakdown` → `LlmUsage rows written (feature=ASSESSMENT)`

---

## 14. Instructor reviews transcript and overrides score (P5)

`Instructor → Task X → Attempts` → `Sees per-student rows with AI score` → `Clicks attempt` → `Full transcript rendered (every AttemptTurn with role + content + tool_calls + tokens)` → `AI score breakdown per criterion shown` → `Instructor disagrees → fills override score (0–1) + reason` → `instructor_override_score saved (final_score preserved); AuditLog row written` → `Student receives ASSESSMENT_OVERRIDE notification (Phase 6)` → `Student's result page shows BOTH scores with timestamps`

---

## 15. Pre-class readiness flag and soft gate (P5)

`Scheduled session is in <24h` → `Instructor opens batch roster` → `GET /batches/:id/roster-with-readiness runs ReadinessService` → `For each gated task whose gate_session_id is the upcoming session: per student, check best GRADED attempt vs Task.pass_threshold_pct` → `Roster shows Ready / Not Ready badges per student` → `Instructor messages "not ready" students or proceeds` → `Student NOT auto-blocked from joining the session (soft gate per CLAUDE.md §2)`

---

## 16. Student receives deadline reminders (P6)

`Scheduler tick (every 5 min via @Cron)` → `Finds tasks where deadline_utc is between now+23h55m and now+24h05m, joined with assignments` → `Per (user, task): upsert Notification with idempotency_key=task:X:user:Y:24h` → `If new row inserted: dispatch email via IMailer + write in-app row` → `Student receives email + bell badge increments` → `Click → opens task → submits/completes` → `Same flow at T-1h with idempotency_key=...:1h`

---

## 17. Discussion roundtrip on a task (P4 + P6)

`Student opens Task X → Discussion tab` → `Posts thread (markdown body, sanitized)` → `Discussion row created` → `Instructor + admin + every batch enrollee can read` → `Instructor replies` → `DiscussionReply row created` → `Original author receives DISCUSSION_REPLY notification (in-app + email)` → `Reload shows full thread`

---

## 18. Calendar subscription (P2)

`Student → /me/calendar` → `Copies signed URL: /calendar/:userId/:token.ics` → `Pastes into Google/Apple Calendar Subscribe` → `Calendar app polls every few hours` → `Sees all sessions across all enrolled batches in their TZ (with VTIMEZONE blocks)` → `Instructor cancels Session Z` → `On next poll: Session Z appears as STATUS:CANCELLED` → `Calendar app greys it out` → `After scheduled_at passes: filtered out`

---

## 19. Admin monitors token consumption (P3)

`Admin → /admin/usage` → `Global cost-USD chart for current month` → `Per-instructor breakdown table` → `Per-feature stacked bar (PLANNING / QUESTION_GEN / ASSESSMENT / CHAT)` → `Spots Instructor Y at 95% of budget` → `Drills into their LlmUsage rows` → `Either raises budget or alerts instructor` → `Instructor receives BUDGET_WARNING notification at 80% (P6)`

---

## 20. Token budget exceeded mid-flow (P3)

`Instructor in Planner chat → submits message` → `BudgetGuard middleware reads InstructorProfile.monthly_token_budget` → `Sums LlmUsage for current month + estimate for this call` → `Sum > cap` → `Returns 429 with { code: BUDGET_EXCEEDED, bucket: PLANNING, used, cap, periodEndsAt }` → `FE shows friendly "Monthly budget reached, contact admin" panel` → `Audit log entry written` → `Instructor checks /me/usage dashboard for breakdown`

---

## Index by phase

| Phase | Flows |
| ----- | ----- |
| P1 — Auth & Users | 1, 2, 3 |
| P2 — Courses, Batches, Enrollment | 4, 5, 6, 7, 18 |
| P3 — Materials & Planner LLM | 8, 9, 19, 20 |
| P4 — Tasks & Discussions | 10, 11, (17 partly) |
| P5 — AI Assessment | 12, 13, 14, 15 |
| P6 — Notifications & PWA | 16, 17 |

## Index by role

| Role | Flows |
| ---- | ----- |
| ADMIN | 1, 19 |
| INSTRUCTOR | 1 (recipient), 4, 5, 7, 8, 9, 10, 11, 12, 14, 15, 17 (replies), 20 |
| STUDENT | 2, 3, 6, 13, 16, 17, 18 |
