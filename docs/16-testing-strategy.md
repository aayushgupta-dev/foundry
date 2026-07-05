# 16 — Testing Strategy

Test pyramid: many fast unit tests, focused integration tests, targeted workflow + e2e tests,
plus load and AI-eval suites. All tests run in CI ([14](./14-deployment-and-infrastructure.md)).

## Backend
### Unit (pytest)
- Domain/service logic with **mocked ports** (no DB/network). Covers auth (token rotation/reuse),
  ABAC decisions, dedup/versioning logic, chunking, retrieval fusion/ranking, SQL guardrails.
- Target ≥ 80% coverage on `services/` + `domain/`.

### Integration (pytest + service containers)
- Real Postgres (+pgvector) and Redis via testcontainers/compose.
- **RLS/tenant-isolation tests** (critical): assert Org A can never read Org B rows across every
  repository — zero-leak requirement.
- Repository CRUD, migrations apply cleanly, cursor pagination, idempotency, cache invalidation.
- pgvector: insert vectors, HNSW search returns expected neighbors; hybrid + rerank ordering.

### Workflow (Temporal)
- **Replay tests** against recorded histories to guard workflow determinism.
- Activity unit tests with mocked adapters.
- **Failure-injection** integration tests: force stage failures → assert retry/backoff, DLQ write,
  and **partial resume** (completed stages skipped, no duplicate data).
- Signal tests: cancel → compensation cleanup; pause/resume.

## Frontend
- **Unit/component** (Vitest + Testing Library): components, hooks, stores, api-client refresh logic.
- **E2E** (Playwright): login → create project → upload (mock S3) → watch progress → chat with
  streamed answer + citation click. Accessibility checks (axe) on key screens.

## AI evaluation
- **Golden set** of Q/A + expected sources (semantic) and Q/expected-result (tabular) in `evals/`.
- **Metrics:** RAGAS-style faithfulness, answer relevance, context precision/recall; SQL
  exact-result checks for analytics questions.
- **CI gate:** build fails if faithfulness < 0.85 or context recall < 0.80 (thresholds in [01](./01-prd.md)).
- Injection test cases: adversarial documents must not subvert instructions or tools.

## Load & performance
- **k6** (or Locust) scenarios: concurrent chat streaming, bulk uploads, list endpoints.
- Ingestion throughput test: N documents → measure per-stage durations and worker scaling.
- Assert SLOs from [04 NFR](./04-non-functional-requirements.md) (TTFT, retrieval p95, ingest p50).

## Security testing
- Automated: dependency scan, image scan (Trivy), secret scan in CI.
- Auth abuse tests (rate-limit enforcement, reset enumeration resistance).
- Upload safety tests (bad magic bytes rejected; ClamAV quarantine path).

## Test data & environments
- Deterministic seed factory (users, orgs, projects, sample multimodal docs).
- Fake/mocked external adapters (OpenAI, S3) for unit; real containers for integration.
- Every PR runs lint + type-check + unit + integration + build; nightly runs e2e + load + eval.
