# ADR-009: Storage abstraction — `IStorage` with LocalFS, MinIO, GCS

- Status: Accepted
- Date: 2026-05-15
- Deciders: Anshul Jain (project owner)
- Phase introduced: Phase 0 (interface + LocalFs); Phase 3 (MinIO/GCS)

## Context

The platform stores binary assets:

- Course/batch/task materials uploaded by instructors.
- LLM-chat-exported markdown (treated as a material, with optional
  attached files).
- User avatars.
- Future: assessment attachments (Phase 5+).

Operational constraints:

- Local dev should not require cloud credentials.
- Production target is undecided in P1 — could be self-hosted MinIO or
  managed GCS.
- The BE must be able to issue **presigned URLs** so the FE can upload
  large files directly without proxying through the API.
- Large files must not bloat the database.

## Decision

Define `IStorage` in `packages/storage` and provide three
implementations selected at boot via `STORAGE_DRIVER`.

```ts
interface IStorage {
  put(
    path: string,
    data: Buffer | Readable,
    opts?: { contentType?: string; cacheControl?: string }
  ): Promise<{ path: string; size: number; etag?: string }>;

  get(path: string): Promise<Readable>;

  delete(path: string): Promise<void>;

  presignGetUrl(path: string, ttlSeconds: number): Promise<string>;
  presignPutUrl(
    path: string,
    ttlSeconds: number,
    contentType: string
  ): Promise<string>;

  head(path: string): Promise<{ exists: boolean; size?: number; contentType?: string }>;
}
```

Implementations:

- `LocalFsStorage`: writes under `STORAGE_LOCAL_ROOT`. `presignGetUrl`
  / `presignPutUrl` return URLs to a small NestJS controller (e.g.
  `/api/_storage/local/...?token=...`) where `token` is a signed JWT
  carrying `path`, `op` (`get|put`), `exp`, and `contentType`. The
  controller verifies the token and streams the file or accepts the
  upload. This makes local dev behave like a presigned-URL flow.
- `MinioStorage`: AWS S3 SDK pointed at `MINIO_ENDPOINT` with
  path-style addressing. Native presigned URL support.
- `GcsStorage`: `@google-cloud/storage` with native V4 signed URLs.

Selection: `STORAGE_DRIVER` ∈ `local | minio | gcs`. Validated at boot
by `packages/config`.

Path conventions:

- `materials/{courseId|batchId}/{materialId}/{filename}`
- `avatars/{userId}/{hash}`
- All paths are content-addressed-ish (random ID + original filename)
  to avoid collisions; original filename retained for download.

## Consequences

### Positive

- Single business-logic surface for storage; swapping backends is a
  config change.
- Presigned URLs (real or simulated) keep large uploads off the API
  request path.
- Local dev works with zero credentials — `LocalFsStorage` is the
  default in `.env.example`.
- MinIO in dev compose mirrors prod-like S3 semantics for engineers
  who want to test the S3 path locally.

### Negative / Trade-offs

- Presigned URL semantics differ slightly across backends:
  - LocalFs proxies through API → counts against API request budget,
    not external bandwidth. Acceptable for dev.
  - MinIO/GCS issue real presigned URLs — uploads land directly on
    object storage.
  - This is a **documented limitation** and tests assert the contract
    (signed URL works for `ttlSeconds`, expires after).
- `etag` is optional in the interface because LocalFs cannot produce a
  meaningful ETag without computing a hash; MinIO/GCS provide it
  natively.
- Three implementations means three integration test paths.

### Neutral

- We do not abstract bucket creation; that is a deploy-time concern.
- We do not enforce server-side encryption in the interface; it is set
  via backend-specific config (SSE in MinIO, default encryption in GCS).
- Lifecycle policies (e.g. delete avatars older than X) are managed
  outside the abstraction.

## Alternatives considered

- **Hard-bind to S3 / MinIO only**: simpler, but local dev has to run
  MinIO from day one — extra container, extra credentials. The
  `LocalFsStorage` path is cheaper for new contributors.
- **Store binaries as `bytea` in Postgres**: bloats the DB, breaks at
  larger files, complicates backups, no presigned URL story.
- **Direct cloud SDK calls in business logic**: violates the
  provider-agnostic discipline (`CLAUDE.md` §3).
- **A SaaS file service (Uploadcare, Filestack)**: external dep, cost,
  vendor lock — unnecessary for this scope.

## Related ADRs / docs

- `CLAUDE.md` §3, §7.
- ADR-003 (backend) — the LocalFs presigned-URL controller lives here.
- ADR-010 (mailer) — same abstraction pattern.
- `docs/development/phase_0.md` — interface + LocalFs scaffolding.
- `docs/development/phase_3.md` — materials upload flow.
