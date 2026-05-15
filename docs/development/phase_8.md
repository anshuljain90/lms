# Phase 8 — Future (Keycloak, RAG, app stores, payments, ELK)

## Goal

A parking lot for capabilities that are intentionally deferred past the
shippable MVP. Each item lists a trigger condition (when to actually
build it), the data model deltas it would introduce, and the notable
risks. This document is **not** a commitment to build any of these in
order; it exists so we have a coherent story when one of the triggers
fires and to prevent earlier phases from accidentally locking out the
upgrade path.

---

## Scope (in)

A short entry per future capability. None of these are scheduled until
their trigger is met.

## Scope (out / deferred)

Implementation of any item in this list. This phase is documentation
only.

## Prerequisites

- All of P0–P7 complete and operational.

## Deliverables

- This document.
- Per-item ADR drafts may be added when an item is unparked, e.g.
  `docs/architecture/021-keycloak-migration.md`.

---

## 8.1 Keycloak + Google OAuth (replaces Phase 1 JWT impl)

- **Trigger condition.** Either: (a) we need SSO with a customer
  organization that brings its own IdP, or (b) self-rolled JWT
  rotation / password reset / 2FA upkeep becomes a maintenance
  burden, or (c) we need OAuth login (Google) without writing the
  flow ourselves.
- **Data model deltas.**
  - Drop `User.password_hash`; add `User.external_idp_subject`
    (unique) plus optional `User.external_idp_provider`.
  - Add `IdentityLink (id, user_id, provider, subject, linked_at)`
    for users that link multiple providers (e.g. Google + email).
  - Retain `User.email` as the canonical join key during migration;
    Keycloak realm seeded with a one-shot dump of existing users so
    the same emails resolve.
  - Phase 1's refresh-token tables are removed; Keycloak issues
    tokens.
- **Notable risks.**
  - Migration cutover: existing sessions invalidated; communicate +
    provide a forced password reset path during the transition.
  - Same `/auth` API contract from FE per CLAUDE.md §3 — adapter on
    the BE side translates Keycloak's responses to the existing
    shape so the FE does not change.
  - Operational complexity: now we run Keycloak (HA Postgres-backed
    if we go further than single instance).
- **Notes.** Use a Keycloak realm per environment. Do not put admin
  console on the public domain.

---

## 8.2 RAG with pgvector

- **Trigger condition.** Instructors report the PlannerAgent and
  AssessorAgent give generic outputs that ignore course-specific
  materials. Symptoms: instructors editing >50% of generated
  questions, students complaining the assessor is unaware of class
  content.
- **Data model deltas.**
  - Add `pgvector` extension to Postgres.
  - `MaterialChunk (id, material_id, chunk_idx, text, embedding
    vector(768), tokens, created_at)`.
  - `TopicChunk (id, topic_id, chunk_idx, text, embedding vector(768),
    tokens, created_at)`.
  - `ChatSnippet (id, owner_user_id, source_kind, source_id, text,
    embedding vector(768), created_at)` — for instructor-saved
    snippets from prior LLM chats.
  - HNSW or IVFFlat index on each `embedding` column.
- **Notable risks.**
  - Embedding cost at scale; route embeddings through Portkey
    too (CLAUDE.md §6). Default model `nomic-embed-text` via Ollama
    keeps local dev free.
  - Stale embeddings when materials are edited — needs an
    invalidate-and-recompute hook on `Material.updated_at`.
  - Context-window blowups; cap retrieved chunks per agent call and
    rely on the existing token budget middleware.
- **Notes.** The retrieval call lives in `packages/llm/rag/` and is
  invoked by both agents through a shared `withRetrievedContext()`
  helper, preserving the provider-agnostic discipline.

---

## 8.3 Capacitor mobile wrapper

- **Trigger condition.** Either: (a) push reliability on iOS PWA
  proves unworkable, or (b) we want listing in Apple App Store /
  Google Play for credibility, or (c) instructors request specific
  native features (camera capture for handwritten work, file picker
  on iOS, native share-sheet integration).
- **Data model deltas.** None directly. Possibly:
  - `DeviceRegistration (id, user_id, platform[IOS|ANDROID],
    push_token, app_version, created_at, last_seen_at)` — if we move
    from Web Push to native push (APNs / FCM).
- **Notable risks.**
  - App store review is gatekept by Apple/Google; expect 1–2 rejection
    cycles. Plan for in-app purchase compliance even if we ship no
    paid features (Apple's rules around promoting external payment
    are strict).
  - Two release pipelines (PWA + binary) — versioning discipline
    matters.
- **Notes.** Capacitor lets the PWA reuse 100% of `apps/web` while
  surfacing native APIs via plugins. Splash, icons, deep links are
  one-time setup per platform.

---

## 8.4 Payment receipts on enrollment

- **Trigger condition.** Instructors start running paid cohorts and
  need evidence that a student has paid before approving enrollment.
  Explicitly: this is **receipt evidence** only; the platform will
  not process payments.
- **Data model deltas.**
  - Add columns to `EnrollmentRequest`:
    `payment_receipt_file_ref?`, `payment_amount_cents?`,
    `payment_currency?`, `payment_paid_at?`,
    `payment_status[NONE|UPLOADED|VERIFIED|REJECTED]`.
  - Or, less invasively, a separate `PaymentReceipt (id,
    enrollment_request_id, file_ref, amount_cents, currency,
    paid_at, status, verified_by_user_id?, verified_at?)`.
