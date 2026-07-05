# 03 — Functional Requirements

Requirements are numbered `FR-<area>-<n>` for traceability. Each maps to user stories
([05](./05-user-stories-and-acceptance.md)) and roadmap epics ([20](./20-roadmap-and-milestones.md)).

## Authentication & accounts (FR-AUTH)

- **FR-AUTH-1** The system shall allow an invited user to accept an invite via a signed, single-use, expiring token and set a password (Argon2id hashed).
- **FR-AUTH-2** The system shall require email verification before granting full access; unverified users have restricted access.
- **FR-AUTH-3** The system shall issue a short-lived access JWT (~15 min) and a rotating refresh token stored in an httpOnly, Secure, SameSite cookie.
- **FR-AUTH-4** The system shall rotate refresh tokens on use and detect reuse of a revoked refresh token (token theft), revoking the family.
- **FR-AUTH-5** The system shall support password reset via a signed, single-use, expiring token delivered by email.
- **FR-AUTH-6** The system shall support logout that revokes the current refresh-token family.
- **FR-AUTH-7** The system shall allow self-serve organization creation only via the initial owner signup path (first user becomes Org Owner); subsequent members join by invite.
- **FR-AUTH-8** The system shall rate-limit auth endpoints (login, reset, verify) more strictly than general endpoints.

## Authorization (FR-AUTHZ)

- **FR-AUTHZ-1** The system shall enforce tenant isolation via Postgres Row-Level Security keyed on `organization_id` for all tenant tables.
- **FR-AUTHZ-2** The system shall evaluate every protected action through an ABAC policy (Casbin) taking `(subject, resource, action, context)`.
- **FR-AUTHZ-3** Org roles (Owner, Admin, Member) and Project roles (Editor, Viewer, Guest) shall map to ABAC policy rules.
- **FR-AUTHZ-4** The system shall deny by default; absence of an allowing policy = 403.
- **FR-AUTHZ-5** The super-admin role shall access a separate control plane, bypassing RLS via an explicit service role, with every action written to the audit log.

## Workspaces, projects & documents (FR-ORG)

- **FR-ORG-1** Users shall create/rename/delete workspaces within their org (permission-gated).
- **FR-ORG-2** Users shall create/rename/archive/delete projects within a workspace.
- **FR-ORG-3** Users shall organize documents in folders and apply tags.
- **FR-ORG-4** Documents shall have an owner and inherit project-level permissions; explicit shares may grant additional access.
- **FR-ORG-5** Deletion shall be soft (trash) with a recovery window; hard-delete removes S3 objects and vectors.
- **FR-ORG-6** Archived projects shall be read-only and excluded from default lists.

## Upload & ingestion (FR-ING)

- **FR-ING-1** The system shall issue presigned multipart upload URLs for direct-to-S3 uploads up to 500 MB.
- **FR-ING-2** On upload completion, the system shall compute a content SHA-256 and dedupe identical content within the org (skip re-processing, link to existing).
- **FR-ING-3** Re-uploading a logically-same document with new content shall create a new **version**, re-index it, and soft-retire prior version vectors.
- **FR-ING-4** The system shall validate file signature/MIME against an allowlist and reject unsupported/oversized files.
- **FR-ING-5** The system shall virus-scan every file (ClamAV) before extraction; infected files are quarantined and flagged.
- **FR-ING-6** The system shall select and start the Temporal workflow matching the file type.
- **FR-ING-7** PDF/DOCX/PPTX shall be parsed natively where a text layer exists, and via self-hosted GLM OCR otherwise, with a vision-LLM fallback only on OCR failure.
- **FR-ING-8** Audio shall be transcribed via self-hosted Whisper; video shall have audio extracted via ffmpeg then transcribed.
- **FR-ING-9** XLSX (and CSV) shall be normalized into Parquet and registered for DuckDB querying (not embedded as prose).
- **FR-ING-10** Images shall be OCR'd (GLM) and/or captioned by a vision model.
- **FR-ING-11** Unstructured text shall be chunked structure-aware (~500–800 tokens, ~15% overlap) with metadata (page/slide/timestamp/section).
- **FR-ING-12** Chunks shall be embedded (`text-embedding-3-small`, 1536) and stored in pgvector with an HNSW index.
- **FR-ING-13** Each ingestion stage shall persist status to the DB and stream updates to the client via WS/SSE.
- **FR-ING-14** Failed activities shall retry with exponential backoff; permanent failures shall be recorded in a DLQ store.
- **FR-ING-15** Users shall retry a failed document; retry shall resume from the last successful stage, not restart.
- **FR-ING-16** Users shall cancel or pause/resume in-flight ingestion; cancellation shall compensate (clean partial S3/vector artifacts).

## Retrieval & chat (FR-CHAT)

- **FR-CHAT-1** Users shall start conversation threads scoped to a project; threads persist with messages.
- **FR-CHAT-2** The chat shall stream tokens to the client over SSE.
- **FR-CHAT-3** The chat agent shall select tools per question: `vector_retrieve`, `sql_query` (DuckDB), and a `router`.
- **FR-CHAT-4** Retrieval shall be hybrid (vector + BM25), fused via RRF, reranked by a cross-encoder, and filtered by RLS + metadata (project, tags, doc, version).
- **FR-CHAT-5** For analytical questions over spreadsheets, the agent shall introspect the DuckDB schema, generate SQL, execute it over Parquet, and return results plus a natural-language explanation and the SQL used.
- **FR-CHAT-6** Every factual claim shall carry an inline citation resolving to a specific chunk/page/slide/timestamp; the UI shall deep-link to the source.
- **FR-CHAT-7** When retrieval confidence is low, the assistant shall state it lacks sufficient context rather than fabricate.
- **FR-CHAT-8** Conversation memory shall keep recent turns verbatim and roll older turns into a running summary to fit the context window.
- **FR-CHAT-9** Retrieved content shall be treated as untrusted and passed through prompt-injection defenses.

## Admin & observability (FR-ADM)

- **FR-ADM-1** Super-admin shall list organizations, view usage, and impersonate for support (audited).
- **FR-ADM-2** Org admins shall view per-org usage/analytics (documents, storage, tokens, queries).
- **FR-ADM-3** The system shall record an immutable audit log of security-relevant actions.
- **FR-ADM-4** The system shall expose health/readiness endpoints and emit metrics, traces, and structured logs.
- **FR-ADM-5** AI calls shall be traced (LangSmith) and evaluable via an offline eval harness.

## Cross-cutting API (FR-API)

- **FR-API-1** Mutating/upload endpoints shall accept an `Idempotency-Key` and return the original result on replay.
- **FR-API-2** List endpoints shall support cursor pagination, filtering, and sorting.
- **FR-API-3** Hot reads shall be cacheable via Redis with explicit invalidation.
- **FR-API-4** All endpoints shall be rate-limited per user/org and per IP.
- **FR-API-5** The API shall be versioned under `/api/v1` and documented via OpenAPI/Swagger.
