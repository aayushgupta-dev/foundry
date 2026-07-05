# 13 — Security Architecture

## Threat model (summary)
Assets: tenant documents, embeddings, credentials, tokens. Adversaries: cross-tenant snooping,
credential/token theft, malicious uploads (malware, injection), prompt injection via document
content, and API abuse. Controls below map to these.

## Authentication
- **Passwords:** Argon2id (tuned memory/time cost); never stored/logged in plaintext.
- **Access tokens:** short-lived (~15 min) JWT (HS256/EdDSA), minimal claims (`sub`, `org`, `roles`, `exp`).
- **Refresh tokens:** opaque, hashed at rest, stored per **family**; rotated on every use.
  Reuse of a rotated token ⇒ **theft detection** ⇒ revoke the whole family.
- **Cookies:** refresh token in `HttpOnly; Secure; SameSite=Lax` cookie; CSRF mitigated via
  same-site + double-submit token on state-changing requests.
- **Email flows:** verification + reset tokens are single-use, expiring, stored hashed; reset
  responses never reveal whether an email exists.

## Authorization
- **Tenant isolation (RLS):** every tenant table enforces `organization_id = current_org` via
  Postgres Row-Level Security; the app connects with a non-superuser role so RLS cannot be bypassed.
- **ABAC (Casbin):** every protected action evaluated as `enforce(subject, resource, action, ctx)`;
  org + project roles compile into policies; **deny by default**.
- **Super-admin:** uses a separate, RLS-exempt `service_admin` DB role, reachable only through the
  audited `/admin` surface; impersonation issues a scoped, time-boxed token and is fully logged.

## Data protection
- **In transit:** TLS everywhere; HSTS; internal service traffic on a private network.
- **At rest:** S3 SSE (or MinIO encryption) for objects; encrypted DB volume; secrets never in images.
- **Downloads:** short-lived signed URLs only; buckets are private; no public ACLs.
- **Secrets:** injected via env/secret manager, validated at startup; rotated on compromise;
  `.env` git-ignored; example `.env.sample` committed.

## Upload & file safety
- **Validation:** magic-byte + MIME sniffing against an allowlist; size cap (500 MB); extension/kind consistency check.
- **Virus scanning:** ClamAV activity runs before extraction; infected files → `quarantined`, never processed, flagged to admins.
- **Isolation:** parsing/OCR/transcription run in workers with least privilege; no shell-out to
  untrusted content; temp files namespaced per run and cleaned up.

## AI / prompt-injection defense
- Retrieved and tabular content treated as **untrusted data**, not instructions (instruction hierarchy).
- Context delimited + source-labeled; injection patterns stripped/neutralized pre-prompt.
- Document content can **never** trigger tool calls; only the agent controller invokes tools.
- Generated SQL is validated (read-only, whitelisted datasets, `LIMIT`, timeout) before execution.
- Output must cite only actually-retrieved sources; unfaithful/uncited claims are filtered.

## API security
- Rate limiting (Redis token bucket) per user/org and per IP; stricter on auth + chat.
- Idempotency keys prevent duplicate side effects on retries.
- Strict input validation (pydantic) at the edge; output DTOs prevent overexposure.
- Security headers (CSP, X-Content-Type-Options, Referrer-Policy, HSTS); CORS allowlist.
- No stack traces or internal identifiers leaked in error responses.

## Audit & monitoring
- **Audit log** (append-only) for auth events, membership/role changes, sharing, deletes, purges,
  admin/impersonation, and exports — with actor, resource, IP, timestamp.
- Security-relevant metrics/alerts: auth failure spikes, 403 spikes, DLQ growth, quarantine hits.

## Compliance-minded practices (portfolio-grade)
- Data deletion honored via hard-delete (S3 + vectors + rows).
- PII minimization in logs (no tokens/passwords/full document text in logs).
- Dependency + image scanning in CI; least-privilege IAM for S3/SES.
