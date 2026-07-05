# 18 — Risks & Trade-offs

## Key risks & mitigations

| # | Risk | Impact | Likelihood | Mitigation |
|---|---|---|---|---|
| R1 | Self-hosted model workers (GLM OCR, Whisper, reranker) are resource-heavy | High local/prod cost; slow demos on weak hardware | High | Isolate on `worker-gpu` queue; make models configurable; allow CPU/quantized variants; document minimum hardware; allow API fallbacks. |
| R2 | OCR quality on poor scans | Bad chunks → bad answers | Medium | Native-parse-first; vision-LLM fallback; confidence thresholds; per-doc reprocess. |
| R3 | Text-to-SQL correctness/safety | Wrong analytics or unsafe queries | Medium | Schema introspection, validation (read-only, whitelisted tables, LIMIT, timeout), return SQL for transparency, eval with exact-result checks. |
| R4 | Embedding cost/latency at scale | Cost + slow ingest | Medium | `text-embedding-3-small`, batching, dedup, caching; usage metering. |
| R5 | RLS misconfiguration → cross-tenant leak | Critical | Low | Non-superuser DB role; policy on every table; automated zero-leak tests; deny-by-default ABAC. |
| R6 | Prompt injection via document content | Data exfiltration / misbehavior | Medium | Untrusted-content handling, delimiters, instruction hierarchy, tools only via controller, injection tests. |
| R7 | Temporal learning curve / determinism bugs | Pipeline flakiness | Medium | Keep I/O in activities; replay tests; idempotent activities; clear activity boundaries. |
| R8 | Large-file upload reliability | Failed uploads | Medium | Presigned multipart, parallel parts, client retry; (tus resumable as future option). |
| R9 | Scope breadth (portfolio) → never "done" | Stalled delivery | High | Milestone ordering (M0→M7); each milestone independently demoable. |
| R10 | Version drift (fast-moving 2026 stack) | Build breaks | Medium | Pinned versions + lockfiles ([02](./02-tech-stack-and-versions.md)); avoid beta/RC lines; renovate later. |
| R11 | pgvector recall vs latency tuning | Poor retrieval | Medium | Tune HNSW `m`/`ef_*`; hybrid + rerank; eval-driven tuning. |
| R12 | Streaming/SSE + WS complexity across proxies | Broken realtime UX | Medium | Standard SSE for chat; WS for status with fallback to polling; test behind proxy. |

## Decision log (ADR-style summary)

| ID | Decision | Rationale | Trade-off accepted |
|---|---|---|---|
| D1 | Shared DB + RLS multi-tenancy | Simplest correct isolation; easy to seed/demo | Weaker physical isolation than DB-per-tenant |
| D2 | Self-built JWT + rotating refresh | Showcases auth engineering | More code/maintenance vs managed auth |
| D3 | ABAC (Casbin) over plain RBAC | Fine-grained, impressive, future-proof | More complexity than role flags |
| D4 | Workflow-per-file-type | Clear, type-specific pipelines, shared activities | Some duplication vs one generic workflow |
| D5 | Single Postgres for vector (pgvector) | One store, transactional, joins with metadata | Not a specialized vector DB at extreme scale |
| D6 | DuckDB text-to-SQL for spreadsheets | Real analytics, not lossy prose embedding | New subsystem + SQL-safety burden |
| D7 | `text-embedding-3-small` (1536) | Cost/latency at demo scale | Slightly lower recall than `-large` |
| D8 | Hybrid + rerank retrieval | Best answer quality | Extra latency + a rerank service |
| D9 | Self-hosted models (OCR/Whisper/rerank) | Cost control + infra showcase | Resource footprint (R1) |
| D10 | TanStack Query + Zustand (no Redux) | 2026 best practice for App Router | Two libraries, clear separation of concerns |
| D11 | Latest stable/LTS, avoid beta/RC | Stability + modern signal | Not bleeding-edge (e.g., TS 7 RC) |
| D12 | Defer SSO/SAML | Focus scope; schema-ready | No enterprise federation in v1 |
| D13 | OTel + LGTM + GlitchTip (self-hosted, free) | Full observability at no license cost | More containers to run |

See [19 — Future Enhancements](./19-future-enhancements.md) for deferred scope.
