# 19 — Future Enhancements

Deferred from v1 to keep scope ordered. The schema and architecture are designed so these slot in
without disruptive rewrites.

## Identity & access
- **SSO / SAML / OIDC federation** (schema already abstracts identity; add `identity_providers` +
  per-org IdP config). Enterprise Google/Microsoft Entra + SP-initiated SAML.
- SCIM user provisioning/deprovisioning.
- Fine-grained sharing links (time-boxed, view-only external shares).

## Ingestion & content
- **Resumable uploads (tus)** for very large media (> 500 MB) and flaky networks.
- **Video visual understanding** (keyframe sampling + vision captions merged with transcript) —
  currently video is transcribed audio-only.
- More formats: HTML, EML/MSG, EPUB, CAD, geospatial.
- Incremental re-ingestion / change-data detection for updated source systems.
- Connectors: Google Drive, SharePoint, S3 buckets, Confluence, Slack.

## Retrieval & AI
- **Query router upgrades:** multi-hop / agentic planning across many datasets.
- **Model routing** by cost/latency/quality; cheaper models for simple queries.
- Self-hosted embeddings option (bge/e5) to remove OpenAI dependency for embeddings.
- **Groundedness re-check** (LLM-judge) gating responses; automatic answer regeneration.
- Citation-level highlighting inside the source viewer (bounding boxes for OCR'd PDFs).
- Cross-project / org-wide knowledge search (with permission-aware fan-out).

## Analytics
- Chart generation from DuckDB results; saved analytical views / dashboards.
- Semantic layer / metric definitions over tabular datasets.
- Joins across multiple spreadsheets/datasets in one question.

## Platform & ops
- Billing/metering (Stripe) with usage-based plans built on `usage_events`.
- Per-org rate/quotas and cost budgets with enforcement.
- Multi-region deployment; DB-per-tenant option for high-isolation customers.
- Admin analytics: retrieval-quality dashboards, drift detection.
- Data residency controls; customer-managed encryption keys (CMEK).

## Collaboration
- Real-time multi-user chat/collaboration; shared threads; comments on documents.
- Notifications center + email/Slack digests for ingestion + activity.

## Mobile
- Native or PWA mobile experience (v1 is responsive web only).
