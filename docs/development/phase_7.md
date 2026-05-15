# Phase 7 — Hardening & CI/CD

## Goal

Take the platform from "works on my machine and in dev compose" to
"deployable, observable, defensible, and recoverable in production".
This phase introduces GitHub Actions for CI and image builds, a Caddy
reverse proxy with automatic HTTPS, tightened rate limiting and brute
force protection, audit log endpoints, scripted backup + a documented
restore runbook, Web Push notifications (now possible because HTTPS is
finally available), and an optional escalation from `@Cron` (P6) to a
durable BullMQ job queue if reminder reliability demands it. It is the
right slice now because all functional surfaces (P0–P6) are in place
and the next thing that can go wrong in real use is operational, not
behavioural.

---

## Scope (in)

- GitHub Actions workflows:
  - `ci.yml` — lint + typecheck + unit + integration on every PR.
  - `build-images.yml` — build and push Docker images to GHCR on
    every merge to `main`.
  - `release.yml` — tagged release: builds versioned images, attaches
    a SBOM and changelog to a GitHub release.
- Caddy reverse proxy added to `infra/docker-compose.yml` with
  automatic HTTPS via Let's Encrypt; configurable to skip TLS in
  local (`CADDY_TLS=internal`).
- Rate limiting tightened via NestJS `@nestjs/throttler`:
  - Global default raised in stringency.
  - `/auth/*` extra-tight (login, refresh, password reset).
  - `/attempts/*/answer` and `/attempts/*/stream` per-user limits to
    cap LLM spend.
- Brute force protection on `/auth/login`: progressive backoff +
  account lock after N failures within a rolling window.
- Audit log endpoints: `GET /admin/audit` with filters by actor,
  action, target type, date range. (Audit rows already written across
  earlier phases per CLAUDE.md §5.)
- Backup script `scripts/backup.sh`:
  - `pg_dump` of the database.
  - Snapshot of the storage volume (whichever `STORAGE_DRIVER` is
    active — local rsync, MinIO `mc mirror`, or `gsutil` for GCS).
  - Encryption with `age` against a public key from env.
  - Upload target documented but pluggable: S3-compatible bucket by
    default; specific provider TBD per deployment.
- Restore runbook documented inline in this phase doc (see Operational
  runbooks below).
- Web Push notifications: VAPID key generation, subscription endpoint,
  push dispatch from the existing `NotificationDispatcher` (P6).
- Optional escalation: BullMQ + Redis if P6's `@Cron` proves
  insufficient (decision criteria documented; default in P7 = stay on
  `@Cron`).
- Secrets handling documented:
  - `.env` for dev; never committed.
  - Prod: docker compose secrets (file-mounted) or a sealed-secrets
    operator if Kubernetes is later introduced.
  - Portkey virtual keys preferred (CLAUDE.md §9) so a leak is
    revocable without rotating provider keys.
- Healthcheck endpoints: split `/health/liveness` (process up) and
  `/health/readiness` (DB + storage + LLM gateway reachable).

## Scope (out / deferred)

- Multi-region deploy or Kubernetes manifests.
- Provider-specific deploy scripts (Fly.io, Render, GCP Cloud Run,
  etc.) — leave to whoever runs the install.
- Penetration test by external party (recommended but out of scope
  for this phase).
- ELK / OpenSearch (Phase 8 if scale requires).
- Capacitor / app stores (Phase 8).
- Keycloak / OIDC (Phase 8).

## Prerequisites

- All of P0–P6 complete and green.
- A domain name and DNS pointed at the deployment host (for Let's
  Encrypt to work end-to-end). For staging, a `*.example.com` wildcard
  is acceptable.
- A GitHub repository with Actions enabled and write access to GHCR.

## Deliverables (concrete artifacts)

