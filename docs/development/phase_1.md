# Phase 1 — Auth & Users

## Goal

Make the platform identity-aware. By the end of Phase 1, real humans can
sign up, verify their email, log in with rotating JWTs, manage their own
profile, and admins can invite instructors via signed links. This is the
smallest credible slice that unblocks every later phase: every Phase 2+
endpoint will hang off the auth and role primitives shipped here. Auth is
intentionally email + password + JWT (CLAUDE.md §3); Keycloak / OAuth is
deferred to Phase 8 behind the same `/auth` API surface.

## Scope (in)

- `User` table fully fleshed out (verification, suspension, profile fields).
- `EmailVerificationToken`, `PasswordResetToken`, `RefreshToken`,
  `InstructorInvite`, and `AuditLog` tables.
- Email-and-password registration for **students only** (self-signup).
- Instructor accounts created **only** via admin-issued signed invite.
- Email verification (signup) and password reset flows.
- JWT auth: 15-minute access token via `Authorization` header; 30-day
  rotating refresh token in an HTTP-only, Secure, SameSite=Strict cookie.
- `/me` profile read + edit (name, timezone, bio, avatar).
- Avatar upload via `packages/storage` from Phase 0.
- Admin user management: list, suspend/unsuspend, change role with audit
  trail, resend verification email, invite instructor.
- Rate limiting on every `/auth/*` endpoint via `@nestjs/throttler`.
- Bcrypt for passwords, cost 12.
- Idempotent ADMIN seed from env on first boot.
- Audit log writes for security-relevant actions.
- FE auth screens: sign up, sign in, verify email, request password
  reset, reset password, profile, accept-instructor-invite.
- FE admin screens: user list, user detail (suspend / role change),
  invite instructor.

## Scope (out / deferred)

- OAuth / SSO providers, Google login — Phase 8 (Keycloak).
- Argon2id password hashing — future option, not in P1.
- Two-factor authentication — not currently planned; revisit in Phase 7.
- Magic-link / passwordless login — out.
- Course / batch / enrollment surfaces — Phase 2.
- LLM features and per-user token budgets enforcement — Phase 3 (the
  `monthly_token_budget` field is set on the invite here but only
  consumed by Phase 3 middleware).
- Notifications (email beyond auth flows; in-app feed) — Phase 6.
- PWA install / offline behaviour — Phase 6.

## Prerequisites

- Phase 0 complete: monorepo, Prisma migrations, `packages/config`,
  `packages/storage`, `packages/mailer`, `packages/observability` all
  green.

## Deliverables

- Migration creating five tables and extending `User`.
- NestJS `AuthModule`, `UsersModule`, `AdminModule` with controllers,
  services, guards, strategies.
- `JwtAuthGuard`, `RolesGuard` (`@Roles('ADMIN' | 'INSTRUCTOR' | 'STUDENT')`),
  `@CurrentUser()` decorator.
- Refresh-token rotation with reuse detection (revokes the entire family
  on detected reuse).
- Email templates (MJML or plain HTML) for: verification, password reset,
  instructor invite, role-changed notice, account-suspended notice.
- Seed script `pnpm db:seed` that idempotently creates the ADMIN user.
- FE auth pages and `useSession` hook backed by a `/me` fetch.
- `apps/web` middleware redirecting unauthenticated users from
  `/dashboard/*` and `/admin/*`.
- Playwright E2E suites listed below.
- OpenAPI doc updated; `openapi-typescript` regenerated FE types.

## Data model deltas

Reference CLAUDE.md §5 for the broader model. Phase 1 lands these:

