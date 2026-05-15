# Phase 0 — Foundation

## Goal

Establish the monorepo skeleton, the local development stack, and every
provider-agnostic abstraction the rest of the project depends on. This phase
ships **no product features**. Its purpose is to make Phase 1 a pure
business-logic exercise: by the end of Phase 0, a fresh clone can run
`docker compose up -d && pnpm test` and get a green build, with all 11
infrastructure components healthy and every package boundary already drawn
correctly. Doing the boring plumbing first prevents Phase 1+ from leaking
provider SDKs into business code or accumulating ad-hoc env handling.

## Scope (in)

- pnpm workspaces + Turborepo monorepo layout per CLAUDE.md §4.
- `apps/api` — NestJS bare app that boots and exposes `GET /health`.
- `apps/web` — Next.js 15 (App Router) bare app that renders a placeholder
  homepage and a `/health` route handler.
- `packages/shared` — Zod + DTO + enum scaffolding (one example schema).
- `packages/llm` — empty `aiClient` stub re-exporting Vercel AI SDK
  surface; no real provider wired up yet.
- `packages/storage` — `IStorage` interface + three implementations
  (`LocalFsStorage`, `MinioStorage`, `GcsStorage`) selected by
  `STORAGE_DRIVER`. Per-implementation unit tests against a temp fixture.
- `packages/mailer` — `IMailer` interface + two implementations
  (`GmailSmtpMailer`, `ProviderApiMailer` Resend-style HTTP). Selected by
  `MAILER_DRIVER`. Mailpit container used in dev for both modes.
- `packages/observability` — OTel SDK setup helper consumed by `apps/api`
  and `apps/web`; structured logger.
- `packages/config` — Zod-validated env loader that boots fail-loud on
  missing required vars.
- `packages/eslint-config` — shared ESLint config with a custom rule that
  forbids importing `openai`, `@anthropic-ai/sdk`, and
  `@google/generative-ai` outside `packages/llm/`.
- `prisma/schema.prisma` — only the `User` table (just enough to verify
  Prisma migrate works end-to-end).
- `infra/docker-compose.yml` — postgres, mailpit, minio, ollama, grafana,
  loki, tempo, prometheus, otel-collector. Healthchecks on each.
  `docker compose up -d --wait` returns clean.
- `.env.example` covering every flag mentioned in CLAUDE.md §7.
- Vitest configured in every package, with a sample test in each.
- Turbo pipelines for `dev`, `build`, `test`, `lint`, `typecheck`.
- Pre-commit setup with lint-staged.
- README quickstart matching CLAUDE.md §11.

## Scope (out / deferred)

- Authentication, sessions, JWT, refresh tokens — Phase 1.
- Real `User` columns beyond minimum for migration smoke test — Phase 1.
- Any business-domain tables (`Course`, `Batch`, etc.) — Phase 2.
- Any real LLM call or Portkey wiring — Phase 3.
- Any agent implementation (`PlannerAgent`, `AssessorAgent`) — Phase 3 / 5.
- GitHub Actions CI — Phase 7.
- Production Caddy reverse proxy — Phase 7.
- Keycloak — Phase 8.

## Prerequisites

None. This is the entry phase.

## Deliverables

- A fresh clone passes the **golden quickstart** in CLAUDE.md §11.
- Workspace packages installable via `pnpm install` (no peer-dep warnings
  for declared deps).
- Running services after `docker compose up -d --wait`:
  - Postgres on `5432`
  - Mailpit on `1025` (SMTP) + `8025` (UI)
  - MinIO on `9000` (S3) + `9001` (UI)
  - Ollama on `11434`
  - Grafana on `3001`
  - Loki on `3100`
  - Tempo on `3200`
  - Prometheus on `9090`
  - OTel collector on `4317` (gRPC) + `4318` (HTTP)
- API boots on `:4000`, exposes `GET /health` returning
  `{ status: "ok", commit: <sha>, version: <pkg.version> }`.
- Web boots on `:3000`, placeholder homepage rendered, `GET /health`
  returns `{ status: "ok" }`.
