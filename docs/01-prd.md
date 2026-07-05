# 01 — Product Requirements Document (PRD)

## 1. Summary

Foundry is a multi-tenant, multimodal knowledge platform that turns an organization's document
sprawl into a conversational, cited, analytically-capable knowledge base. Users create
workspaces and projects, upload documents of many types, and chat with their knowledge through
an agentic RAG system that both retrieves grounded passages and runs real analytics over
spreadsheet data.

## 2. Goals & success metrics

| Goal | Metric | Target (demo scale) |
|---|---|---|
| Trustworthy answers | % answers with valid inline citations | ≥ 95% |
| Grounding | RAGAS faithfulness score on eval set | ≥ 0.85 |
| Retrieval quality | Context recall on eval set | ≥ 0.80 |
| Ingestion reliability | % documents reaching `indexed` without manual intervention | ≥ 99% |
| Ingestion resumability | Failed-stage retries that resume (not restart) | 100% of retryable failures |
| Responsiveness | Chat time-to-first-token (p50) | ≤ 1.5 s |
| Ingestion latency | 20-page PDF end-to-end (p50) | ≤ 60 s |
| Isolation | Cross-tenant data leaks | 0 (enforced by RLS + tests) |

## 3. Personas & top jobs-to-be-done

See [Overview](./00-overview.md). The primary JTBD: *"As an exec/consultant, I want to ask
natural-language questions across all my project's documents and spreadsheets and get a
trustworthy, cited answer, so I don't have to read everything or wait on an analyst."*

## 4. Feature set (in scope)

### 4.1 Accounts, tenancy & access
- Invite-only onboarding; email verification; password reset.
- Org → Workspace → Project → Document hierarchy.
- Roles: Org (Owner/Admin/Member) + Project (Editor/Viewer/Guest), enforced by **ABAC** policies.
- Shared-DB multi-tenancy with **Row-Level Security** on `organization_id`.
- Minimal, audited super-admin control plane.

### 4.2 Document management
- Upload up to **500 MB** per file via presigned S3 multipart (direct-to-storage).
- Supported types: PDF, DOCX, PPTX, XLSX, Markdown, images (PNG/JPG/TIFF), audio (mp3/wav/m4a), video (mp4/mov).
- Folders, tags, ownership, sharing, soft-delete + recovery (trash), archival.
- Content-hash **dedup**; **versioning** on content change (old vectors retired).

### 4.3 Ingestion pipeline (Temporal)
- One durable workflow per file type; shared reusable activities.
- Stages: validate → virus scan → extract (OCR/transcribe/parse) → chunk → embed → persist → index.
- Per-activity retries w/ backoff; **DLQ** (poison store); **partial resume** from last good stage.
- Cancel / pause / resume via signals; heartbeats on long OCR/transcription activities.
- Live per-stage status via DB + WebSocket/SSE; Temporal Web UI for deep debugging.

### 4.4 Knowledge stores
- **Unstructured** → chunks + embeddings in pgvector (HNSW, cosine, 1536-dim).
- **Structured (XLSX)** → normalized to Parquet, queried by DuckDB (agentic text-to-SQL).
- Rich chunk metadata (doc, version, page/slide/timestamp, section) for citations + filtering.

### 4.5 Chat (agentic RAG)
- Streaming responses (SSE) with tool-calling: `vector_retrieve`, `sql_query`, `router`.
- Hybrid retrieval (vector + BM25 → RRF → rerank → RLS/metadata filters).
- Persisted conversation threads with sliding-window + summarized long-term memory.
- Inline citations deep-linking to source (page/slide/timestamp); "insufficient context" fallback.
- Prompt-injection defenses on retrieved content.

### 4.6 Observability & evaluation
- LangSmith tracing of retrieval/generation; RAGAS-style offline eval set run in CI.
- OpenTelemetry → Prometheus/Tempo/Loki/Grafana; GlitchTip errors; audit logs; health probes.

### 4.7 Frontend
- Auth screens, dashboard, workspace/project navigation, drag-drop upload with live progress,
  streaming chat with citations + source viewer, admin console + usage analytics.
- Dark mode, responsive, accessible (WCAG-minded).

## 5. Out of scope (v1)
SSO/SAML, billing, on-prem, real-time co-editing, native mobile. See [Future Enhancements](./19-future-enhancements.md).

## 6. Constraints & assumptions
- **Everything Dockerized**; local dev is `docker compose up`.
- Single Postgres 18 for relational + vector (pgvector extension) + full-text.
- Self-hosted model workers (GLM OCR, Whisper, reranker) to minimize external cost and showcase infra.
- OpenAI used for chat generation + embeddings.
- Latest stable/LTS versions per [02](./02-tech-stack-and-versions.md).

## 7. Dependencies & risks
See [Risks & Trade-offs](./18-risks-and-tradeoffs.md). Key risks: self-hosted model resource
footprint, OCR quality on poor scans, text-to-SQL correctness, and demo-time cost of embeddings.

## 8. Release plan
Phased milestones M0–M7; see [Roadmap](./20-roadmap-and-milestones.md).
