# ADR-006: LLM gateway — Vercel AI SDK behind Portkey

- Status: Accepted
- Date: 2026-05-15
- Deciders: Anshul Jain (project owner)
- Phase introduced: Phase 3 (PlannerAgent); Phase 5 (AssessorAgent)

## Context

The differentiating capability of the platform is per-student AI
assessment, plus instructor-facing planning help. LLM calls happen
across multiple features:

- PlannerAgent (instructor planning).
- AssessorAgent (student assessments).
- Question pool generation.
- Free-form chat helpers.

Operational requirements (`CLAUDE.md` §3, §6):

- Provider-agnostic — never import a vendor SDK in business logic.
- Streaming + tool-calling are first-class.
- Production needs routing, fallback model on error, retries, virtual
  keys (revocable on leak), and observability with trace IDs we can
  pin to our own audit rows.
- Local dev must run against Ollama with no code change — only env.
- Token usage must be measurable per call to drive `LlmUsage` and
  per-bucket budgets (ADR-008).

## Decision

Layer two pieces:

- **Vercel AI SDK** (`ai` package) is the developer-facing API used in
  `packages/llm/aiClient`. It provides idiomatic `generateText`,
  `streamText`, and tool-calling primitives across providers.
- **Portkey** is the production LLM gateway. The AI SDK is configured
  with `baseURL` pointing at the Portkey endpoint; Portkey holds the
  virtual keys, the primary + fallback model routing, retries, and
  observability.

Concrete shape:

- `LLM_PROVIDER=portkey` (default): AI SDK → Portkey → primary model
  with fallback rule.
- `LLM_PROVIDER=ollama`: AI SDK `baseURL` swaps to Ollama's
  OpenAI-compatible endpoint. Same `aiClient` surface; nothing else
  changes. Token tracking still runs (cost = $0).
- A single `aiClient` is exported from `packages/llm`. All agents and
  features import only from this package.
- An ESLint rule (`no-restricted-imports`) bans direct imports of
  `openai`, `@anthropic-ai/sdk`, `@google/generative-ai`, etc.,
  anywhere outside `packages/llm/`.
- Every call records the Portkey trace ID into `LlmUsage.portkey_trace_id`
  and into `AttemptTurn.portkey_trace_id` for assessor turns.

## Consequences

### Positive

- Idiomatic streaming + tool-calling DX from the AI SDK; less glue code
  for the agents.
- Portkey gives prod-grade routing, retries, and a unified observability
  view across providers — capabilities we would otherwise build by hand.
- Virtual keys mean a leaked key is revocable in Portkey without
  rotating the underlying provider key.
- Local dev with Ollama is a one-line env change — fast iteration, no
  cost, useful in CI.
- Trace-ID pinning makes "find this agent turn in Portkey" trivial,
  which matters for assessment audits.

### Negative / Trade-offs

- Two layers to learn (AI SDK + Portkey) and to keep aligned on tool
  schemas and model IDs.
- Some AI SDK provider features may lag Portkey's; we treat the AI SDK
  surface as canonical and skip provider-only features that would
  bypass it.
- Portkey is a managed dependency; outages affect prod calls. Mitigated
  by Portkey's own multi-provider fallback and our queue-and-retry on
  background jobs.

### Neutral

- We do not pin to a specific provider in code; the model ID lives in
  `LLM_PRIMARY_MODEL` / `LLM_FALLBACK_MODEL` env vars and in Portkey
  config.
- Batch/non-interactive jobs (e.g. question generation) use the same
  client but with longer timeouts.

## Alternatives considered

- **Portkey SDK only**: works, but the streaming and tool-calling DX is
  less idiomatic than the Vercel AI SDK and we would write more wrapper
  code.
- **LangChain.js**: heavy, leaky abstractions, frequent breaking
  changes. We reject it for production agents; small parts of it (e.g.
  text splitters for RAG in Phase 8) may be borrowed in isolation.
- **Custom thin wrapper directly over provider SDKs**: maximum control,
  most maintenance — we would reinvent retries, fallback, and
  observability over time.
- **OpenRouter / LiteLLM**: comparable gateways. Portkey was chosen for
  its observability UI, virtual keys, and config-driven fallback rules
  matching our needs.

## Related ADRs / docs

- `CLAUDE.md` §3, §6.
- ADR-007 (agent design) — both agents call `aiClient`.
- ADR-008 (token tracking) — middleware sits in `packages/llm`.
- `docs/development/phase_3.md` — PlannerAgent introduction.
- `docs/development/phase_5.md` — AssessorAgent introduction.
