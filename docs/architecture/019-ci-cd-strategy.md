# ADR-019: CI/CD strategy — local-only in P1–P6, GitHub Actions in P7, GHCR for images

- Status: Accepted
- Date: 2026-05-15
- Deciders: Anshul Jain (project owner)
- Phase introduced: Local checks from Phase 0; GitHub Actions in Phase 7

## Context

The project ships in eight phases (CLAUDE.md §8). The earliest phases
are foundational scaffolding where the codebase shape is still moving;
heavy CI investment up front would mean rewriting workflows every
phase. By Phase 7 the surface is stable enough that branch protection
and required checks pay for themselves.

Constraints:

- Solo developer in build phase; CI work competes directly with
  feature work.
- Trunk-based with squash merges (CLAUDE.md §9).
- Tests already run locally via Vitest + testcontainers (ADR-013).
- Container images need a registry that's free for both public and
  private repos and uses GitHub-native auth.

## Decision

### Phase 1 through Phase 6: local-only

- The `package.json` exposes Turborepo-orchestrated tasks:
  - `pnpm lint` — ESLint across all packages.
  - `pnpm typecheck` — `tsc --noEmit` per package.
  - `pnpm test` — Vitest unit + integration (testcontainers spins up
    Postgres on demand).
  - `pnpm e2e` — Playwright against `docker compose up --wait`.
- Pre-commit hook (Husky + lint-staged) runs lint + typecheck on
  changed files. **Hooks are not bypassed**; if they fail, fix the
  cause.
- `docker compose up -d --wait` validates the full stack boots; this
  is the ground-truth integration check while CI is absent.

### Phase 7: GitHub Actions

Three workflows:

1. **`.github/workflows/ci.yml`** — triggers on PRs.
   - Steps: checkout → setup pnpm + Node 22 → install (with cache) →
     `pnpm lint` → `pnpm typecheck` → `pnpm test` (unit + integration
     via testcontainers in the runner's Docker) → `pnpm e2e:headless`
     against the compose stack.
   - Matrix: Node 22 only (single supported version; broaden later
     if needed).
   - Coverage report posted as a PR comment (ADR-013).

2. **`.github/workflows/build-images.yml`** — triggers on pushes to
   `main` and on tag pushes.
   - Builds `apps/api` and `apps/web` Docker images with multi-stage
     Dockerfiles.
   - Pushes to **GHCR** (`ghcr.io/<owner>/lms-api`,
     `ghcr.io/<owner>/lms-web`).
   - Image tags: `sha-<short>`, `branch-main`, and on tags the
     semver (`v1.2.3`) plus `latest`.
   - Auth via the workflow's `GITHUB_TOKEN` (no extra secret
     management).

3. **`.github/workflows/release.yml`** — triggers on `v*` tag.
   - Generates release notes from conventional commits between the
     current tag and the previous one.
   - Attaches the built image references as release assets.

### Branch protection (Phase 7)

- `main` is protected:
  - Requires `ci.yml` green.
  - Requires at least 1 review (self-review counts during solo phase
    but PR-first discipline stays).
  - Squash merge only; linear history enforced.
  - No force-push; no deletion.

### Why this split

- Phase 1–6 work changes folder layout, packages, and tooling
  frequently. Locking in CI early would mean reworking it on each
  phase.
- By Phase 7 the layout is stable, integration tests exist, and a
  second contributor is plausible — exactly when CI pays off.

## Consequences

### Positive

- Zero CI maintenance during the most volatile phases.
- When CI lands, it's wrapping a test suite that already passes
  locally — first green build is realistic on day one of P7.
- GHCR with `GITHUB_TOKEN` means no extra secret rotation and works
  the same for private repos.
- Conventional commits (CLAUDE.md §9) make automated release notes
  free.

### Negative / Trade-offs

- Until P7, "it works on my machine" is the integration story. We
  mitigate with the `docker compose up --wait` discipline before
  every phase exit.
- Without CI, secrets like `.env` could be committed by mistake —
  Husky pre-commit runs `gitleaks` (or equivalent) to catch this.
- A second contributor before P7 would force CI earlier; acceptable
  cost.

### Neutral

- Turborepo cache is local-only in P1–P6; remote cache (S3/GHA) can
  be added with the P7 workflows without code changes.
- E2E artifacts (Playwright traces) are uploaded only on failure to
  keep storage costs sane.
- The same Dockerfiles used in CI image builds are the ones used in
  `docker-compose.yml`, so we don't maintain two definitions.

## Alternatives considered

- **GitHub Actions from Phase 1**: extra config maintenance during
  the most-churning phases; minimal benefit until tests and shape
  exist.
- **CircleCI / GitLab CI**: introduces a vendor account separate from
  GitHub for no additional capability we need.
- **Husky-only, no CI ever**: bypassable; doesn't enforce on PRs from
  collaborators; doesn't gate merges.
- **Docker Hub for images**: rate limits for unauthenticated pulls
  and an extra account; GHCR is co-located with the source.
- **Self-hosted runners**: not warranted at this scale; would replace
  one ops surface with another.

## Related ADRs / docs

- `CLAUDE.md` §3 (Tech stack — CI/CD row), §9 (Conventions — Git).
- ADR-013 (Testing strategy) — what CI runs.
- ADR-020 (Configuration) — env validation as part of CI.
- `docs/development/phase_7.md` — workflows land, branch protection
  configured, image build pipeline live.