- `pnpm db:migrate` applies the initial migration creating `users`.
- All packages export at least one symbol and have a passing Vitest test.
- ESLint rule in CI blocks a deliberately added bad import in a fixture.

## Data model deltas

Only the `User` table — just enough to prove Prisma works. Full Phase 1
columns (verification, suspension, etc.) are added in Phase 1.

```prisma
// prisma/schema.prisma — Phase 0

generator client {
  provider = "prisma-client-js"
}

datasource db {
  provider = "postgresql"
  url      = env("DATABASE_URL")
}

enum UserRole {
  ADMIN
  INSTRUCTOR
  STUDENT
}

model User {
  id            String   @id @default(cuid())
  email         String   @unique
  passwordHash  String   @map("password_hash")
  role          UserRole
  name          String
  createdAt     DateTime @default(now()) @map("created_at")
  updatedAt     DateTime @updatedAt      @map("updated_at")

  @@map("users")
}
```

Reference: this is a stub of the wider model in CLAUDE.md §5; remaining
columns and tables come in Phase 1+.

## API surface

| Method | Path      | Purpose                          | Auth   |
| ------ | --------- | -------------------------------- | ------ |
| GET    | `/health` | Liveness — used by docker + tests | public |

The Next.js side ships an equivalent `GET /health` route handler with the
same response shape, served from `apps/web/app/health/route.ts`.

Swagger is mounted at `/api/docs` even though only `/health` is documented;
this proves the doc pipeline works.

## UI surface

| Route   | Purpose                                            | Role gating |
| ------- | -------------------------------------------------- | ----------- |
| `/`     | Placeholder homepage with project name + a link to `/health` | public |
| `/health` | JSON liveness route handler                      | public      |

No real layout, no navigation, no auth screens.

## Configuration

`.env.example` is the source of truth (CLAUDE.md §7). Phase 0 introduces:

| Var                              | Required | Default                   | Notes                                    |
| -------------------------------- | -------- | ------------------------- | ---------------------------------------- |
| `NODE_ENV`                       | yes      | `development`             | `development | test | production`        |
| `API_PORT`                       | yes      | `4000`                    |                                          |
| `WEB_PORT`                       | yes      | `3000`                    |                                          |
| `DATABASE_URL`                   | yes      | postgres compose default  | Postgres connection string               |
| `STORAGE_DRIVER`                 | yes      | `local`                   | `local | minio | gcs`                    |
| `STORAGE_LOCAL_ROOT`             | if local | `./.local-storage`        |                                          |
| `STORAGE_MINIO_ENDPOINT`         | if minio | `http://localhost:9000`   |                                          |
| `STORAGE_MINIO_ACCESS_KEY`       | if minio | —                         |                                          |
| `STORAGE_MINIO_SECRET_KEY`       | if minio | —                         |                                          |
| `STORAGE_MINIO_BUCKET`           | if minio | `lms`                     |                                          |
| `STORAGE_GCS_PROJECT_ID`         | if gcs   | —                         |                                          |
| `STORAGE_GCS_BUCKET`             | if gcs   | —                         |                                          |
| `STORAGE_GCS_CREDENTIALS_JSON`   | if gcs   | —                         | inline JSON or path                      |
| `MAILER_DRIVER`                  | yes      | `gmail-smtp`              | `gmail-smtp | provider-api`              |
| `MAILER_FROM`                    | yes      | `lms@localhost`           |                                          |
| `MAILER_GMAIL_HOST`              | if smtp  | `localhost` (mailpit)     |                                          |
| `MAILER_GMAIL_PORT`              | if smtp  | `1025`                    |                                          |
| `MAILER_GMAIL_USER`              | if smtp  | (empty in dev)            |                                          |
| `MAILER_GMAIL_PASS`              | if smtp  | (empty in dev)            |                                          |
| `MAILER_PROVIDER_API_URL`        | if api   | —                         |                                          |
| `MAILER_PROVIDER_API_KEY`        | if api   | —                         |                                          |
| `LLM_PROVIDER`                   | yes      | `ollama`                  | `portkey | ollama` (real wiring in P3)   |
| `LLM_PRIMARY_MODEL`              | yes      | `llama3.1:8b`             | Placeholder until P3                     |
| `LLM_FALLBACK_MODEL`             | no       | —                         |                                          |
| `OTEL_EXPORTER_OTLP_ENDPOINT`    | yes      | `http://localhost:4318`   |                                          |
| `OTEL_SERVICE_NAME_API`          | yes      | `lms-api`                 |                                          |
| `OTEL_SERVICE_NAME_WEB`          | yes      | `lms-web`                 |                                          |
| `LOG_LEVEL`                      | yes      | `info`                    | `trace | debug | info | warn | error`    |
| `ADMIN_EMAIL`                    | yes      | `admin@lms.local`         | Used by Phase 1 seed; declared here for `.env.example` completeness |
| `ADMIN_PASSWORD`                 | yes      | `change-me`               | Same as above                            |