```prisma
// Extends Phase 0 User
model User {
  id               String    @id @default(cuid())
  email            String    @unique
  passwordHash     String    @map("password_hash")
  role             UserRole
  name             String
  avatarUrl        String?   @map("avatar_url")
  timezone         String    @default("UTC")
  bio              String?
  emailVerifiedAt  DateTime? @map("email_verified_at")
  suspendedAt      DateTime? @map("suspended_at")
  suspendedReason  String?   @map("suspended_reason")
  createdAt        DateTime  @default(now()) @map("created_at")
  updatedAt        DateTime  @updatedAt      @map("updated_at")

  instructorProfile        InstructorProfile?
  refreshTokens            RefreshToken[]
  emailVerificationTokens  EmailVerificationToken[]
  passwordResetTokens      PasswordResetToken[]
  instructorInvites        InstructorInvite[] @relation("InvitedBy")
  auditLogsAsActor         AuditLog[]         @relation("Actor")

  @@map("users")
}

model InstructorProfile {
  userId              String   @id @map("user_id")
  headline            String?
  bioLong             String?  @map("bio_long")
  socialLinksJson     Json?    @map("social_links_json")
  monthlyTokenBudget  Int      @map("monthly_token_budget") // tokens, enforced in P3

  user User @relation(fields: [userId], references: [id], onDelete: Cascade)

  @@map("instructor_profiles")
}

model EmailVerificationToken {
  id         String    @id @default(cuid())
  userId     String    @map("user_id")
  tokenHash  String    @unique @map("token_hash") // sha256 of the random token
  expiresAt  DateTime  @map("expires_at")         // now + 24h
  consumedAt DateTime? @map("consumed_at")
  createdAt  DateTime  @default(now()) @map("created_at")

  user User @relation(fields: [userId], references: [id], onDelete: Cascade)

  @@index([userId])
  @@map("email_verification_tokens")
}

model PasswordResetToken {
  id         String    @id @default(cuid())
  userId     String    @map("user_id")
  tokenHash  String    @unique @map("token_hash")
  expiresAt  DateTime  @map("expires_at")         // now + 1h
  consumedAt DateTime? @map("consumed_at")
  createdAt  DateTime  @default(now()) @map("created_at")

  user User @relation(fields: [userId], references: [id], onDelete: Cascade)

  @@index([userId])
  @@map("password_reset_tokens")
}

model RefreshToken {
  id          String    @id @default(cuid())
  userId      String    @map("user_id")
  familyId    String    @map("family_id")          // shared by rotated descendants
  tokenHash   String    @unique @map("token_hash")
  expiresAt   DateTime  @map("expires_at")          // now + 30d
  revokedAt   DateTime? @map("revoked_at")
  replacedById String?  @map("replaced_by_id")
  userAgent   String?   @map("user_agent")
  ip          String?
  createdAt   DateTime  @default(now()) @map("created_at")

  user User @relation(fields: [userId], references: [id], onDelete: Cascade)

  @@index([userId])
  @@index([familyId])
  @@map("refresh_tokens")
}

model InstructorInvite {
  id                  String    @id @default(cuid())
  email               String
  name                String
  monthlyTokenBudget  Int       @map("monthly_token_budget")
  invitedById         String    @map("invited_by_id")
  tokenHash           String    @unique @map("token_hash")
  expiresAt           DateTime  @map("expires_at")            // now + 7d
  acceptedAt          DateTime? @map("accepted_at")
  acceptedUserId      String?   @map("accepted_user_id")
  createdAt           DateTime  @default(now()) @map("created_at")

  invitedBy User @relation("InvitedBy", fields: [invitedById], references: [id])

  @@index([email])
  @@map("instructor_invites")
}

enum AuditAction {
  AUTH_LOGIN_OK
  AUTH_LOGIN_FAIL
  AUTH_LOGOUT
  AUTH_PASSWORD_CHANGED
  AUTH_PASSWORD_RESET_REQUESTED
  AUTH_PASSWORD_RESET_COMPLETED
  AUTH_EMAIL_VERIFIED
  USER_ROLE_CHANGED
  USER_SUSPENDED
  USER_UNSUSPENDED
  INSTRUCTOR_INVITE_SENT
  INSTRUCTOR_INVITE_ACCEPTED
}

model AuditLog {
  id            String      @id @default(cuid())
  actorUserId   String?     @map("actor_user_id")  // null for system / unauth attempts
  action        AuditAction
  targetType    String      @map("target_type")    // e.g. "User", "InstructorInvite"
  targetId      String?     @map("target_id")
  metadataJson  Json?       @map("metadata_json")
  ip            String?
  userAgent     String?     @map("user_agent")
  createdAt     DateTime    @default(now()) @map("created_at")

  actor User? @relation("Actor", fields: [actorUserId], references: [id])

  @@index([actorUserId])
  @@index([targetType, targetId])
  @@index([action, createdAt])
  @@map("audit_logs")
}
```

Notes:
- All token tables store **hashes** (`sha256`), never the raw token.
- Refresh tokens use a `familyId` to detect reuse (a single revoked
  token in a family revokes the rest).
- Audit log is append-only; no UPDATE / DELETE operations are permitted
  via Prisma client repos.

## API surface

All routes are mounted under `/api`. JSON bodies. Error envelope per
CLAUDE.md §9.

### Public

