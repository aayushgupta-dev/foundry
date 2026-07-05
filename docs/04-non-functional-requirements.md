# 04 — Non-Functional Requirements

Numbered `NFR-<area>-<n>`.

## Performance & latency (NFR-PERF)
- **NFR-PERF-1** Chat time-to-first-token p50 ≤ 1.5 s, p95 ≤ 4 s at demo scale.
- **NFR-PERF-2** Hybrid retrieval (vector + BM25 + rerank) p95 ≤ 800 ms for top-k ≤ 20.
- **NFR-PERF-3** 20-page digital PDF ingested end-to-end p50 ≤ 60 s.
- **NFR-PERF-4** API read endpoints p95 ≤ 300 ms (cached) / ≤ 700 ms (uncached).
- **NFR-PERF-5** Presigned-URL issuance ≤ 200 ms; uploads bypass the API (direct to S3).

## Scalability (NFR-SCALE)
- **NFR-SCALE-1** The ingestion tier shall scale horizontally by adding Temporal workers; throughput scales ~linearly with workers.
- **NFR-SCALE-2** The API tier shall be stateless (session state in Redis/DB) and horizontally scalable.
- **NFR-SCALE-3** pgvector HNSW parameters shall be tuned for ≥ 1M vectors per org at acceptable recall.
- **NFR-SCALE-4** DuckDB analytics shall handle spreadsheets up to ~10M rows via Parquet columnar scans.

## Reliability & availability (NFR-REL)
- **NFR-REL-1** Ingestion shall be durable: worker crashes never lose progress (Temporal event history).
- **NFR-REL-2** Every ingestion stage shall be idempotent and safely retryable.
- **NFR-REL-3** Permanent failures shall land in a DLQ and be operator-retryable without data corruption.
- **NFR-REL-4** Target availability (demo) 99.5%; no single-writer step without retry/compensation.
- **NFR-REL-5** RPO ≤ 24 h (daily backups), RTO ≤ 4 h for the stateful tier.

## Security & privacy (NFR-SEC)
- **NFR-SEC-1** All traffic over TLS; HSTS enabled.
- **NFR-SEC-2** Encryption at rest: S3 SSE + Postgres volume encryption.
- **NFR-SEC-3** Secrets from environment/secret manager; never committed; rotated on compromise.
- **NFR-SEC-4** Passwords hashed with Argon2id (tuned params); no plaintext or reversible storage.
- **NFR-SEC-5** Strict tenant isolation enforced by RLS + tested by automated cross-tenant tests.
- **NFR-SEC-6** Downloads via short-lived signed URLs; no public buckets.
- **NFR-SEC-7** Uploaded files validated (magic bytes) and virus-scanned before processing.
- **NFR-SEC-8** Retrieved/third-party content treated as untrusted (prompt-injection defense).
- **NFR-SEC-9** Audit log is append-only and tamper-evident.

## Maintainability (NFR-MAINT)
- **NFR-MAINT-1** Backend follows layered clean architecture (routers→services→repositories→models) with DI; domain logic unit-testable without I/O.
- **NFR-MAINT-2** External systems (S3, email, LLM, embeddings, OCR, transcription) behind interfaces with swappable implementations.
- **NFR-MAINT-3** All schema changes via Alembic migrations; no manual DDL in prod.
- **NFR-MAINT-4** Code formatted/linted (ruff/black, eslint/prettier); typed (mypy strict where practical, TS strict).
- **NFR-MAINT-5** ≥ 80% unit coverage on services/domain; critical workflows covered by integration/replay tests.

## Observability (NFR-OBS)
- **NFR-OBS-1** Structured JSON logs with correlation IDs (request_id, workflow_id, org_id) across services.
- **NFR-OBS-2** Metrics for API latency/error rates, queue depth, ingestion stage durations, token usage, cache hit rate.
- **NFR-OBS-3** Distributed traces (OpenTelemetry) spanning API → workflow → activities → model calls.
- **NFR-OBS-4** Grafana dashboards incl. ingestion pipeline + failure boards; alerts on error-rate/DLQ growth/latency SLO breaches.
- **NFR-OBS-5** AI calls traced in LangSmith; eval metrics reported in CI.

## Accessibility & UX (NFR-UX)
- **NFR-UX-1** WCAG 2.1 AA-minded: keyboard nav, focus states, ARIA, sufficient contrast.
- **NFR-UX-2** Responsive from mobile to desktop.
- **NFR-UX-3** Dark and light themes.
- **NFR-UX-4** Clear error, empty, and loading states for every async surface.

## Portability & ops (NFR-OPS)
- **NFR-OPS-1** Entire platform runs via `docker compose up` locally with seed data.
- **NFR-OPS-2** Each service has a multi-stage Dockerfile and pinned base images.
- **NFR-OPS-3** Config via environment variables validated at startup (fail fast on missing config).
- **NFR-OPS-4** CI runs lint, type-check, tests, build, and image scan on every PR.

## Cost (NFR-COST)
- **NFR-COST-1** Prefer self-hosted models (OCR, Whisper, reranker) to bound external cost.
- **NFR-COST-2** Cache repeat query embeddings; batch embedding calls; use `text-embedding-3-small`.
- **NFR-COST-3** Token usage recorded per org for analytics and guardrails.
