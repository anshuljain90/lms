# ADR-002: Frontend framework — Next.js 15 (App Router) with Tailwind and shadcn/ui

- Status: Accepted
- Date: 2026-05-15
- Deciders: Anshul Jain (project owner)
- Phase introduced: Phase 0 (scaffolding); Phase 2 (public catalog uses SSR)

## Context

The frontend has three audiences with overlapping needs:

- **Public visitors** browsing the course catalog (Phase 2). Needs SEO,
  fast first paint, social previews.
- **Instructors** authoring courses, batches, sessions, materials, tasks
  and reviewing AI-generated content (Phases 2–5). Needs rich forms and
  streaming LLM UIs.
- **Students** taking AI assessments where the AssessorAgent streams its
  reply over SSE (Phase 5). Needs robust streaming and a PWA story for
  mobile use.

Constraints from `CLAUDE.md`:

- TypeScript strict mode.
- Mobile is PWA-first (manifest + service worker), Capacitor wrapper later.
- Time zones rendered in user's TZ on the FE; UTC over the wire.
- Streaming responses from the backend are SSE.

## Decision

Use **Next.js 15 with the App Router**. Server Components by default;
client components opt in with `"use client"`.

Companion choices:

- **Styling**: Tailwind CSS.
- **UI primitives**: shadcn/ui (copy-pasted into `apps/web/components/ui`,
  not a dependency). Themed via CSS variables.
- **Server-state on the client**: TanStack Query for data fetching from
  client components; Server Components fetch directly via `fetch` with
  Next.js caching primitives.
- **Local UI state**: zustand for the small handful of cross-component
  client stores.
- **Forms**: react-hook-form + Zod (using the shared schemas from
  `packages/shared`).
- **Streaming**: native `EventSource` for SSE consumption from the
  AssessorAgent endpoints; Vercel AI SDK's `useChat`-style hooks where
  the data shape lines up.

## Consequences

### Positive

- Public catalog renders on the server: good SEO, low time-to-content.
- Server Components reduce JS shipped on dashboard-heavy pages.
- Streaming is first-class — fits the AssessorAgent's chat UX.
- Route-level code splitting and image optimization out of the box.
- shadcn/ui keeps primitives in our repo: trivial to fork, theme, audit,
  and patch — no opaque vendor lock-in.
- Tailwind + Zod + react-hook-form + shadcn is a well-trodden stack
  with abundant community examples, accelerating Phase 2/3/4 build-out.
- TanStack Query handles caching, retries, and request deduplication for
  the dashboard surfaces driven by the REST API.

### Negative / Trade-offs

- App Router has a steeper mental model than the Pages Router (Server
  vs Client component split, caching directives, dynamic vs static).
- shadcn/ui requires curation: any update is a manual re-copy, not an
  npm bump. We accept this in exchange for control.
- Service Components + TanStack Query mix needs discipline: pick one per
  surface to avoid double-fetching.

### Neutral

- We deploy with `next start` behind Caddy (Phase 7) rather than Vercel,
  so SSR runs in our own container.
- No CSS-in-JS runtime; Tailwind handles everything statically.

## Alternatives considered

- **Vite + React SPA**: simpler dev loop but loses SSR, hurting SEO on
  the public catalog and complicating the streaming story.
- **Remix**: comparable App Router story, smaller ecosystem and fewer
  shadcn/ecosystem examples.
- **SvelteKit / Nuxt / Solid Start**: would force a TS-non-React stack;
  team and ecosystem reasons keep us on React.
- **Pages Router (Next.js 14)**: stable but the App Router's streaming
  and Server Component story matches our LLM use case better.
- **MUI / Chakra UI / Mantine**: complete component libraries, but
  harder to theme deeply and add weight we do not need; shadcn covers
  primitives we extend.

## Related ADRs / docs

- `CLAUDE.md` §3, §6 (LLM streaming via SSE).
- ADR-003 (backend framework) — provides the OpenAPI we type against.
- ADR-006 (LLM gateway) — the AI SDK ships React hooks consumed here.
- ADR-007 (agent design) — the AssessorAgent's streaming UX lives here.
- `docs/development/phase_2.md` — public catalog SSR.
- `docs/development/phase_5.md` — AI assessment UI.
