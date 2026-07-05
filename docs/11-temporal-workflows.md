# 11 — Temporal Workflow Design

## Principles
- **One workflow per file type.** The dispatcher maps a document's `kind` to a dedicated workflow
  (`PdfIngestWorkflow`, `DocxIngestWorkflow`, `PptxIngestWorkflow`, `XlsxIngestWorkflow`,
  `AudioIngestWorkflow`, `VideoIngestWorkflow`, `ImageIngestWorkflow`, `MarkdownIngestWorkflow`).
- **Shared common activities.** Reusable, coarse-grained activities are defined once and composed
  by each workflow. Activities are kept **few and coarse** (not one micro-activity per line).
- **Durable & idempotent.** Every activity is idempotent (keyed by `document_version_id` + stage)
  so retries never duplicate data.
- **Deterministic workflows.** Workflow code contains only orchestration; all I/O is in activities.

## Task queues & workers
- `ingest-cpu` — validation, parsing, chunking, persistence.
- `ingest-gpu` (or heavy CPU) — GLM OCR, Whisper transcription, reranking, embeddings batching.
- Workers scale horizontally; heavy model work isolated on its own queue for independent scaling.

## Common activities (shared library)
| Activity | Purpose | Idempotency key |
|---|---|---|
| `validate_file` | magic-byte/MIME/size checks | version_id |
| `scan_virus` | ClamAV scan; quarantine on hit | version_id |
| `persist_chunks` | write chunks + tsvector | version_id (upsert by ordinal) |
| `embed_chunks` | batch embeddings → pgvector | chunk_id (upsert) |
| `index_finalize` | mark active, retire old version vectors, set `indexed` | version_id |
| `update_stage_status` | write stage row + publish WS/SSE event | run_id+stage |
| `compensate_cleanup` | delete partial S3/vectors on cancel/failure | run_id |

## Per-type extraction activities
| Workflow | Extraction activity chain |
|---|---|
| PDF/DOCX/PPTX | `extract_text_native` → if no/low text layer → `ocr_glm` → fallback `ocr_vision_llm` |
| Image | `ocr_glm` and/or `caption_vision_llm` |
| Audio | `transcribe_whisper` (timestamped segments) |
| Video | `extract_audio_ffmpeg` → `transcribe_whisper` |
| XLSX/CSV | `normalize_to_parquet` → `register_tabular_dataset` (schema introspection) — **no embedding of prose**; queried later via DuckDB |
| Markdown | `parse_markdown` (structure-aware) |

## Canonical workflow shape (unstructured)
```
IngestDocumentWorkflow(version_id):
  validate_file
  scan_virus
  <type-specific extract>        # OCR / transcribe / parse
  chunk_content                  # structure-aware, overlap, metadata
  embed_chunks                   # batched
  persist_chunks
  index_finalize                 # retire prior version vectors, set status=indexed
  (each step: update_stage_status before/after; heartbeats on long steps)
```

XLSX workflow diverges after `scan_virus`: `normalize_to_parquet` → `register_tabular_dataset` →
`index_finalize` (no chunk/embed).

## Retry & failure policy
- **Per-activity `RetryPolicy`:** exponential backoff (initial 1s, factor 2, max ~60s), max
  attempts tuned per activity (e.g. transcription fewer, transient network more).
- **Non-retryable errors** (unsupported format, virus hit) are raised as
  `ApplicationError(non_retryable=True)` → no retry.
- **DLQ / poison store:** on terminal failure, workflow writes a `dead_letters` row with the
  failed stage, error, and last-good `payload`, then sets document `failed`.
- **Partial resume:** `retry` reads the last successful stage from `ingestion_stages` and starts a
  new workflow run that **skips** completed stages (their outputs already persisted + idempotent).

## Signals & queries
- **Signals:** `cancel` (→ compensating cleanup, status `cancelled`), `pause`, `resume`.
- **Queries:** `get_status()` returns current stage/progress (used by API as a fallback to DB).
- **Heartbeats:** long activities (`ocr_glm`, `transcribe_whisper`, `embed_chunks`) heartbeat so
  stalls trip `heartbeat_timeout` and reschedule; cancellation is checked between heartbeats.

## Batch ingestion (bulk upload)
Optional `BatchIngestWorkflow` spawns one child `*IngestWorkflow` per document and rolls up
progress; per-document durability is preserved by the children. (Milestone M4.)

## Status propagation
Each `update_stage_status` activity writes the `ingestion_stages` row **and** publishes an event
to Redis pub/sub; the API relays to the client via SSE/WS. DB is the source of truth; pub/sub is
the low-latency notifier. Temporal Web UI is available for deep debugging.

## Sequence (ingestion) — see also [17](./17-diagrams.md)
```mermaid
sequenceDiagram
  participant C as Client
  participant API as FastAPI
  participant S3
  participant T as Temporal
  participant W as Worker
  participant DB as Postgres(+pgvector)
  C->>API: complete upload (etags, content_hash)
  API->>DB: dedup check; create document_version
  API->>T: start <Kind>IngestWorkflow(version_id)
  API-->>C: {document_id, ingestion_run_id}
  T->>W: schedule activities
  W->>DB: update_stage_status(validate..index) + publish events
  W->>S3: read object / write parquet
  W->>DB: persist chunks + embeddings
  W->>DB: index_finalize (status=indexed)
  DB-->>C: SSE/WS live stage updates
```

## Testing
- Workflow **replay tests** guard determinism against history.
- Activities unit-tested with mocked ports.
- Failure-injection integration tests assert DLQ + partial-resume behavior.
