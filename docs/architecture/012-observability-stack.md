# ADR-012: Observability stack — OTel + LGTM + Portkey

- Status: Accepted
- Date: 2026-05-15
- Deciders: Anshul Jain (project owner)
- Phase introduced: Phase 1 (foundational); LLM dashboards ride on Portkey from Phase 3

## Context

We need to debug a system whose behaviour is partly non-deterministic
(LLM agents) from day one. Three signal types matter:

- **Logs** — structured, queryable; especially around auth, enrollment
  decisions, and agent tool calls.
- **Traces** — end-to-end request flow including DB queries and outgoing
  LLM calls. Critical for assessor debugging where one student message
  might fan out into multiple tool calls and DB writes.
- **Metrics** — request rate, error rate, latency percentiles, LLM
  token usage and cost.

Constraints:

- Self-hostable on a single VM via `docker compose`.
- Must accept OpenTelemetry natively so the same SDK exports all three
  signals from `apps/api` and `apps/web`.
- LLM-specific observability (per-call cost, prompt, fallback hits)
  has to land somewhere already wired, not in a parallel stack.
- Solo developer — fewer moving parts wins.

## Decision

- Instrument both apps with the **OpenTelemetry SDK** (Node for the
  API, Web SDK for the FE).
- Send everything to a local **OTel Collector** which fans out:
  - logs → **Loki**
  - traces → **Tempo**
  - metrics → **Prometheus**
- Visualize all three in **Grafana**, with provisioned dashboards in
  `infra/grafana/`.
- Use **Portkey UI** as the dedicated LLM observability surface
  (per-call traces, prompt diffs, fallback events, cost). Portkey is
  already mandatory in the gateway path (CLAUDE.md §6), so this is
  free signal.
- Auto-instrument: HTTP server + client, Prisma, `pg` driver, outgoing
  `fetch` (which captures Portkey/LLM calls).
- Custom spans for the high-value flows:
  - `PlannerAgent.turn` and each tool call within it.
  - `AssessorAgent.turn` and each tool call within it.
  - `scoreAnswer` (with rubric size + answer length attributes).
  - `proposeQuestionPool` (with `n` and `difficulty_mix`).
  - DB transactions in enrollment approval and attempt finalize.
- Every span carries `user_id`, `role`, and (when present)
  `portkey_trace_id` so Grafana traces and Portkey traces can be
  cross-linked.

## Consequences

### Positive

- LGTM stack runs in ~5 containers — measurably lighter than a
  comparable ELK/OpenSearch deployment.
- Single SDK and single export pipeline for logs, traces, and metrics.
- Switching backends later (e.g., to a managed vendor) is a Collector
  config change, not a code change.
- LLM cost and latency are visible without us building a dashboard:
  Portkey already shows per-virtual-key spend, per-model latency, and
  fallback counts.

### Negative / Trade-offs

- Loki's label-based indexing requires discipline (high-cardinality
  fields like `user_id` go in log body, not labels).
- Grafana dashboards committed in-repo need to be kept in sync with
  metric/label changes; we'll PR-review them like code.
- Portkey UI is a SaaS surface (or self-hosted later) — observability
  for LLM spend is split across two tools. Acceptable because the
  cross-link via `portkey_trace_id` is cheap.

### Neutral

- A starter Grafana dashboard ships in `infra/grafana/dashboards/`
  with HTTP RED + LLM tokens-per-feature + Postgres connections.
- Datasources are provisioned via YAML so a fresh `docker compose up`
  brings dashboards online with no clicks.
- No paging/alerting routes wired in P1; Grafana alert rules added
  in Phase 7.

## Alternatives considered

- **ELK / OpenSearch from day 1**: heavier (Elasticsearch alone wants
  more memory than the entire LGTM stack); deferred to Phase 8 if log
  analytics needs grow past Loki's comfort zone.
- **Managed SaaS (Datadog, Honeycomb, New Relic)**: excellent UX, but
  a recurring cost and an external dependency at a stage where we want
  full local reproducibility.
- **Logs-only (pino + file)**: cheapest, but blind on traces, which is
  exactly where LLM agent bugs hide.
- **Langsmith** (or similar) for LLM observability: would duplicate
  what Portkey already gives us, and route calls through a second
  vendor. Rejected for now.

## Related ADRs / docs

- `CLAUDE.md` §3 (Tech stack — Observability row), §6 (LLM strategy).
- ADR-005 (LLM gateway) — Portkey is the source of LLM traces.
- ADR-013 (Testing strategy) — integration tests assert that key
  spans are emitted.
- ADR-020 (Configuration) — `OTEL_EXPORTER_OTLP_ENDPOINT` and related
  vars validated at boot.
- `infra/otel-collector.yaml`, `infra/grafana/`.