`packages/config` parses these with Zod and prints a table of resolved
values at boot (with secrets redacted).

## Testing plan

### Unit tests (Vitest, in-package)

- `packages/config`: rejects when a required var is missing; coerces
  numeric strings; redacts secrets in `toString`.
- `packages/storage`: each implementation passes the same shared contract
  test (`put`, `get`, `delete`, `getSignedUrl`, `exists`) against a temp
  fixture (local fs root, in-process Minio via testcontainers, GCS via
  `@google-cloud/storage` with the mock server).
- `packages/mailer`: each implementation sends a known message; assertions
  via Mailpit's HTTP API for SMTP, via a mocked HTTP server for the
  provider-api driver.
- `packages/observability`: tracer + logger exported and usable; no-op
  exporter in test mode does not throw.
- `packages/eslint-config`: ESLint rule fixture proves a banned import is
  flagged outside `packages/llm/` and allowed inside.

### Integration tests

- `apps/api`: bring up Postgres via testcontainers; run Prisma migrate;
  insert and read a `User`; assert migration is idempotent.
- `apps/api`: `/health` end-to-end inside a NestJS test module.

### E2E (Playwright)

Phase 0 has one minimal E2E to prove the harness works:

- `phase0-smoke`: `pnpm dev` + `docker compose up --wait`; visit `/`,
  assert placeholder text; visit `/health`, assert `{status: "ok"}`.

## Exit criteria

- [ ] `git clone` → `pnpm install` → `cp .env.example .env`
      → `docker compose up -d --wait` → `pnpm db:migrate`
      → `pnpm test` is green.
- [ ] `pnpm build` succeeds for every workspace package.
- [ ] `pnpm lint` and `pnpm typecheck` pass with zero warnings (warnings
      treated as errors in CI).
- [ ] `docker compose ps` shows every service `healthy`.
- [ ] ESLint rule blocks a fixture file that imports `openai` from
      outside `packages/llm/`; allows the same import inside.
- [ ] Storage and mailer contract tests run against all configured
      drivers (local + minio for storage; smtp + provider-api for mail).
- [ ] OTel traces from `apps/api` `/health` are visible in Tempo via
      Grafana Explore.
- [ ] Quickstart in CLAUDE.md §11 reproduces this from a fresh checkout.

## Risks & open questions

- **Testcontainers on macOS performance**: spinning up Postgres + Minio
  per test file can be slow. Mitigation: scope containers to suite, not
  test, and gate slow integration suites behind a `pnpm test:integration`
  task.
- **GCS test coverage without real credentials**: the GCS implementation
  relies on a mock server (`fake-gcs-server`). Confirm parity with real
  GCS signed URL semantics before any prod use.
- **Ollama image size**: pulling `llama3.1:8b` adds ~5 GB to first-time
  setup. Decision: ship compose with the model name configured but
  document the manual `ollama pull` step; do not block compose health on
  it.
- **OTel collector config drift**: `infra/otel-collector.yaml` must stay
  in sync with the SDK setup. Add a smoke test that emits one span and
  asserts it lands in Tempo.
- **Open question**: should `packages/storage` expose multipart upload in
  P0, or only single-shot `put`? Recommendation: single-shot only;
  multipart added in Phase 3 when material upload sizes matter.
