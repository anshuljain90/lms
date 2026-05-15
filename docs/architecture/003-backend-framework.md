# ADR-003: Backend framework — NestJS with TypeScript

- Status: Accepted
- Date: 2026-05-15
- Deciders: Anshul Jain (project owner)
- Phase introduced: Phase 0

## Context

The backend grows steadily over eight phases:

- Phase 1: auth, users, JWT, email verification.
- Phase 2: courses, batches, enrollment workflow.
- Phase 3: materials, planner LLM endpoints (streaming).
- Phase 4: tasks, discussions.
- Phase 5: AI assessment, AssessorAgent, attempt scoring.
- Phase 6: notifications, PWA push.
- Phase 7: hardening, rate limits, audit logs.
- Phase 8: Keycloak swap-in, RAG endpoints.

Cross-cutting needs:

- Role-based authorization on most endpoints (ADMIN, INSTRUCTOR, STUDENT).
- Request validation on input (Zod or class-validator) — must align with
  the shared schemas.
- OpenAPI generation (`CLAUDE.md` §3 — FE types come from
  `openapi-typescript`).
- LLM token-budget middleware (`CLAUDE.md` §6).
- SSE streaming endpoints for agents.
- Structured logging + OTel traces.
- Test ergonomics with testcontainers for real Postgres.

## Decision

Use **NestJS** (Express adapter) in TypeScript strict mode for
`apps/api`.

- Modules per feature (`AuthModule`, `CoursesModule`, `BatchesModule`,
  `AssessmentModule`, etc.).
- Guards for auth/role enforcement; interceptors for logging, OTel
  spans, and token-budget enforcement.
- DTOs use **`nestjs-zod`** so the shared Zod schemas in
  `packages/shared` drive both runtime validation and Swagger generation.
- Swagger module enabled at `/api/docs` — emits OpenAPI consumed by
  `openapi-typescript` to generate FE types.
- SSE via NestJS's built-in `@Sse()` decorator.
- Prisma client is the only DB access path (see ADR-004).

## Consequences

### Positive

- Structured DI and modules scale cleanly across the eight phases — new
  features land as new modules without rearranging the world.
- Guards and interceptors give a single, declarative place for
  cross-cutting concerns (auth, RBAC, token budgets, tracing).
- Built-in Swagger module produces an OpenAPI document we consume from
  the FE — no hand-written client SDK.
- `nestjs-zod` lets us reuse shared Zod schemas without duplicating
  decorators-vs-Zod definitions.
- Mature testing story: `Test.createTestingModule` + testcontainers fits
  our integration test plan.
- Familiar Express adapter under the hood; we can drop to raw middleware
  when needed.

### Negative / Trade-offs

- More ceremony than Express (modules, providers, decorators) — overkill
  for trivial endpoints, justified at this surface area.
- Decorator-heavy: TypeScript decorators metadata + reflect-metadata is
  another moving part.
- Bundle size and cold start are heavier than Hono/Fastify-only setups;
  fine for our long-running container target.

### Neutral

- Express adapter chosen over Fastify for ecosystem maturity (passport,
  multer, broad middleware compatibility). We can switch to the Fastify
  adapter later if a specific perf need emerges.
- ESM vs CJS: NestJS works with both; we standardize on ESM.

## Alternatives considered

- **Express alone**: too unstructured at this size; we would reinvent
  modules, DI, guards, and OpenAPI generation by hand.
- **Hono**: very lean and fast, but lacks built-in DI, guards, OpenAPI
  generation, and the SSE ergonomics we need; rebuilding those would
  cost more than NestJS's overhead.
- **Fastify (alone)**: faster than Express but same lack of structure.
- **FastAPI (Python)**: excellent OpenAPI and validation story, but
  splits the monorepo language. We would lose the shared Zod schemas
  and pay an integration cost. The Python LLM ecosystem advantage is
  blunted because the Vercel AI SDK (ADR-006) already covers our needs.
- **tRPC**: great DX inside an all-TypeScript monorepo, but the public
  catalog and any future non-TS clients (mobile native, partner
  integrations) prefer a stable OpenAPI surface.

## Related ADRs / docs

- `CLAUDE.md` §3, §6, §9.
- ADR-001 (monorepo) — `apps/api` is a workspace member.
- ADR-004 (database & ORM) — Prisma is the only DB path here.
- ADR-005 (auth) — guards live in this app.
- ADR-006 (LLM gateway) — services here call `packages/llm`.
- ADR-008 (token tracking) — middleware/interceptor lives here.
