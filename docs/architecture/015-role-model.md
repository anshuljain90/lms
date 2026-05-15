# ADR-015: Role model — ADMIN / INSTRUCTOR / STUDENT, no orgs, Udemy-like

- Status: Accepted
- Date: 2026-05-15
- Deciders: Anshul Jain (project owner)
- Phase introduced: Phase 1 (auth + roles); enforcement extends through Phase 2+

## Context

The platform is for live-cohort teaching of GenAI to mixed-background
professionals (CLAUDE.md §1). It is run today by one instructor with
the expectation of bringing colleagues on later. Students span many
countries and (eventually) many instructors.

Key product framings:

- The catalog should feel like Udemy: anyone can browse published
  courses without logging in.
- Enrollment is approved by the owning instructor (cohort fit, not
  payment).
- Multi-tenancy/organizations are explicitly out of scope (CLAUDE.md
  §12).
- Spam from open instructor signup is a real risk; we'd rather invite
  instructors than vet them post-hoc.

## Decision

Three roles, fixed:

| Role         | Creation                                                  | Edits                       | Notes                                    |
| ------------ | --------------------------------------------------------- | --------------------------- | ---------------------------------------- |
| `ADMIN`      | Seeded from env (`ADMIN_EMAIL`/`ADMIN_PASSWORD`) on first boot, idempotent | Anything                    | Single admin in P1; multiple allowed structurally. |
| `INSTRUCTOR` | **Admin invite only** — signed link, set password, log in | Own courses/batches/etc.    | Soft ownership via `owner_instructor_id` columns. |
| `STUDENT`    | Self-register **or** instructor invite                    | Own enrollments, own posts  | Globally identified by email.            |

Visibility & ownership:

- **Public catalog**: every `Course` with `status = PUBLISHED` is
  discoverable by anyone, logged in or not.
- **Edit scope**: an INSTRUCTOR can edit only the rows they own
  (`owner_instructor_id = self.id`). ADMIN can edit any.
- **Enrollment approval**: the batch's owning instructor approves or
  rejects with optional remark; admin can override.
- **Cross-instructor students**: one student account can hold
  enrollments across many instructors. Email is the global identity.

Promotion / demotion rules:

- Admin can **promote** a STUDENT → INSTRUCTOR. Action writes to
  `AuditLog`. Student keeps their existing enrollments.
- Admin **cannot** demote an ADMIN. Removing admin rights from
  another admin requires a manual DB step, intentionally.
- Suspending a user (`User.suspended_at`) blocks all auth without
  deleting data.

ADMIN seeding:

- On boot, `apps/api` reads `ADMIN_EMAIL` and `ADMIN_PASSWORD`. If no
  user exists with that email, one is created with role ADMIN. If one
  exists with role ADMIN, password is **not** rotated automatically
  (avoid surprise lockout). If a non-admin exists with that email,
  boot fails loud.

RBAC enforcement:

- Implemented as Nest guards on every controller. The guard reads
  the JWT, attaches `RequestContext { userId, role }`, and
  the controller decorates routes with `@Roles(...)` and
  `@OwnerOnly(resourceLoader)` where ownership is required.
- Public routes are explicitly opt-in via `@Public()`; default is
  authenticated.

## Consequences

### Positive

- Three roles cover every use case in P1–P7. Zero accidental
  permissions to debug.
- Soft ownership via `owner_instructor_id` is a single column to
  filter on; SQL stays simple, no JOIN-through-org-membership.
- "Admin invite only" instructor signup eliminates spam without
  building moderation tooling.
- Public catalog drives organic discovery; matches the mental model
  students already have from Udemy/Coursera.
- ADMIN seeding from env makes fresh deploys reproducible and avoids
  a chicken-and-egg "log in to create the first admin" flow.

### Negative / Trade-offs

- No teaching assistants (TAs) yet. If a course needs co-teachers,
  we'll likely add a `CourseInstructor` join table rather than a
  fourth role. That's an additive change.
- A student who later becomes an instructor uses the same account.
  The promotion is auditable but means a single user_id wears two
  hats over time — UI needs to be aware.
- No org boundaries means data isolation is per-row, not per-tenant.
  Acceptable now; explicitly out of scope to revisit.

### Neutral

- The `Role` enum is shared via `packages/shared`.
- Audit log entries for role changes use
  `action = 'user.promote'` / `'user.suspend'` etc. for filtering.

## Alternatives considered

- **Multi-tenant from day one (orgs/workspaces)**: forces every query
  to be tenant-scoped, every UI to show a tenant switcher, every
  invite to be tenant-bound. Cost is high; benefit is zero today.
- **Single-instructor only**: would block colleagues from teaching on
  the same platform — explicitly against the medium-term plan.
- **Open instructor signup**: spam and quality risks; could be added
  later behind an email-domain allowlist if demand justifies it.
- **Per-permission ACL (CASL or similar) instead of roles**: more
  flexible, but premature when three role buckets cover everything.

## Related ADRs / docs

- `CLAUDE.md` §2 (Platform model — Roles, Visibility).
- ADR-016 (Data model) — `User.role`, `owner_instructor_id` columns,
  `EnrollmentRequest`, `AuditLog`.
- ADR-020 (Configuration) — `ADMIN_EMAIL`/`ADMIN_PASSWORD` validation.
- `docs/development/phase_1.md` (auth + ADMIN seed),
  `docs/development/phase_2.md` (course/batch ownership +
  enrollment approval).