- `.github/workflows/ci.yml`, `build-images.yml`, `release.yml`.
- `infra/caddy/Caddyfile` and a `caddy` service in
  `infra/docker-compose.yml` (with a separate
  `infra/docker-compose.prod.yml` override binding `:80`/`:443` and
  enabling Let's Encrypt).
- Updated `apps/api/src/auth/*` with throttler + brute-force guard.
- New `AuditController` (`/admin/audit`) backed by existing
  `AuditLog` table.
- `scripts/backup.sh` and `scripts/restore.sh` with `--dry-run` flags.
- `apps/api/src/notifications/web-push.service.ts` and FE
  `NotificationsClient.subscribePush()` flow.
- `BullMQ` migration documented but only enacted if the trigger
  conditions below are met during this phase.
- ADR proposals:
  - `docs/architecture/019-reverse-proxy-and-tls.md`.
  - `docs/architecture/020-backup-and-restore.md`.

---

## Data model deltas

```prisma
// Web Push subscriptions, one user can have many devices.
model WebPushSubscription {
  id           String   @id @default(cuid())
  userId       String
  user         User     @relation(fields: [userId], references: [id], onDelete: Cascade)
  endpoint     String   @unique
  p256dh       String
  auth         String
  userAgent    String?
  createdAt    DateTime @default(now())
  lastSeenAt   DateTime @default(now())
  failureCount Int      @default(0)

  @@index([userId])
}

// Brute-force counters; ephemeral, can be pruned by job.
model AuthAttempt {
  id           String   @id @default(cuid())
  identifier   String   // email lowercased OR ip:address
  kind         AuthAttemptKind
  succeeded    Boolean
  ipHash       String   // SHA-256 of remote IP, never raw
  userAgent    String?
  attemptedAt  DateTime @default(now())

  @@index([identifier, attemptedAt])
}

enum AuthAttemptKind {
  LOGIN
  REFRESH
  PASSWORD_RESET_REQUEST
  PASSWORD_RESET_CONFIRM
}
```

The existing `AuditLog` table (CLAUDE.md §5) is unchanged — Phase 7
only adds a query surface over it.

Optional, only if BullMQ is adopted:

```prisma
// We do not store BullMQ job state in Postgres — Redis is its store.
// No Prisma changes required for the queue itself.
```

---

## API surface

| Method | Path                                       | Purpose                                                         | Auth gate                  |
| ------ | ------------------------------------------ | --------------------------------------------------------------- | -------------------------- |
| GET    | `/admin/audit`                             | Filter audit log: `actorUserId, action, targetType, from, to`.  | ADMIN only.                |
| GET    | `/admin/audit/:id`                         | Single audit row detail.                                        | ADMIN only.                |
| GET    | `/health/liveness`                         | Returns `200` if process is up.                                 | Public.                    |
| GET    | `/health/readiness`                        | `200` only when DB + storage + LLM gateway all reachable.       | Public (rate-limited).     |
| POST   | `/me/web-push/subscribe`                   | Body: `{ endpoint, keys: { p256dh, auth }, userAgent }`.        | Authenticated user.        |
| POST   | `/me/web-push/unsubscribe`                 | Body: `{ endpoint }`.                                           | Authenticated user.        |
| GET    | `/me/web-push/vapid-public-key`            | Returns the configured VAPID public key for SW subscription.    | Authenticated user.        |
| GET    | `/admin/scheduler/status`                  | Lists registered jobs, last run, next run.                      | ADMIN only.                |

Throttler tightening (no new endpoints, just config):

- `/auth/login`: 5 req / minute / IP, 10 req / minute / email.
- `/auth/refresh`: 30 req / minute / user.
- `/auth/password-reset/*`: 3 req / hour / IP, 3 req / hour / email.
- `/attempts/*/answer`: 60 req / minute / user.
- `/attempts/*/stream`: 1 active connection / user / attempt.
- Global default: 120 req / minute / IP for unauthenticated, 600
  req / minute / user for authenticated.

Brute-force lockout:

- After 10 failed `LOGIN` attempts within 15 minutes for the same
  email, lock the account for 30 minutes (returns `423 Locked`).
- Successful login resets the counter.
- Lockout writes an `AuditLog` row with `action = AUTH_ACCOUNT_LOCKED`.

---

## UI surface

| Route / Component                       | Purpose                                                                 | Role gating          |
| --------------------------------------- | ----------------------------------------------------------------------- | -------------------- |
| `/admin/audit`                          | Searchable, filterable audit log view with CSV export.                  | ADMIN only.          |
| `/me/settings/notifications`            | Toggle Web Push opt-in; lists active subscriptions + revoke per-device. | Authenticated user.  |
| `/admin/scheduler`                      | Lists jobs and last/next run times (read-only).                         | ADMIN only.          |
| Login page                              | Surfaces the lockout countdown when `423 Locked`.                       | Public.              |

---

## Configuration (new env vars)

Add to `.env.example`:

- `CADDY_DOMAIN` (e.g. `lms.example.com`).
- `CADDY_TLS` (`internal` for local dev, `letsencrypt` in prod).
- `CADDY_ACME_EMAIL` (required when `CADDY_TLS=letsencrypt`).
- `RATE_LIMIT_GLOBAL_PER_MIN_AUTH` (default `600`).
- `RATE_LIMIT_GLOBAL_PER_MIN_PUBLIC` (default `120`).
- `RATE_LIMIT_LOGIN_PER_MIN_IP` (default `5`).
- `BRUTE_FORCE_LOCK_THRESHOLD` (default `10`).
- `BRUTE_FORCE_LOCK_WINDOW_MIN` (default `15`).
- `BRUTE_FORCE_LOCK_DURATION_MIN` (default `30`).
- `WEB_PUSH_VAPID_PUBLIC_KEY`, `WEB_PUSH_VAPID_PRIVATE_KEY`,
  `WEB_PUSH_VAPID_SUBJECT` (mailto: address).
- `BACKUP_TARGET_S3_BUCKET`, `BACKUP_TARGET_S3_PREFIX`,
  `BACKUP_AGE_RECIPIENT_PUBLIC_KEY`.
- `JOB_QUEUE_DRIVER` (`cron | bullmq`, default `cron`).
- `REDIS_URL` (only required when `JOB_QUEUE_DRIVER=bullmq`).
- `HEALTH_READINESS_TIMEOUT_MS` (default `2000`).

All Zod-validated at boot per CLAUDE.md §7.

---

## CI/CD pipelines

### `ci.yml` (on pull_request, push to feature branches)

Jobs (run in parallel where possible):

1. `setup` — checkout, setup-node@20, pnpm install with cache.
2. `lint` — `pnpm lint` (root turbo command).
3. `typecheck` — `pnpm typecheck`.
4. `unit` — `pnpm test:unit` (Vitest).
5. `integration` — Postgres service container; `pnpm test:integration`
   (testcontainers; cached Docker layers via buildx).
6. `build` — `pnpm build` to ensure both apps compile.

PR is blocked from merge unless all jobs pass. No e2e in PR (too slow);
e2e runs nightly on `main` (additional `nightly-e2e.yml`, scoped out
of P7 if it pushes the timeline; document as follow-up).

### `build-images.yml` (on push to `main`)

1. Build `apps/api` and `apps/web` Docker images with buildx.
2. Tag: `ghcr.io/<org>/lms-api:<sha>` and `:main`. Same for `lms-web`.
3. Push to GHCR (using `GITHUB_TOKEN` with `packages: write`).
4. Generate SBOM (`syft`) and attach as build artifact.

### `release.yml` (on tag `v*.*.*`)

1. Re-tag the latest `:main` images with `:vX.Y.Z` and `:latest`.
2. Generate changelog from commits since previous tag.
3. Create GitHub Release with changelog + SBOM.

Secrets needed:

- `GITHUB_TOKEN` (default).
- For e2e (when added): `LLM_PROVIDER=ollama` so no real Portkey key
  is needed in CI.

---

## Operational runbooks

### Backup

```
# Run on the host that has access to the postgres container + storage volume.
scripts/backup.sh \
  --postgres-container lms-postgres \
  --storage-driver "$STORAGE_DRIVER" \
  --output /tmp/lms-backup-$(date -u +%Y%m%dT%H%M%SZ).tar.age \
  --encrypt-to "$BACKUP_AGE_RECIPIENT_PUBLIC_KEY"
```

The script:

1. `pg_dump --format=custom` against the running container.
2. Snapshots storage:
   - `local`: tars the volume.
   - `minio`: `mc mirror` into a temp dir, then tar.
   - `gcs`: `gsutil -m cp -r` into a temp dir, then tar.
3. Concatenates into a single `tar` and pipes through `age -r <key>`.
4. Uploads to `BACKUP_TARGET_S3_BUCKET/BACKUP_TARGET_S3_PREFIX/` with
   server-side encryption.

Schedule: cron on the host (or a sidecar container) at 02:00 UTC daily.

### Restore

```
# Acquire the encrypted bundle.
age -d -i /path/to/age-secret-key restored.tar.age | tar -x -C /tmp/restore

# Restore Postgres into a fresh database.
pg_restore --clean --create --no-owner -d postgres /tmp/restore/postgres.dump

# Restore storage volume (per driver).
# local: rsync into the volume mount.
# minio: mc mirror back.
# gcs: gsutil -m cp back.

# Re-issue any short-lived secrets that may be in the dump (refresh cookies,
# pending email verification tokens) — invalidate them in the DB or expire
# manually before bringing the API back up.

# Bring the stack up.
docker compose -f infra/docker-compose.yml -f infra/docker-compose.prod.yml up -d
```

The restore runbook is part of this doc and must be exercised in the
roundtrip E2E test.

---

## Testing plan

### Unit (Vitest)

- Throttler config matches the documented limits per route (config
  snapshot test).
- Brute-force service: locks at threshold, unlocks after window,
  resets on success.
- VAPID subscribe / unsubscribe rejects malformed payloads.
- `WebPushService.send` swallows known transient errors (410 gone →
  delete subscription) and re-throws unknown ones.
- Backup script unit-tested via shellcheck + bats for argument
  parsing and dry-run path (no real DB writes).

### Integration (testcontainers)

- `/admin/audit` returns rows produced by earlier services (write
  audit row in setup, query, assert filter behaviour).
- `/health/readiness` returns 503 when Postgres is killed mid-test
  and 200 when restored.
- Web Push subscribe + send: a fake push receiver intercepts the
  payload; assert the kind and payload match the in-app
  notification's `payloadJson`.
- Account lockout end-to-end against the real auth flow.

### E2E (Playwright + docker compose)

1. **ci-runs-on-pr** — meta-test: a sample PR triggers `ci.yml` and
   all jobs pass. (Verify via `gh run list` in CI itself; also
   document a manual smoke step.)
2. **prod-compose-up-with-https-cert** — bring up
   `docker-compose.prod.yml` against a staging domain; assert HTTPS
   responds with a valid Let's Encrypt cert (via `curl -v`).
3. **brute-force-blocks-after-N-attempts** — script 10 failed logins
   for the same email; assert `423` on the 11th; wait the lockout
   window in fast-forward (test-only endpoint resets clock); assert
   recovery.
4. **backup-restore-roundtrip** — seed data → run backup → drop DB
   and storage volume → run restore → assert all seeded entities are
   present and a sample login works.
5. **web-push-permission-and-delivery** — opt in to push from
   settings; trigger a `TASK_ASSIGNED` event; assert SW receives the
   push (Playwright service worker hooks).
6. **rate-limit-on-attempt-answer** — exceed `60 req / min / user`;
   assert `429` with `Retry-After` header.
7. **audit-search-by-action** — admin queries `action =
   ASSESSMENT_OVERRIDE` and gets the expected matching rows from a
   prior P5 override.

---

## Exit criteria (checklist)

- [ ] All three GitHub Actions workflows present, green on a sample
      PR and a sample tag.
- [ ] Caddy serves the API + Web behind a valid TLS cert in staging.
- [ ] Throttler limits enforced and tested per route.
- [ ] Account lockout works and writes an audit row.
- [ ] `GET /admin/audit` returns paginated, filterable results.
- [ ] `scripts/backup.sh` runs cleanly and `scripts/restore.sh`
      restores into a clean compose; the roundtrip E2E test passes.
- [ ] Web Push works on Chrome desktop + Android; degrades gracefully
      on Safari/iOS (which only supports Web Push from PWAs installed
      to home screen — document the limitation).
- [ ] `JOB_QUEUE_DRIVER=bullmq` either adopted (and Redis added to
      compose) or explicitly deferred with a decision note in
      ADR-018.
- [ ] `/health/liveness` and `/health/readiness` documented for the
      reverse proxy and any future orchestrator.
- [ ] No secret or `.env` committed; `git-secrets` (or equivalent) on
      the CI lint job catches accidental commits.
- [ ] ADR-019 and ADR-020 committed.
- [ ] CLAUDE.md §5 still matches `prisma/schema.prisma` after the new
      tables.

---

## Risks & open questions

- **Let's Encrypt rate limits.** During CI rehearsals against a real
  domain, we can blow the staging rate limit. Use Let's Encrypt's
  staging endpoint in non-prod (`CADDY_TLS=letsencrypt-staging`).
- **Backup target choice.** The script supports any S3-compatible
  bucket but the actual deployment target is undecided. Until we
  pick one, the backup uploads will be a smoke (local file) and the
  prod runbook section will note "TODO: configure
  `BACKUP_TARGET_S3_BUCKET`".
- **Restore PII contention.** The restore brings back hashed
  credentials and refresh tokens; if the backup is ever leaked, the
  damage radius is large. Mitigations: encrypt with `age` to a key
  held offline; document a rotation procedure for password reset
  links and refresh tokens after any forced restore.
- **BullMQ vs `@Cron` decision.** Trigger to escalate: missed
  reminders observed in production, or scaling beyond a single API
  process. Keep the upgrade path open by routing all scheduled work
  through a thin `JobsService` so the implementation can swap
  without touching callers.
- **Web Push on iOS.** Requires the user to add the PWA to home
  screen first; UX should explain this clearly. iOS prompts and
  behavior change between releases — keep the feature behind a
  user-controlled toggle.
- **Audit log growth.** `AuditLog` will dominate row count over
  time. Add a documented retention policy (e.g. 1 year hot, archive
  to cold storage thereafter); implement archival in P8 if needed.
