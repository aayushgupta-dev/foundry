# 15 — Observability

Stack: **OpenTelemetry** instrumentation → **Prometheus** (metrics), **Tempo** (traces),
**Loki** (logs), visualized in **Grafana**; **GlitchTip** (Sentry-compatible) for error tracking;
**LangSmith** for AI-specific tracing. All self-hostable and Dockerized (free tier / OSS).

## Logs
- **Structured JSON** logs from every service with correlation fields: `request_id`, `trace_id`,
  `org_id`, `user_id`, `workflow_id`, `run_id`, `stage`.
- No secrets/PII/full document text in logs.
- Shipped to Loki via the OTel collector; queryable in Grafana; retained per environment policy.

## Metrics (Prometheus)
- **API:** request rate, latency histograms (p50/p95/p99), error rate by route, in-flight requests.
- **Auth:** login success/failure, refresh rotations, token-reuse detections.
- **Ingestion:** workflows started/completed/failed, per-stage duration histograms, DLQ size,
  quarantine count, retry/resume counts, queue backlog per task queue.
- **AI:** embedding calls, chat requests, time-to-first-token, tokens in/out, tool-call counts,
  rerank latency, cache hit rate.
- **Infra:** DB connections/pool saturation, pgvector query latency, Redis hit rate, S3 op latency.
- **Usage:** per-org tokens, storage, documents, queries (also persisted to `usage_events`).

## Traces (OpenTelemetry → Tempo)
- End-to-end spans: `HTTP request → service → repository/DB`, and
  `API → Temporal workflow → activities → model calls`.
- Trace + log correlation via shared `trace_id`; jump from a log line to its trace in Grafana.
- Model calls (OpenAI, OCR, Whisper, rerank) are child spans with token/latency attributes.

## AI tracing & evaluation (LangSmith)
- Every retrieval + generation traced: query, retrieved chunks (ids + scores), rerank order,
  prompt, tool calls, SQL executed, tokens, latency.
- Offline eval runs (RAGAS-style) logged for trend tracking; CI gate on faithfulness/recall.

## Error tracking (GlitchTip)
- Backend + frontend exceptions captured with breadcrumbs and correlation ids.
- Release-tagged; alerting on new/regressed error signatures.

## Dashboards (Grafana, provisioned)
1. **API Overview** — latency/error/throughput SLOs.
2. **Ingestion Pipeline** — funnel by stage, durations, throughput, workers.
3. **Ingestion Failures** — DLQ growth, failure reasons, retry/resume, quarantine.
4. **AI Quality & Cost** — TTFT, tokens, cache hit, eval-score trends.
5. **Infra Health** — DB/Redis/S3, pool saturation, pgvector latency.

## Alerts (examples)
- API 5xx rate > 2% for 5m; p95 latency SLO breach.
- DLQ size increasing / ingestion failure rate > 5%.
- Auth failure spike (possible attack).
- Worker backlog above threshold; Temporal task-queue latency high.
- Postgres connection saturation; disk usage high.

## Health checks
- `/health` (liveness) and `/ready` (readiness: DB, Redis, Temporal, S3 reachable).
- Compose/orchestrator uses these to gate traffic and ordering.
