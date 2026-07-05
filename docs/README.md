# Foundry — Engineering Specification

> **Foundry** is an enterprise-grade, multimodal knowledge platform. Organizations upload
> large volumes of documents (PDF, DOCX, PPTX, XLSX, Markdown, images, audio, video), which
> are processed asynchronously through durable **Temporal** workflows, stored as semantic
> vectors in **pgvector** (and as queryable **DuckDB/Parquet** for tabular data), and made
> conversational through an **agentic Retrieval-Augmented Generation** chat.

This `/docs` set is the single source of truth for building Foundry. It is written to be
consumed **file-by-file by an autonomous coding agent** as well as by human engineers.
Read the documents in order; each one links forward to the next.

## How to read this spec

1. Start with the [Overview](./00-overview.md) for vision, personas, and scope.
2. Read the [Tech Versions](./02-tech-stack-and-versions.md) doc **before writing any code** — every version is pinned.
3. The [PRD](./01-prd.md), [Functional](./03-functional-requirements.md) and [Non-Functional](./04-non-functional-requirements.md) requirements define *what* to build.
4. The architecture docs (05–12) define *how*.
5. The [Roadmap](./20-roadmap-and-milestones.md) defines the *order* of work.

## Document index

| # | Document | Purpose |
|---|----------|---------|
| 00 | [Overview](./00-overview.md) | Vision, personas, scope, version strategy |
| 01 | [Product Requirements (PRD)](./01-prd.md) | Problem, goals, features, success metrics |
| 02 | [Tech Stack & Pinned Versions](./02-tech-stack-and-versions.md) | Exact, current (2026) versions to use |
| 03 | [Functional Requirements](./03-functional-requirements.md) | Numbered functional requirements |
| 04 | [Non-Functional Requirements](./04-non-functional-requirements.md) | Performance, scale, reliability, compliance |
| 05 | [User Stories & Acceptance Criteria](./05-user-stories-and-acceptance.md) | Stories with Given/When/Then AC |
| 06 | [UX Flows](./06-ux-flows.md) | Key end-to-end flows + diagrams |
| 07 | [Database Schema & ER Diagram](./07-database-schema.md) | Tables, DDL, RLS, ER diagram |
| 08 | [Backend Architecture](./08-backend-architecture.md) | Clean architecture, folder structure, DI |
| 09 | [Frontend Architecture](./09-frontend-architecture.md) | Next.js App Router, state, screens |
| 10 | [API Specification & Contracts](./10-api-specification.md) | REST/SSE/WS contracts, conventions |
| 11 | [Temporal Workflow Design](./11-temporal-workflows.md) | Workflows, activities, retries, signals |
| 12 | [RAG & Analytics Pipeline](./12-rag-and-analytics-pipeline.md) | Chunking, embeddings, hybrid retrieval, text-to-SQL |
| 13 | [Security Architecture](./13-security-architecture.md) | Authn/z, ABAC, encryption, scanning, injection defense |
| 14 | [Deployment & Infrastructure](./14-deployment-and-infrastructure.md) | Docker, Compose, CI/CD, environments |
| 15 | [Observability](./15-observability.md) | Logs, metrics, traces, dashboards, alerts |
| 16 | [Testing Strategy](./16-testing-strategy.md) | Unit → integration → workflow → e2e → load |
| 17 | [Diagrams](./17-diagrams.md) | Sequence, class, component, infra diagrams |
| 18 | [Risks & Trade-offs](./18-risks-and-tradeoffs.md) | Known risks, mitigations, decisions log |
| 19 | [Future Enhancements](./19-future-enhancements.md) | Deferred scope (SSO, etc.) |
| 20 | [Roadmap & Milestones](./20-roadmap-and-milestones.md) | Phased milestones → epics |

## Decision ledger (authoritative)

- **Tenancy:** single Postgres, shared schema, **Row-Level Security** on `organization_id`.
- **Hierarchy:** Organization → Workspace → Project → Document.
- **AuthN:** self-built JWT access + rotating refresh tokens (httpOnly cookie); Argon2id password hashing.
- **AuthZ:** **ABAC** policy engine (Casbin) layered over coarse org/project roles.
- **Onboarding:** invite-only + email verification + password reset (Resend/SES).
- **Super-admin:** minimal internal control plane (audited).
- **Ingestion:** ≤ 500 MB, presigned S3 multipart upload; content-hash dedup + document versioning.
- **Orchestration:** Temporal, **one workflow per file type**, shared common activities, DLQ + partial resume, cancel/pause signals + heartbeats.
- **Unstructured parsing:** self-hosted GLM OCR (PDF/DOCX/PPTX) with vision-LLM fallback; self-hosted Whisper + ffmpeg for audio/video.
- **Structured parsing:** XLSX → DuckDB + Parquet analytics path (agentic text-to-SQL).
- **Embeddings:** OpenAI `text-embedding-3-small` (1536 dims), pgvector HNSW, cosine.
- **Retrieval:** hybrid (vector + BM25, RRF) → cross-encoder rerank → RLS/metadata filters.
- **Chat:** streaming, agentic (retrieval + SQL + router tools), persisted threads + summarized memory, inline grounded citations.
- **AI eval:** LangSmith tracing + RAGAS-style offline eval set in CI.
- **Frontend state:** TanStack Query (server) + Zustand (client UI) + nuqs (URL) + RSC for reads.
- **Observability:** OpenTelemetry → Prometheus + Tempo + Loki + Grafana; GlitchTip (Sentry-compatible) for errors.
- **Everything is Dockerized.**
