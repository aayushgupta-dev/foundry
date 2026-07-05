# 17 — Diagrams

All diagrams are Mermaid so they render in GitHub/most viewers and are agent-readable.

## System context / component diagram
```mermaid
flowchart TB
  subgraph Client
    FE[Next.js 16 App Router<br/>TanStack Query + Zustand]
  end
  subgraph Edge
    LB[Load Balancer / CDN]
  end
  subgraph Backend
    API[FastAPI /api/v1<br/>SSE + WS]
    WCPU[Temporal Worker: CPU]
    WGPU[Temporal Worker: OCR/Whisper/Rerank/Embed]
    TMP[Temporal Server + UI]
  end
  subgraph Data
    PG[(PostgreSQL 18<br/>+ pgvector 0.8.2<br/>relational+vector+FTS)]
    RDS[(Redis 8)]
    S3[(S3 / MinIO<br/>files + Parquet)]
  end
  subgraph Models
    OAI[OpenAI Responses + Embeddings]
    OCR[GLM OCR self-hosted]
    WHS[Whisper self-hosted]
    RRK[Reranker self-hosted]
    DDB[DuckDB embedded]
  end
  subgraph Obs
    OTEL[OTel Collector] --> PRM[Prometheus] & TMPO[Tempo] & LOKI[Loki]
    PRM & TMPO & LOKI --> GRAF[Grafana]
    GT[GlitchTip]
    LS[LangSmith]
  end
  FE --> LB --> API
  API --> PG & RDS & S3
  API --> TMP
  TMP --> WCPU & WGPU
  WCPU --> PG & S3
  WGPU --> PG & S3 & OCR & WHS & RRK & OAI
  API --> OAI & RRK & DDB
  DDB --> S3
  API & WCPU & WGPU --> OTEL
  API & WGPU --> LS
  API & FE --> GT
```

## ER diagram
See [07 — Database Schema](./07-database-schema.md) for the full ER diagram and DDL.

## Ingestion sequence (durable)
```mermaid
sequenceDiagram
  autonumber
  participant C as Client
  participant API as FastAPI
  participant S3 as S3/MinIO
  participant T as Temporal
  participant W as Worker
  participant DB as Postgres(+pgvector)
  C->>API: POST /uploads (filename,size,mime)
  API-->>C: presigned multipart URLs
  C->>S3: upload parts (direct)
  C->>API: POST /uploads/{id}/complete (etags, content_hash)
  API->>DB: dedup check + create document_version
  API->>T: start <Kind>IngestWorkflow(version_id)
  API-->>C: {document_id, ingestion_run_id}
  loop each stage
    T->>W: schedule activity (retry policy + heartbeat)
    W->>DB: update_stage_status + publish event
    W-->>C: SSE/WS live update
  end
  W->>S3: read object / write parquet
  W->>DB: persist chunks + embeddings (idempotent)
  W->>DB: index_finalize (status=indexed)
  Note over W,DB: on terminal failure → dead_letters (DLQ); retry resumes from last good stage
```

## Chat (agentic RAG) sequence
```mermaid
sequenceDiagram
  autonumber
  participant C as Client
  participant API as FastAPI
  participant AG as Chat Agent
  participant R as Retrieval (pgvector+BM25+rerank)
  participant SQL as DuckDB text-to-SQL
  participant M as OpenAI Responses
  C->>API: POST /conversations/{id}/messages (SSE)
  API->>AG: run(question, memory)
  AG->>AG: router: semantic | analytical | hybrid
  alt semantic
    AG->>R: hybrid retrieve (RLS+filters)
    R-->>AG: top-k chunks + metadata
  else analytical
    AG->>SQL: introspect schema → generate+validate SQL → execute
    SQL-->>AG: rows + SQL used
  end
  AG->>M: grounded prompt + context (stream)
  M-->>API: token stream
  API-->>C: event: token / tool_call / tool_result / citation / done
  API->>API: validate citations map to retrieved sources
```

## Auth token rotation
```mermaid
sequenceDiagram
  participant C as Client
  participant API
  participant DB
  C->>API: login
  API->>DB: create refresh family (hashed)
  API-->>C: access JWT + refresh cookie
  C->>API: /auth/refresh (cookie)
  API->>DB: validate + rotate (mark old replaced_by new)
  API-->>C: new access + new refresh
  Note over API,DB: if a rotated token is reused → revoke whole family (theft)
```

## Class diagram (backend core, simplified)
```mermaid
classDiagram
  class DocumentService {
    +create_upload()
    +complete_upload()
    +reupload()
    +soft_delete()
    +purge()
  }
  class IngestionService {
    +start_workflow(version)
    +get_status(run)
    +retry(run)
    +cancel(run)
  }
  class RetrievalService {
    +hybrid_search(query, filters) List~Chunk~
  }
  class ChatService {
    +stream_answer(conversation, question)
  }
  class AnalyticsService {
    +text_to_sql(question, datasets)
  }
  class DocumentRepository
  class ChunkRepository
  class VectorRepository
  class PolicyEngine
  class ObjectStorage
  class LLMClient
  class EmbeddingClient
  DocumentService --> DocumentRepository
  DocumentService --> ObjectStorage
  DocumentService --> PolicyEngine
  IngestionService --> DocumentRepository
  RetrievalService --> VectorRepository
  RetrievalService --> ChunkRepository
  RetrievalService --> EmbeddingClient
  ChatService --> RetrievalService
  ChatService --> AnalyticsService
  ChatService --> LLMClient
```

## Deployment / infrastructure (prod)
```mermaid
flowchart TB
  U[Users] --> V[Vercel: Next.js]
  U --> CDN[CDN/WAF]
  V --> APILB[API Load Balancer]
  subgraph Cloud[Render / AWS]
    APILB --> API1[API tasks x N]
    WQ1[worker-cpu x N]
    WQ2[worker-gpu x M]
    TC[Temporal Cloud / self-hosted]
    API1 --> TC --> WQ1 & WQ2
    API1 --> RDSPG[(RDS Postgres 18 + pgvector)]
    API1 --> EC[(ElastiCache Redis)]
    API1 --> S3P[(S3)]
    WQ2 --> OCRS[OCR svc] & WHSS[Whisper svc] & RRKS[Reranker svc]
  end
  API1 --> OAI[OpenAI API]
  Cloud --> OBS[Grafana LGTM + GlitchTip]
```
