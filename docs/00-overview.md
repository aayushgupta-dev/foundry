# 00 — Overview

## What Foundry is

Foundry is an enterprise-grade, multimodal knowledge platform. Organizations upload large
volumes of heterogeneous documents; Foundry ingests them asynchronously through durable
workflows, extracts and structures their content, stores semantic and tabular representations,
and exposes an agentic chat that answers questions **grounded in the organization's own data**
with inline citations back to the exact source.

The product is deliberately built to resemble a real, production SaaS — with multi-tenancy,
RBAC/ABAC, observability, durable orchestration, and evaluation — rather than a demo. Its
purpose is to serve as a **flagship portfolio system** demonstrating full-stack AI systems
architecture, while remaining a genuinely usable, generic knowledge platform.

## The problem it solves

Non-technical decision-makers (C-suite, consultants, analysts) sit on large, mixed piles of
documents — reports, decks, spreadsheets, contracts, recorded calls — and need answers and
insights from them without manually reading everything or waiting on an analyst to do bespoke
research. Foundry compresses that "gather → read → cross-reference → synthesize" loop into a
conversation: ask a question, get a grounded, cited answer, including **real analytics over
spreadsheet data** (aggregations, trends), not just semantic lookup.

## Personas

| Persona | Description | Primary needs |
|---|---|---|
| **Knowledge Consumer** (primary) | Non-technical exec/consultant/analyst who asks questions. | Fast, trustworthy, cited answers; analytics over tabular data; conversation history. |
| **Knowledge Curator / Admin** | Org admin who structures workspaces/projects, uploads/manages documents, manages members. | Bulk upload, ingestion visibility, permissions, versioning, recovery. |
| **Org Owner / Buyer** | Signs up the org, owns billing/security posture (hypothetical buyer: non-IT enterprise with lots of data). | Data isolation, audit logs, access control, reliability. |
| **Platform Super-Admin** (internal) | Foundry operator. | Cross-tenant visibility, support impersonation (audited), health. |
| **Evaluator** | Someone Aayush grants limited access to assess his capabilities. | A polished, obviously-production-grade end-to-end experience. |

## Scope philosophy

There is no artificially minimal MVP; the **full vision** is the target, delivered in ordered
milestones (see [Roadmap](./20-roadmap-and-milestones.md)). Every subsystem named in this spec
is in scope. Only **SSO/SAML** is explicitly deferred (see
[Future Enhancements](./19-future-enhancements.md)); the identity schema is designed so it can be
added without migration pain.

## Core capabilities (at a glance)

1. **Multi-tenant accounts** — Org → Workspace → Project → Document, shared-DB isolation via RLS.
2. **Invite-based auth** — JWT + rotating refresh, email verification, password reset, ABAC.
3. **Multimodal ingestion** — PDF, DOCX, PPTX, XLSX, Markdown, images, audio, video.
4. **Durable pipelines** — Temporal workflow per file type; dedup, versioning, DLQ, resume, cancel.
5. **Dual knowledge store** — pgvector (semantic) + DuckDB/Parquet (tabular analytics), one Postgres.
6. **Agentic RAG chat** — streaming, tool-using (retrieval + text-to-SQL + router), grounded citations.
7. **Evaluation & observability** — LangSmith + RAGAS-style evals; OTel/Prometheus/Grafana; audit logs.
8. **Fully Dockerized** — one `docker compose up` brings up the entire platform locally.

## Version strategy (summary)

Everything runs on the **latest stable / LTS** lines as of July 2026 — Python 3.14, Next.js 16,
React 19.2, Node 24 LTS, PostgreSQL 18 + pgvector 0.8.2, Redis 8, Tailwind 4, TypeScript 6,
TanStack Query 5, Zustand 5. Beta/RC lines are avoided. Exact pins live in
[02 — Tech Stack & Versions](./02-tech-stack-and-versions.md).

## Non-goals (v1)

- SSO/SAML/OIDC federation (schema-ready, deferred).
- Billing/metering/payments.
- On-prem/air-gapped deployment.
- Real-time multi-user co-editing.
- Native mobile apps (responsive web only).
