# 06 — UX Flows

Primary end-to-end flows. Detailed sequence diagrams are in [17 — Diagrams](./17-diagrams.md).

## Navigation model

```
Login / Invite / Verify
        │
        ▼
Dashboard (org overview: recent projects, ingestion activity, usage)
        │
        ▼
Workspace list ──► Workspace ──► Project list ──► Project
                                                     │
             ┌───────────────────────────────────────┼───────────────────────────┐
             ▼                                         ▼                           ▼
     Documents (folders, tags,               Chat (threads, streaming            Project settings
     upload, status, versions)               answers, citations, sources)        (members, roles, archive)
Admin console (super-admin): Orgs ► Org detail ► Usage ► Impersonate (audited)
```

## Flow 1 — Invite → first answer (happy path)
1. Admin invites user by email + role → invite email sent.
2. User clicks invite → sets password → verifies email.
3. User lands on dashboard → opens a project.
4. User drags a PDF into the upload zone.
5. Client requests presigned multipart URLs → uploads directly to S3 → notifies API.
6. Ingestion workflow starts; UI shows a live per-stage progress card.
7. On `indexed`, the document appears as ready.
8. User opens Chat, asks a question → answer streams with inline citations.
9. User clicks a citation → source viewer opens at the exact page.

## Flow 2 — Upload with live progress
- Drag-drop or file picker → validation (type/size) client-side.
- Presigned multipart: parts upload in parallel with an aggregate progress bar; resumable on transient part failure.
- After completion, a document row shows status pill: `Queued → Scanning → Extracting → Chunking → Embedding → Indexing → Ready` (or `Failed`).
- Status updates arrive via WebSocket/SSE (no manual refresh).
- Failure shows a red pill + reason + **Retry** (resumes from failed stage) and **Cancel**.

## Flow 3 — Chat (agentic)
- User types a question in a thread.
- The agent may: retrieve passages (semantic), run DuckDB SQL (analytical), or both (router decides).
- Tokens stream in; a collapsible "reasoning/steps" panel shows tool calls (retrieval hits, SQL executed).
- Answer renders with inline citation chips `[1] [2]`; a sources panel lists each with a deep link.
- For tabular answers, results render as a small table/chart plus the SQL used (expandable).
- If context is insufficient, the assistant clearly says so and suggests uploading relevant docs.

## Flow 4 — Manage documents
- List with columns: name, type, status, version, size, tags, updated.
- Actions: view source, view versions, re-upload (new version), move (folder), tag, share, archive, delete (→ trash).
- Trash view: restore or permanently delete.

## Flow 5 — Admin console (super-admin)
- Orgs table (name, members, storage, docs, last active).
- Org detail: usage charts, member list, ingestion health.
- Impersonate (with confirmation) → banner indicates impersonation → all actions audited.

## Key screens (inventory)
Auth: Login, Invite-accept, Verify-email, Forgot/Reset password.
App: Dashboard, Workspaces, Projects, Documents (+ folders/tags), Upload, Document detail/versions, Trash, Chat (threads + composer + sources), Project settings/members, Org settings, Usage analytics, Profile/theme.
Admin: Orgs list, Org detail, Usage, Impersonation.
System states: loading skeletons, empty states, error boundaries, toasts, 403/404 pages.

## Accessibility & theming
- Keyboard-first: all actions reachable without a mouse; visible focus rings.
- ARIA roles on interactive components (via Radix/shadcn primitives).
- Live regions announce streaming answer completion and status changes.
- Dark/light theme toggle persisted (client state), respects `prefers-color-scheme`.
