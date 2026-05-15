# ADR-001: Monorepo structure with pnpm workspaces and Turborepo

- Status: Accepted
- Date: 2026-05-15
- Deciders: Anshul Jain (project owner)
- Phase introduced: Phase 0

## Context

This LMS spans a Next.js frontend, a NestJS backend, and several
cross-cutting libraries (LLM client + agents, storage, mailer,
observability, config, shared Zod schemas). All of them are TypeScript
and most must share types across the FE/BE boundary (notably the
Zod schemas in `packages/shared`).

Constraints:

- Single language (TypeScript) across apps and packages.
- Shared types must round-trip without publishing to npm.
- Test, lint, and build must be cacheable so CI (Phase 7) and local dev
  stay fast as the repo grows through eight phases.
- One developer + occasional contributors — tooling overhead must be low.
- Docker Compose drives local dev; the workspace must be friendly to
  containerized builds.

See `CLAUDE.md` §3 (Tech stack) and §4 (Repository layout).

## Decision

Use **pnpm workspaces** for dependency management and **Turborepo** for
the task pipeline.

- `pnpm-workspace.yaml` lists `apps/*` and `packages/*`.
- `turbo.json` defines the pipeline: `build`, `lint`, `test`, `typecheck`,
  `dev`, with explicit `dependsOn` between packages and apps.
- Shared internal packages are referenced by name with `workspace:*`.
- Node version pinned via `.nvmrc` and `packageManager` in
  `package.json`.

## Consequences

### Positive

- Content-addressable pnpm store: minimal disk usage and fast installs
  even with many packages.
- Strict dependency hoisting by default: a package cannot import a
  transitive dep it did not declare. Catches drift early.
- Turborepo caches outputs of `build`, `test`, and `lint` per package;
  re-running an unchanged package is a no-op.
- Parallel pipeline runs across packages out of the box.
- Native support for `workspace:*` so internal packages stay versionless
  during development.
- Trivial path for CI remote cache later (Phase 7) without rewrites.

### Negative / Trade-offs

- pnpm's strict resolution occasionally surfaces peer-dependency warnings
  that yarn/npm would silently allow; need a small `pnpm` override
  block in the root `package.json` for known offenders.
- Turborepo introduces a second tool to learn alongside pnpm scripts.
- Some tools (older bundlers, certain Jest plugins) assume a flat
  `node_modules`; we mitigate by sticking to Vitest and modern toolchain
  choices already in the stack.

### Neutral

- Lockfile (`pnpm-lock.yaml`) is the single source of truth; committed.
- Turbo's daemon mode is enabled locally for speed; disabled in CI.

## Alternatives considered

- **Nx**: more opinionated, includes generators and a graph UI, but
  heavier and steers you toward Nx-specific project boundaries. Overkill
  for the shape of this repo.
- **Bun workspaces**: promising performance, but the runtime + package
  manager combination is too young to bet a multi-phase production
  project on.
- **npm or Yarn (classic) workspaces**: slower installs, no
  content-addressable store, weaker isolation; nothing they offer
  outweighs pnpm's strictness and speed.
- **Yarn Berry (PnP)**: works, but PnP's editor and tooling integrations
  are still occasionally rough; not worth the marginal benefit.
- **Polyrepo**: forces npm publishing or git submodules to share Zod
  schemas across FE/BE — a tax we would pay every day.

## Related ADRs / docs

- `CLAUDE.md` §3, §4.
- ADR-002 (frontend framework) — `apps/web` is a workspace member.
- ADR-003 (backend framework) — `apps/api` is a workspace member.
- `docs/development/phase_0.md` — bootstraps this layout.
