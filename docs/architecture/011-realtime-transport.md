# ADR-011: Realtime transport — SSE for streaming, REST for state

- Status: Accepted
- Date: 2026-05-15
- Deciders: Anshul Jain (project owner)
- Phase introduced: Phase 3 (planner streaming), broadened in Phase 5 (assessor)

## Context

The platform needs two different network shapes:

- Streaming token output from LLM calls (PlannerAgent in Phase 3, the
  AssessorAgent message stream in Phase 5). These are one-way, server →
  client, frequent small frames.
- Everything else: CRUD on courses, batches, materials, tasks,
  enrollments, submissions, listings, search. Plain request/response.

Constraints carried in from `CLAUDE.md`:

- PWA-first frontend (ADR-018). Whatever transport we pick has to work
  inside iOS/Android browsers and a future Capacitor webview.
- We don't need bidirectional presence in P1–P7. The only "live" thing
  is the AssessorAgent stream and instructor planning.
- Infra should stay simple — no Redis pub/sub adapter, no sticky
  sessions if avoidable.
- Auth uses short-lived JWT access tokens with HTTP-only refresh
  cookies (CLAUDE.md §3).

## Decision

- **Server-Sent Events (SSE)** for LLM token streams and any other
  one-way server → client push (assessor turns, planner outputs).
- **REST** (HTTP/JSON, OpenAPI-described) for all state mutations and
  listings.
- **WebSockets** are deferred. Re-evaluate only when we need
  bidirectional presence (typing indicators, multi-cursor editing,
  live cohort dashboards).

Implementation:

- API side uses Nest's `@Sse()` decorator returning
  `Observable<MessageEvent>`. The underlying response sets
  `Content-Type: text/event-stream` and disables compression on that
  route.
- FE consumes via the `@microsoft/fetch-event-source` polyfill so we
  can attach the `Authorization: Bearer <accessToken>` header (the
  native `EventSource` cannot set headers).
- Each event carries a structured payload: `{ type, data, traceId }`.
  `type` discriminates `token`, `tool_call`, `tool_result`,
  `done`, `error`.
- Token refresh during a long stream: client opens a new SSE with the
  refreshed token; server resumes from the last `turn_idx`.
- Backpressure: streams are bounded by the agent turn budget
  (CLAUDE.md §6). No server-side queueing beyond that.

## Consequences

### Positive

- Native browser support; no socket library required client-side
  beyond a small fetch polyfill.
- Auto-reconnect semantics built into SSE; works through most proxies
  and CDNs that allow long-lived HTTP responses.
- Stateless: any API replica can serve any stream. No sticky sessions
  or cross-node pub/sub needed.
- Same auth path as REST (Bearer header), no separate ticketing flow.
- Plays well with Portkey and the Vercel AI SDK, both of which already
  expose token streams as async iterators we can pipe into the SSE
  observable.

### Negative / Trade-offs

- One-way only. If a future feature needs server ↔ client, we'll add
  WebSockets alongside (not replacing) SSE for that surface.
- Some corporate proxies buffer responses and break streaming; we
  document a `?stream=poll` fallback that the FE can opt into for the
  rare environment that strips it.
- Native `EventSource` can't set headers, so we depend on
  `fetch-event-source`. Acceptable — it's small, well-maintained, and
  a common pattern.

### Neutral

- We add a small `useSse(url, options)` hook in `apps/web` and a
  matching helper in `packages/shared` for typed event payloads.
- SSE endpoints are documented in OpenAPI as `text/event-stream`
  responses with the event payload schema referenced.

## Alternatives considered

- **WebSockets (Socket.IO or ws)**: bidirectional and battle-tested,
  but heavier — needs a separate auth handshake, often a sticky-session
  load balancer, and a Redis adapter for multi-replica fanout. We
  don't need any of that yet.
- **Long polling**: trivially simple but bad UX for token streams
  (perceived latency, wasted requests).
- **HTTP/2 server push**: deprecated in Chrome and never broadly
  adopted; not a serious option.
- **gRPC streaming**: powerful but mismatched with a browser frontend
  and our OpenAPI-first contract.

## Related ADRs / docs

- `CLAUDE.md` §3 (Tech stack — Realtime row), §6 (LLM strategy).
- ADR-003 (NestJS) — `@Sse()` decorator availability.
- ADR-005 (LLM gateway) — token streams originate from the Vercel AI
  SDK behind Portkey.
- ADR-017 (Question pool & scoring) — assessor message stream consumer.
- `docs/development/phase_3.md` and `docs/development/phase_5.md` —
  flows that exercise SSE.
