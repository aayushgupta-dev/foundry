# 08 — Backend Architecture

## Style
Layered **clean architecture** with the **repository pattern** and **dependency injection**.
Dependencies point inward: `api → services → repositories/ports → models`. Framework and I/O
live at the edges; domain/use-case logic is pure and unit-testable without a database or network.

```
HTTP / SSE / WS  ─┐
Temporal workers ─┼─►  application services (use-cases)  ─►  ports (interfaces)
CLI / jobs       ─┘                    │                         │
                                       ▼                         ▼
                                  domain models        adapters (Postgres, S3, OpenAI,
                                                        GLM OCR, Whisper, Redis, email)
```

## Ports (interfaces) — swappable adapters
- `EmailSender` → Resend/SES adapter (+ console/Mailpit for local dev).
- `ObjectStorage` → S3 adapter (MinIO locally).
- `LLMClient` → OpenAI Responses API adapter.
- `EmbeddingClient` → OpenAI embeddings adapter (batchable).
- `OCRClient` → self-hosted GLM OCR adapter (+ vision-LLM fallback adapter).
- `TranscriptionClient` → faster-whisper adapter.
- `RerankClient` → cross-encoder adapter.
- `VectorRepository`, `ChunkRepository`, `DocumentRepository`, ... → SQLAlchemy adapters.
- `Cache` → Redis adapter. `PolicyEngine` → Casbin adapter.

DI via a lightweight container (FastAPI `Depends` + a `providers` module, or `dependency-injector`).
Each request resolves a `UnitOfWork` bound to a DB session with `app.current_org` set for RLS.

## Folder structure

```
backend/
├── pyproject.toml            # uv/poetry; pinned deps (see 02-tech-stack-and-versions.md)
├── alembic.ini
├── Dockerfile                # multi-stage
├── src/foundry/
│   ├── main.py               # FastAPI app factory, middleware, router mounting
│   ├── config.py             # pydantic-settings, validated env, fail-fast
│   ├── container.py          # DI wiring / providers
│   ├── logging.py            # structured JSON logging + OTel setup
│   │
│   ├── api/                  # transport layer (thin)
│   │   └── v1/
│   │       ├── routers/      # auth, orgs, workspaces, projects, documents,
│   │       │                 # uploads, ingestion, chat, admin, health
│   │       ├── deps.py       # auth principal, current_org, pagination, idempotency
│   │       ├── schemas/      # pydantic request/response DTOs
│   │       └── errors.py     # exception handlers → RFC7807 problem+json
│   │
│   ├── services/             # use-cases (application logic)
│   │   ├── auth_service.py
│   │   ├── membership_service.py
│   │   ├── document_service.py
│   │   ├── upload_service.py
│   │   ├── ingestion_service.py   # starts/queries Temporal workflows
│   │   ├── retrieval_service.py   # hybrid search + rerank
│   │   ├── chat_service.py        # agent loop, streaming, memory
│   │   ├── analytics_service.py   # DuckDB text-to-SQL
│   │   └── admin_service.py
│   │
│   ├── domain/               # entities, value objects, policies, errors (pure)
│   │   ├── models.py
│   │   ├── policies.py       # ABAC rule definitions
│   │   └── errors.py
│   │
│   ├── ports/                # interfaces (Protocols/ABCs)
│   ├── adapters/             # implementations
│   │   ├── db/               # SQLAlchemy models, repositories, UoW, session
│   │   ├── storage/          # S3/MinIO
│   │   ├── ai/               # openai, glm_ocr, whisper, rerank, embeddings
│   │   ├── cache/            # redis
│   │   ├── email/            # resend/ses/mailpit
│   │   └── policy/           # casbin
│   │
│   ├── workflows/            # Temporal (see 11-temporal-workflows.md)
│   │   ├── common_activities.py
│   │   ├── pdf_workflow.py
│   │   ├── docx_workflow.py
│   │   ├── pptx_workflow.py
│   │   ├── xlsx_workflow.py
│   │   ├── audio_workflow.py
│   │   ├── video_workflow.py
│   │   ├── image_workflow.py
│   │   ├── markdown_workflow.py
│   │   ├── dispatcher.py     # file kind → workflow
│   │   └── worker.py         # worker entrypoint
│   │
│   ├── cli/                  # typer CLI: seed, create-admin, reindex, eval
│   └── evals/               # RAGAS-style eval harness + golden set
│
├── migrations/               # alembic versions
└── tests/
    ├── unit/  integration/  workflow/  e2e/  load/
```

## Request lifecycle
1. Middleware: request-id, CORS, security headers, rate-limit (Redis token bucket), OTel span.
2. Auth dependency: verify access JWT → principal; load memberships; `SET LOCAL app.current_org`.
3. Idempotency dependency (mutating routes): check/store `Idempotency-Key` in Redis+DB.
4. Router calls a **service**; service uses **repositories/ports** via the injected UoW.
5. ABAC check (`PolicyEngine.enforce(subject, resource, action, ctx)`) before side effects.
6. Response serialized via pydantic DTO; errors mapped to `application/problem+json`.

## Configuration
`pydantic-settings` model validated at startup; missing/invalid config aborts boot. Keys:
`DATABASE_URL`, `REDIS_URL`, `S3_*`, `OPENAI_API_KEY`, `OPENAI_CHAT_MODEL`, `EMBEDDING_MODEL`,
`TEMPORAL_*`, `GLM_OCR_URL`, `WHISPER_URL`, `RERANKER_URL`, `EMAIL_*`, `JWT_*`, `LANGSMITH_*`.

## Background work
Temporal owns durable pipelines. Redis is used for caching, rate-limit buckets, idempotency,
and pub/sub fan-out of ingestion status to WS/SSE subscribers (workers publish; API relays).

## Error handling
Domain raises typed errors (`NotFound`, `Forbidden`, `Conflict`, `ValidationError`,
`RateLimited`). A single exception-handler layer maps them to problem+json with a stable
`code`, `message`, and `request_id`. No stack traces leak to clients.

## API versioning & caching
All routes under `/api/v1`. Additive changes in-place; breaking changes → `/api/v2`. Cache-aside
in Redis for permission lookups, list endpoints, and repeat query embeddings, with explicit
invalidation on writes and short TTLs.
