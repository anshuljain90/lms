# ADR-020: Configuration management — Zod-validated env, .env.example as source of truth

- Status: Accepted
- Date: 2026-05-15
- Deciders: Anshul Jain (project owner)
- Phase introduced: Phase 0 (`packages/config` scaffold), Phase 1 (first required vars), Phase 7 (prod secrets)

## Context

The platform has interface-driven integrations (CLAUDE.md §3) where
the chosen driver (`STORAGE_DRIVER`, `MAILER_DRIVER`, `LLM_PROVIDER`)
determines which other env vars are required. Without strict validation
this becomes a class of bugs that only surface on the first request
that touches the missing dependency — sometimes hours after boot.

Constraints:

- TypeScript strict mode everywhere (CLAUDE.md §9). Config consumers
  expect typed values, not `string | undefined`.
- Single developer + occasional contributors; secret hygiene must
  not depend on memory.
- Multiple environments: local dev, CI, staging (TBD), prod (TBD).
- `.env.example` already named as the source of truth (CLAUDE.md §7).

## Decision

### Validation

- Every env var passes through a **Zod schema** in `packages/config`
  before any code reads it.
- Schemas are composed: a base schema with always-required vars, plus
  driver-conditional sub-schemas that activate based on the driver
  flag.
  - `STORAGE_DRIVER=minio` → requires `MINIO_*` vars.
  - `MAILER_DRIVER=gmail-smtp` → requires `GMAIL_USER`, `GMAIL_APP_PASSWORD`.
  - `LLM_PROVIDER=portkey` → requires `PORTKEY_API_KEY`,
    `PORTKEY_VIRTUAL_KEY`.
  - `LLM_PROVIDER=ollama` → requires `OLLAMA_BASE_URL`.
- **Boot fails loud** if any required var is missing or a value
  doesn't match its schema. The error message lists the missing/invalid
  keys and the active driver flags. No silent defaults for required
  vars.
- Optional vars have explicit defaults declared in the schema; the
  default is the value used, not `undefined`.

### `.env.example` as source of truth

- Lives at the repo root.
- Contains **every** key the app reads, with placeholder values and
  inline comments describing the purpose and any allowed enum values.
- A CI lint (added in Phase 7) compares the keys actually referenced
  in code (`process.env.X` or `config.X`) against `.env.example` and
  fails if any key is referenced but missing from the example file.
- Secrets in `.env.example` are placeholders only (`changeme`,
  `your-key-here`). Real secrets never live in this file.

### Driver-style envs

- `STORAGE_DRIVER` ∈ `local | minio | gcs`.
- `MAILER_DRIVER` ∈ `gmail-smtp | provider-api`.
- `LLM_PROVIDER` ∈ `portkey | ollama` (portkey is prod default).
- Each driver value selects a sub-schema; only the selected
  sub-schema's vars are required. Others may be present but are not
  validated.

### Secret handling

- Local dev: `.env` (gitignored), loaded by Next.js / Nest at
  startup.
- CI: secrets injected as GitHub Actions secrets; mapped to env vars
  in the workflow; same Zod validation applies.
- Production (P7+): secrets via the deploy target's secret manager.
  Initial target is **Docker secrets** for compose deploys; if we
  move to k8s later, sealed-secrets or external-secrets-operator.
  Decision deferred to Phase 7 deploy ADR.
- The `ADMIN_EMAIL` / `ADMIN_PASSWORD` pair is a special case: read
  only on the seed step, never logged.

### Boot diagnostics

- On boot, the API logs only **the keys present** and **the
  resolved drivers** — never values.
- Example log line: `config: loaded 23 keys; drivers: storage=minio,
  mailer=gmail-smtp, llm=portkey`.
- A `--validate-config` CLI flag runs the schema check and exits 0/1
  without starting the server; used in CI image smoke checks.

### Hot reload

- Not supported in P1. Config changes require a process restart.
  Acceptable because changes happen during deploys, not at runtime.

## Consequences

### Positive

- Misconfiguration fails at boot, never at the first user request.
  Drastically shortens "why is this broken in staging?" debugging.
- The Zod schemas double as documentation and as TypeScript types
  (`z.infer`), so consumers get autocompletion + compile-time safety.
- `.env.example` lint guarantees every key in code is documented,
  and every key documented is real.
- Driver sub-schemas keep the burden on the relevant deployment
  shape — a local dev with `STORAGE_DRIVER=local` doesn't need to
  populate MinIO vars.

### Negative / Trade-offs

- Adding a new env var means three steps: schema, `.env.example`,
  consumer. We accept this as the cost of correctness; the lint
  catches the second step if forgotten.
- Conditional schemas via Zod's `discriminatedUnion` are slightly
  more verbose than a single flat schema; readability trade-off
  is worth it.
- No hot reload limits live reconfiguration. Acceptable at this
  scale.

### Neutral

- The `packages/config` package exports a singleton `config`
  object — typed, frozen, the only way to read env in app code. Lint
  rule forbids direct `process.env.*` access outside this package.
- A `getConfig()` test helper builds a config from an in-memory
  object so tests don't touch real env.
- `.env.example` is committed; `.env` and `.env.*.local` are in
  `.gitignore`.

## Alternatives considered

- **dotenv-only**: loads but doesn't validate. Every "undefined" bug
  becomes a runtime mystery.
- **Spring-style environment profiles** (e.g., `application-prod.yml`):
  overkill for a Node project; adds a YAML loader and a profile
  resolver for marginal benefit.
- **Doppler / Vault from day one**: vendor lock-in and operational
  overhead before we have a production environment to manage.
- **Splitting config across multiple packages**: pulls the validation
  surface apart; we want a single place that fails or succeeds.

## Related ADRs / docs

- `CLAUDE.md` §3 (Tech stack), §7 (Configuration & secrets), §10
  (Working agreement — keep `.env.example` in sync).
- ADR-005 (LLM gateway) — `LLM_PROVIDER`, `LLM_PRIMARY_MODEL`,
  `LLM_FALLBACK_MODEL`, `PORTKEY_*`.
- ADR-012 (Observability) — `OTEL_EXPORTER_OTLP_ENDPOINT`.
- ADR-015 (Role model) — `ADMIN_EMAIL`, `ADMIN_PASSWORD` seeding.
- ADR-019 (CI/CD) — env-key lint, secret injection.
