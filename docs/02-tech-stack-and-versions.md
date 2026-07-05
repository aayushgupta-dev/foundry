# 02 — Tech Stack & Pinned Versions

> **Rule for the implementing agent:** Use the versions pinned below. Prefer the **latest stable**
> release of each pinned major/LTS line. **Do not** silently downgrade a major (e.g. do not use
> Next.js 15 when 16 is specified) and **do not** adopt beta/RC/pre-release lines (e.g. TypeScript 7
> RC, SQLAlchemy 2.1 beta, PostgreSQL 19 beta). Versions verified July 2026.

## Version strategy

- **Runtimes on LTS:** Node.js on the current **Active LTS** (24.x). Node 26 is *Current*, not LTS until Oct 2026 — do not use it.
- **Languages/libraries on latest stable major:** Python 3.14, Next.js 16, React 19.2, Tailwind 4.
- **Avoid pre-release lines:** TypeScript stays on 6.x (7.0 is RC and Go-based — too new); SQLAlchemy stays on 2.0.x (2.1 is beta); PostgreSQL stays on 18.x (19 is beta); DuckDB uses the 1.4 **LTS** line for stability with 1.5.x as an option.
- **Pin exact versions** in lockfiles (`package-lock.json` / `uv.lock` or `poetry.lock`). Ranges below indicate the acceptable line.

## Backend (Python)

| Technology | Pinned line | Verified latest (Jul 2026) | Notes |
|---|---|---|---|
| Python | 3.14.x | 3.14.6 | Latest stable feature release. Use 3.14 (not 3.13 maintenance line). |
| FastAPI | 0.139.x | 0.139.0 | Requires Python ≥3.10. |
| Uvicorn | latest stable | — | ASGI server, run behind Gunicorn or standalone in container. |
| Pydantic | 2.13.x | 2.13.4 | v2 API only. |
| SQLAlchemy | 2.0.x | 2.0.51 | **Do not** use 2.1 beta. 2.0 declarative + async. |
| Alembic | 1.18.x | 1.18.5 | Migrations. |
| Temporal Python SDK (`temporalio`) | latest 1.x | 2026-07 release | Pin exact at scaffold time. |
| `psycopg` (v3) | latest stable | — | Async Postgres driver. |
| `pgvector` (python client) | latest stable | — | SQLAlchemy integration for vector columns. |
| `redis` (python client) | latest stable | — | Supports Redis 8.x. |
| `duckdb` (python) | 1.4.x LTS | 1.4.4 LTS (1.5.3 latest) | Use LTS for stability; 1.5.x acceptable. |
| `pyarrow` | latest stable | — | Parquet read/write. |
| `argon2-cffi` | latest stable | — | Password hashing (Argon2id). |
| `casbin` | latest stable | — | ABAC policy engine. |
| `python-jose` / `pyjwt` | latest stable | — | JWT sign/verify (choose `pyjwt`). |
| `openai` (python) | latest stable | — | Responses API + Embeddings. |
| `langchain` / `langsmith` | latest stable | — | LangChain **only where it adds value**; LangSmith tracing. |
| `boto3` | latest stable | — | S3 / SES. |
| `ffmpeg` (system) | latest stable | — | Audio extraction from video. |
| `faster-whisper` | latest stable | — | Self-hosted transcription. |
| `pytest`, `pytest-asyncio`, `httpx` | latest stable | — | Testing. |

## Frontend (Node/TypeScript)

| Technology | Pinned line | Verified latest (Jul 2026) | Notes |
|---|---|---|---|
| Node.js | 24.x **LTS** | 24.x Active LTS | Runtime for build/SSR. Not Node 26 (Current). |
| Next.js | 16.x | 16.2.7 | App Router, RSC, Turbopack default, React Compiler stable. |
| React / React DOM | 19.2.x | 19.2.7 | Bundled/compatible with Next 16. |
| TypeScript | 6.0.x | 6.0 | **Not** 7.0 RC. |
| Tailwind CSS | 4.x | 4.3 | v4 engine (CSS-first config). |
| TanStack Query | 5.x | 5.101.2 | Server-state management. |
| Zustand | 5.0.x | 5.0.14 | Client/UI state. |
| `nuqs` | latest stable | — | URL/searchParams state (filters, sort, pagination). |
| shadcn/ui + Radix | latest stable | — | Accessible component primitives. |
| `@tanstack/react-table` | latest stable | — | Data tables (admin, docs list). |
| Playwright | latest stable | — | E2E tests. |
| Vitest + Testing Library | latest stable | — | Unit/component tests. |

## Data & Infrastructure

| Technology | Pinned line | Verified latest (Jul 2026) | Notes |
|---|---|---|---|
| PostgreSQL | 18.x | 18.4 | **Single DB** for relational + vector + full-text. Not 19 beta. |
| pgvector (extension) | 0.8.x | 0.8.2 | Postgres **extension**, same DB. HNSW index. 0.8.2 fixes CVE-2026-3172. |
| Redis | 8.x | 8.8 | Cache, rate-limit buckets, idempotency store, pub/sub. |
| Temporal Server | latest stable | — | Orchestration; ships with Temporal Web UI. |
| DuckDB engine | 1.4 LTS | 1.4.4 LTS | Embedded analytics over Parquet. |
| MinIO | latest stable | — | S3-compatible object store (local/dev); AWS S3 in prod. |
| ClamAV | latest stable | — | Virus scanning service. |
| Docker Engine / Compose | latest stable | — | Compose v2 syntax. |
| OpenTelemetry Collector | latest stable | — | Telemetry pipeline. |
| Prometheus / Tempo / Loki / Grafana | latest stable | — | Metrics / traces / logs / dashboards. |
| GlitchTip | latest stable | — | Sentry-compatible error tracking (lightweight, self-hosted). |

## AI models & endpoints

| Purpose | Model / service | Notes |
|---|---|---|
| Chat generation | OpenAI **Responses API** (current default reasoning/general model) | Streaming + tool calling. |
| Embeddings | OpenAI `text-embedding-3-small` | **1536 dims**, cosine. Store `embedding_model` + `dim` for re-index. |
| OCR | **GLM OCR** (self-hosted worker) | PDF/DOCX/PPTX rasterized pages; primary. |
| OCR fallback | Vision LLM | Only when GLM OCR fails/low-confidence. |
| Transcription | **faster-whisper** (self-hosted) | Audio + (video→audio via ffmpeg). |
| Reranking | Cross-encoder reranker (self-hosted, e.g. `bge-reranker`) | Post-fusion rerank. |
| Tabular analytics | **DuckDB text-to-SQL** (LLM-generated SQL) | Over Parquet extracts of XLSX. |

## Notes on model/version drift

Because model names and library patch versions move quickly, the implementing agent should:
1. Read `embedding_model`/`dim` from config, never hard-code dimension math in more than one place.
2. Keep an `OPENAI_CHAT_MODEL` env var; do not scatter model strings across the codebase.
3. Re-verify pinned versions at scaffold time and record the exact resolved versions in lockfiles.
