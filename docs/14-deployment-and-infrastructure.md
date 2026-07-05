# 14 — Deployment & Infrastructure

**Everything is Dockerized.** `docker compose up` brings up the entire platform locally with seed
data. Each service has its own multi-stage Dockerfile with pinned base images
(`python:3.14-slim`, `node:24-slim`).

## Services (Docker Compose, local/dev)

| Service | Image / build | Purpose |
|---|---|---|
| `api` | build backend | FastAPI (`/api/v1`), SSE/WS, health |
| `worker-cpu` | build backend | Temporal worker: validate/parse/chunk/persist |
| `worker-gpu` | build backend | Temporal worker: OCR/Whisper/rerank/embed batching |
| `frontend` | build frontend | Next.js 16 (SSR) |
| `postgres` | `postgres:18` + pgvector 0.8.2 | relational + vector + full-text (single DB) |
| `redis` | `redis:8` | cache, rate-limit, idempotency, pub/sub |
| `temporal` | temporal server | orchestration |
| `temporal-ui` | temporal web | workflow debugging UI |
| `minio` | minio | S3-compatible object storage (dev) |
| `glm-ocr` | self-hosted GLM OCR | OCR inference |
| `whisper` | faster-whisper server | transcription |
| `reranker` | cross-encoder server | rerank inference |
| `clamav` | clamav/clamd | virus scanning |
| `mailpit` | mailpit | local email capture (dev) |
| `otel-collector` | otel collector | telemetry pipeline |
| `prometheus` / `tempo` / `loki` / `grafana` | LGTM stack | metrics / traces / logs / dashboards |
| `glitchtip` | glitchtip | error tracking (Sentry-compatible) |

> DuckDB runs **in-process** inside the workers/analytics service (embedded), reading Parquet from
> MinIO/S3 — it is not a standalone service.

## Compose layout
```
infra/
├── docker-compose.yml          # base: all services + healthchecks + depends_on
├── docker-compose.override.yml # dev: hot-reload, mailpit, minio console, seed
├── docker-compose.obs.yml      # optional observability stack (LGTM + glitchtip)
├── .env.sample                 # documented env vars
└── grafana/  prometheus/  otel/  # provisioned dashboards + configs
```
- Healthchecks + `depends_on: condition: service_healthy` gate startup ordering (DB/Temporal before API/workers).
- Named volumes persist `postgres`, `minio`, `grafana`, `redis`.
- One-shot `migrate` + `seed` init containers run Alembic migrations and load demo data.

## Environments
| Env | Frontend | Backend/workers | Data |
|---|---|---|---|
| Local | Next dev (Docker) | api + workers (Docker) | postgres+pgvector, redis, minio, temporal (Docker) |
| CI | build + test only | build + test | ephemeral Postgres/Redis services |
| Prod | **Vercel** | **Render/AWS** (ECS/Fargate or Render services) | **AWS RDS Postgres 18 + pgvector**, ElastiCache Redis, S3, Temporal Cloud or self-hosted |

## CI/CD (GitHub Actions)
Pipelines:
1. **Backend CI:** ruff/black, mypy, `pytest` (unit + integration via service containers),
   workflow replay tests, build image, image scan (Trivy).
2. **Frontend CI:** eslint/prettier, `tsc`, vitest, `next build`, Playwright e2e (against compose).
3. **AI eval:** run RAGAS-style eval set; fail on threshold regressions.
4. **Deploy:** on `main` → build+push images → deploy backend (Render/AWS) + frontend (Vercel);
   run migrations as a release step; smoke tests post-deploy.

## Scaling & resilience (prod)
- **API** stateless → horizontal autoscale behind a load balancer.
- **Workers** scale per task queue; `worker-gpu` scales independently for model-heavy load.
- **Postgres** primary + read replica; pgvector on primary; connection pooling (PgBouncer).
- **Redis** managed with persistence for idempotency.
- **Object storage** S3 with lifecycle rules (transition temp/parquet).

## Backup & disaster recovery
- Daily automated Postgres snapshots + PITR (WAL); S3 versioning + cross-region replication (prod).
- **RPO ≤ 24 h, RTO ≤ 4 h** (see [04 NFR](./04-non-functional-requirements.md)).
- Restore runbook documented; backups periodically restore-tested.
- Temporal history provides ingestion durability; re-running a failed run is safe (idempotent).

## Configuration & secrets
- All config via env, validated at boot (fail-fast). Prod secrets from AWS SSM/Secrets Manager
  (or Render secrets); never baked into images.

## Infrastructure diagram
See [17 — Diagrams](./17-diagrams.md).
