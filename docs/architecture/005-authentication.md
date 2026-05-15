# ADR-005: Authentication — Email + JWT in P1; Keycloak in P8

- Status: Accepted
- Date: 2026-05-15
- Deciders: Anshul Jain (project owner)
- Phase introduced: Phase 1 (initial); Phase 8 (Keycloak swap-in)

## Context

The platform has three roles (`CLAUDE.md` §2): `ADMIN`, `INSTRUCTOR`,
`STUDENT`. Account creation paths differ:

- ADMIN seeded from env on first boot.
- INSTRUCTOR via admin invite (signed link → set password → log in).
- STUDENT self-register via email/password, or instructor invite.

Phase 1 must ship a working auth surface without dragging in a heavy
identity stack. Phase 8 introduces Keycloak with Google OAuth — but
the FE and most BE code must not change when that swap happens.

Other constraints:

- PWA + future mobile (Capacitor) — needs a bearer-token flow that
  works cleanly for both web and webview clients.
- Email verification + password reset (P1 deliverables).
- Audit-friendly: account events must land in `AuditLog`.

## Decision

**Phase 1**: email + password with `bcrypt` (cost factor 12) and JWT.

- **Access token**: JWT, 15-minute lifetime, sent in
  `Authorization: Bearer <token>`.
- **Refresh token**: opaque random string, 30-day lifetime, stored in an
  HTTP-only, `Secure`, `SameSite=Lax` cookie scoped to the API origin.
  Stored hashed in the DB so we can revoke on logout / suspension.
- **Refresh rotation**: every refresh issues a new refresh token and
  invalidates the prior one (DB row check). Replay of an invalidated
  refresh triggers a forced logout of the entire family.
- **Role change / suspension revokes refresh tokens**: when an admin
  changes a user's role or suspends them, the same transaction deletes
  every refresh-token row for that user. Subsequent `/auth/refresh`
  calls return 401; the user must log in again to receive a token
  bearing the new role. This caps stale-role privilege at the
  15-minute access-token lifetime.
- **Email verification**: signed time-bounded URL token (HMAC over
  `userId|purpose|exp`) — no DB row required beyond the user record.
- **Password reset**: same signed-token pattern, single-use enforced by
  bumping a `password_reset_nonce` column.
- **Routes**: `/auth/register`, `/auth/login`, `/auth/refresh`,
  `/auth/logout`, `/auth/verify-email`, `/auth/request-password-reset`,
  `/auth/reset-password`, `/auth/me`.

**Phase 8**: Keycloak (with Google OAuth) sits behind the same `/auth/*`
API surface. The BE validates Keycloak-issued JWTs (JWKS) instead of
self-issued ones; the FE keeps the same calls.

## Consequences

### Positive

- Tight control over the P1 auth experience without operational cost
  of running Keycloak from day one.
- Bearer access token plays nicely with both PWA fetch and future
  Capacitor webview clients.
- Refresh-token rotation + DB row check defends against replay; we can
  hard-revoke any session by deleting the row.
- Signed verification/reset tokens avoid extra DB rows for short-lived
  flows.
- Stable `/auth/*` contract means the Phase 8 swap to Keycloak does not
  ripple into FE work.

### Negative / Trade-offs

- We own password storage, lockout policies, and reset flows in P1 —
  classic auth pitfalls we must implement carefully.
- Cookie + bearer hybrid means CORS and CSRF need careful configuration
  (refresh endpoint is the only cookie-using endpoint and uses
  `SameSite=Lax` plus an explicit origin allowlist).
- JWTs are not server-revocable mid-lifetime; we accept the 15-minute
  window. Suspension and role changes take effect on next refresh
  (which fails after refresh-token revocation, forcing re-login).

### Neutral

- Argon2 considered but bcrypt cost-12 is sufficient for P1 and avoids
  another native dep. Revisit in Phase 7 hardening.
- We do not implement social login in P1; Phase 8's Keycloak handles
  Google directly.
- 2FA is not in scope until Phase 8.

## Alternatives considered

- **NextAuth / Auth.js**: tightly coupled to the Next.js FE, awkward
  for a webview/native client and for admin BE-only flows like seeding.
- **Lucia**: lightweight and pleasant, but younger and lacks the broad
  documentation surface NestJS users expect.
- **Keycloak from day one**: another container, another data store,
  another admin surface to learn during P1. Postpones core LMS work.
- **Auth0 / Clerk**: vendor lock + cost from day one for a project that
  wants to be portable.
- **Session cookies only (no JWT)**: simpler in pure web, but harder
  for the future Capacitor wrapper and clearer to keep separate.

## Related ADRs / docs

- `CLAUDE.md` §2, §3.
- ADR-003 (backend) — guards/interceptors implement the JWT verification.
- ADR-010 (mailer) — verification + reset emails go through `IMailer`.
- `docs/development/phase_1.md` — initial auth deliverable.
- `docs/development/phase_8.md` — Keycloak swap-in.