| Method | Path                              | Purpose                                  |
| ------ | --------------------------------- | ---------------------------------------- |
| POST   | `/auth/register`                  | Student self-signup                      |
| POST   | `/auth/login`                     | Login → access token + refresh cookie    |
| POST   | `/auth/refresh`                   | Rotate refresh + issue new access token  |
| POST   | `/auth/logout`                    | Revoke current refresh token family      |
| POST   | `/auth/verify-email`              | Consume signup verification token        |
| POST   | `/auth/request-password-reset`    | Always 200; sends reset email if exists  |
| POST   | `/auth/reset-password`            | Consume password reset token             |
| GET    | `/auth/invite/:token`             | Preview invite (email, name, expiry)     |
| POST   | `/auth/invite/:token/accept`      | Set password, complete instructor signup |

### Authenticated (any role)

| Method | Path                       | Purpose                                |
| ------ | -------------------------- | -------------------------------------- |
| GET    | `/me`                      | Current user profile                   |
| PATCH  | `/me`                      | Update name, timezone, bio, avatar URL |
| POST   | `/me/avatar`               | Upload avatar (multipart) → storage    |
| POST   | `/me/change-password`      | Verify old password, set new           |

### Admin

| Method | Path                                          | Purpose                                       |
| ------ | --------------------------------------------- | --------------------------------------------- |
| GET    | `/admin/users`                                | Paginated list (filter by role, status, q)    |
| GET    | `/admin/users/:id`                            | User detail                                   |
| PATCH  | `/admin/users/:id`                            | Suspend/unsuspend, change role (audited)      |
| POST   | `/admin/users/:id/resend-verification`        | Re-send the verification email                |
| POST   | `/admin/instructors/invite`                   | Create invite, email signed link              |
| GET    | `/admin/instructors/invites`                  | List pending / accepted / expired invites     |
| DELETE | `/admin/instructors/invites/:id`              | Revoke an unaccepted invite                   |
| GET    | `/admin/audit-log`                            | Read-only paginated audit feed                |

### Auth requirements

| Endpoint group                    | Guard                                      |
| --------------------------------- | ------------------------------------------ |
| `/auth/*`                         | Public, throttled                          |
| `/me/*`                           | `JwtAuthGuard`                             |
| `/admin/*`                        | `JwtAuthGuard` + `RolesGuard('ADMIN')`     |

### Throttling defaults

- `/auth/login`: 5 attempts / minute per IP + per email (whichever hits
  first).
- `/auth/register`: 3 / hour per IP.
- `/auth/request-password-reset`: 3 / hour per IP + per email.
- `/auth/refresh`: 30 / minute per IP.
- `/auth/invite/:token/accept`: 5 / hour per IP.

## UI surface

| Route                              | Purpose                                                | Role gating                |
| ---------------------------------- | ------------------------------------------------------ | -------------------------- |
| `/sign-up`                         | Student signup form                                    | public; redirects if logged in |
| `/sign-in`                         | Login form                                             | public                     |
| `/verify-email?token=…`            | Consumes verification token                            | public                     |
| `/forgot-password`                 | Request reset                                          | public                     |
| `/reset-password?token=…`          | Set new password                                       | public                     |
| `/invite/:token`                   | Instructor invite acceptance (preview + set password)  | public                     |
| `/dashboard`                       | Placeholder landed-in screen (real content in P2)      | authenticated              |
| `/me`                              | Profile read + edit, avatar upload, change password    | authenticated              |
| `/admin`                           | Admin home (links to users + invites)                  | ADMIN                      |
| `/admin/users`                     | User list with filters                                 | ADMIN                      |
| `/admin/users/:id`                 | User detail (suspend, role change, resend verification)| ADMIN                      |
| `/admin/instructors/invite`        | New invite form                                        | ADMIN                      |
| `/admin/instructors/invites`      | Invite list                                            | ADMIN                      |
| `/admin/audit-log`                 | Audit log table                                        | ADMIN                      |

Auth UI uses shadcn/ui form components. Server-side `middleware.ts`
gates `/dashboard/*`, `/me/*`, `/admin/*`.

## Configuration

New env vars on top of Phase 0:

| Var                          | Required | Default                              | Notes                                  |
| ---------------------------- | -------- | ------------------------------------ | -------------------------------------- |
| `JWT_ACCESS_SECRET`          | yes      | —                                    | Min 32 chars                           |
| `JWT_REFRESH_SECRET`         | yes      | —                                    | Min 32 chars                           |
| `JWT_ACCESS_TTL_SEC`         | yes      | `900` (15 min)                       |                                        |
| `JWT_REFRESH_TTL_SEC`        | yes      | `2592000` (30 days)                  |                                        |
| `BCRYPT_COST`                | yes      | `12`                                 |                                        |
| `AUTH_COOKIE_DOMAIN`         | yes      | `localhost`                          |                                        |
| `AUTH_COOKIE_SECURE`         | yes      | `false` in dev, `true` in prod       |                                        |
| `WEB_PUBLIC_URL`             | yes      | `http://localhost:3000`              | Used in email links                    |
| `EMAIL_VERIFICATION_TTL_HRS` | yes      | `24`                                 |                                        |
| `PASSWORD_RESET_TTL_HRS`     | yes      | `1`                                  |                                        |
| `INSTRUCTOR_INVITE_TTL_DAYS` | yes      | `7`                                  |                                        |
| `DEFAULT_INSTRUCTOR_BUDGET`  | yes      | `1000000`                            | Tokens/month default for new invites; consumed in P3 |
| `ADMIN_EMAIL`                | yes      | (declared P0)                        | Seeded if absent                       |
| `ADMIN_PASSWORD`             | yes      | (declared P0)                        | Seeded if absent; rotated on next login recommended |

Boot fails loud if any of these are missing in non-test env.

## Testing plan

### Unit tests

- Password hashing service: bcrypt cost honoured; rehash detection on
  cost upgrade.
- Token services: hash-only persistence; expiry check; idempotent
  consumption; reuse-detection on refresh family.
- Audit log writer: never throws (failures logged but do not break
  caller flow).
- `RolesGuard` matrix across ADMIN/INSTRUCTOR/STUDENT.
- DTO validators: reject weak passwords (min 10 chars, mixed classes),
  reject invalid timezone strings (`Intl.supportedValuesOf('timeZone')`).

### Integration tests (testcontainers)

- Full signup → verify → login → refresh → logout cycle.
- Refresh-token reuse: replay revoked token revokes whole family,
  returns 401.
- Password reset: token consumed once; second use rejected.
- Instructor invite end-to-end: admin invites → email captured (Mailpit
  HTTP API) → token accepted → instructor logs in → audit rows present.
- Admin suspending a user invalidates active refresh tokens.
- Idempotent ADMIN seed: running twice does not duplicate.

### E2E (Playwright)

Named golden flows that must pass:

1. `student-signup-flow`: signup → click verify link from Mailpit →
   land on `/dashboard`.
2. `login-and-refresh`: log in, force access-token expiry, next
   request seamlessly refreshes.
3. `password-reset`: request reset → consume link → log in with new
   password → audit row exists.
4. `admin-invites-instructor`: admin issues invite → invite email
   present in Mailpit → invite preview screen renders.
5. `instructor-completes-invite`: instructor accepts invite, sets
   password, logs in, sees instructor-only nav.
6. `admin-suspends-user`: admin suspends a logged-in student; student's
   next API call returns 401 with `code: "ACCOUNT_SUSPENDED"`.

## Exit criteria

- [ ] All Phase 1 migrations applied cleanly on a fresh DB.
- [ ] All listed E2E flows pass under `pnpm test:e2e`.
- [ ] Coverage on `apps/api` ≥ 70% lines.
- [ ] No `/auth/*` endpoint unthrottled in production config.
- [ ] No raw token (verification, reset, refresh, invite) stored in DB
      — confirmed by inspecting the schema and a grep test.
- [ ] Audit log entries exist for every action enumerated in
      `AuditAction`.
- [ ] Admin seed runs idempotently on every boot.
- [ ] OpenAPI doc regenerated; `openapi-typescript` produces FE types
      with no diff in CI re-run.
- [ ] Mailpit captures every outbound email in dev; no email leaves the
      box.

## Risks & open questions

- **Refresh-token theft detection**: replay-based revocation is the
  baseline; consider adding a server-side blocklist of access-token JTIs
  for instant logout-everywhere if perf allows. Decision can wait until
  Phase 7.
- **Rate-limit identity at the edge**: NestJS Throttler uses IP by
  default. Deciding on a per-email key for `/auth/login` requires care
  to avoid leaking which emails exist via timing — keep the response
  shape and timing constant for "user not found" vs "wrong password".
- **Email deliverability in prod**: Phase 1 supports SMTP and
  provider-api drivers but does not yet include SPF/DKIM/DMARC ops.
  Tracked for Phase 7 hardening.
- **Avatar moderation**: Phase 1 accepts any image type (allowed list)
  up to N MB but does not scan for NSFW. Tracked for Phase 7.
- **Open question**: should role changes auto-log the affected user out
  of all sessions? Recommendation: yes, revoke their refresh family.
  Confirm before implementation.