- **Notable risks.**
  - Storing financial documents shifts compliance posture (regional
    receipt-keeping rules, PII handling). Get a brief legal review
    before enabling.
  - Forged receipts; the instructor still has to verify manually.
    Document this clearly so we are not selling "verified
    payments" we cannot actually verify.
- **Notes.** Use the existing `IStorage` for receipts; never put
  card data anywhere. If we ever do real payments later, that is a
  separate phase with PCI scope.

---

## 8.5 ELK / OpenSearch + Kibana

- **Trigger condition.** Loki + Grafana (CLAUDE.md §3) become
  insufficient for log analytics, typically when:
  - Daily event volume exceeds ~50k structured events/day, or
  - We need full-text search across multi-day windows that Loki's
    label model can't serve cheaply, or
  - Stakeholders want pre-built Kibana dashboards.
- **Data model deltas.** None in Postgres. New infra:
  - Filebeat / Logstash sidecar in `infra/docker-compose.yml` to
    ship logs to ES/OpenSearch.
  - Index lifecycle policy (hot 7d, warm 30d, cold 365d).
- **Notable risks.**
  - OpenSearch is heavy to operate (RAM, disk).
  - Two log destinations during migration (Loki and OpenSearch) is
    operationally noisy; pick one and commit.
- **Notes.** Prefer OpenSearch over Elasticsearch for licensing
  clarity. Keep traces in Tempo and metrics in Prometheus regardless.

---

## 8.6 Marketplace v0

- **Trigger condition.** We have ≥10 instructors and prospective
  students start asking "how do I find a good instructor for X?".
- **Data model deltas.**
  - `InstructorProfile.public = true` (default false).
  - `InstructorRating (id, instructor_user_id, student_user_id,
    batch_id, stars[1..5], comment?, created_at)`.
  - Materialised view or rollup: `InstructorAggregateRating
    (instructor_user_id, avg_stars, count, last_updated_at)`.
  - Search index over `InstructorProfile.headline`, `bio_long`,
    `Course.title`, `Course.description`, `Topic.title`. Postgres
    full-text search is enough; if RAG (8.2) is already in, reuse
    pgvector for semantic search by topic.
- **Notable risks.**
  - Ratings invite gaming and brigading. Restrict ratings to
    students with at least one `Enrollment` row in the instructor's
    batches.
  - Public instructor pages mean PII surfaces (name, photo). Provide
    explicit opt-in.
- **Notes.** Build instructor profile pages first; ratings second;
  search third. Each is independently shippable.

---

## 8.7 Discussion forums per course (not per task/session)

- **Trigger condition.** Students or instructors ask for a place to
  discuss a course independent of any single task or session — e.g.
  course-wide announcements, FAQs, post-course alumni chat.
- **Data model deltas.**
  - Extend `DiscussionScope` enum with `COURSE` and `BATCH` values.
  - Add `Discussion.course_id?` and `Discussion.batch_id?` columns.
  - Service-layer constraint: exactly one of
    `(task_id, session_id, course_id, batch_id)` is non-null.
- **Notable risks.**
  - Without moderation tooling, course-wide discussions degrade
    quickly. Plan basic moderation (pin, lock, soft-delete) before
    enabling.
  - Notification fan-out grows; a single post on a popular course
    can trigger hundreds of `DISCUSSION_REPLY` notifications. Add a
    digest mode (P6's notification fatigue note applies double here).
- **Notes.** This is the smallest item in this list and is a good
  candidate to bundle with another change rather than ship alone.

---

## Configuration (new env vars)

None at parking-lot stage. Each item, when unparked, will introduce
its own env vars (Keycloak realm + client, pgvector model name,
Capacitor app IDs, OpenSearch endpoint, etc.) and they go through the
standard `packages/config` Zod validation.

## Testing plan

Per-item, when unparked. As a rule, no Phase 8 item ships without:

- Unit tests for new schemas and services.
- Integration tests via testcontainers where infra changes (e.g.
  pgvector, Keycloak).
- At least one E2E flow exercising the user-visible behaviour.

## Exit criteria (checklist)

- [ ] This document exists and is reviewed.
- [ ] Each item has a clear trigger condition that the team agrees on.
- [ ] Earlier phases (P0–P7) have not introduced lock-in that
      prevents any of these items (re-check during P7 sign-off).

## Risks & open questions

- **Risk of building too early.** The whole point of this phase is to
  resist building these before triggers fire. Revisit quarterly; do
  not add items casually.
- **Risk of building too late.** Some items (Keycloak, RAG) take
  weeks. When a trigger fires, allocate time honestly rather than
  half-shipping.
- **Lock-in checks.** Confirm during P7 review:
  - The auth surface in `/auth` is still adapter-friendly for
    Keycloak.
  - `packages/llm/` exposes a hook for retrieved context so RAG can
    slot in without rewriting agents.
  - PWA shell can be wrapped by Capacitor without code changes.
  - `IStorage` already handles arbitrary files (receipts).
  - Logging is structured JSON so an OpenSearch shipper can ingest
    it without re-parsing.
