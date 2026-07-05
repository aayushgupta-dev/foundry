# 09 — Frontend Architecture

## Stack (pinned — see [02](./02-tech-stack-and-versions.md))
Next.js 16 (App Router, RSC, Turbopack, React Compiler) · React 19.2 · TypeScript 6 ·
Tailwind CSS 4 · shadcn/ui + Radix · **TanStack Query 5** (server state) ·
**Zustand 5** (client/UI state) · **nuqs** (URL state) · Node 24 LTS runtime.

## State management strategy (researched, 2026 best practice)
The 2026 consensus for Next.js App Router is to **separate server state from client state** and
not manage async data in a global client store:

- **Server state → TanStack Query.** All API data: caching, dedup, invalidation, retries,
  optimistic updates, and **polling/refetch** of ingestion status. Query keys are namespaced by
  org/project so cache invalidation is precise.
- **Client/UI state → Zustand.** Ephemeral UI: modals, drawers, selected items, chat composer
  draft, theme, command palette. Small, typed slices; **no** duplication of server data.
- **URL state → nuqs.** Filters, sort, pagination cursors, active tab — shareable/bookmarkable
  and SSR-friendly via `searchParams`.
- **Server Components (RSC)** for initial data-heavy, read-only renders (dashboards, doc lists),
  hydrating into client components for interactivity. Mutations via Server Actions or Query
  mutations calling the API.
- **Realtime** (ingestion progress, streamed chat) via SSE/WebSocket hooks that write into Query
  cache / a Zustand slice — never a third store duplicating the same data.

> Redux Toolkit is intentionally **not** used: over-engineered for this app in 2026. Jotai
> considered but unnecessary given the Query+Zustand split.

## Folder structure
```
frontend/
├── package.json            # pinned deps
├── next.config.ts
├── tailwind.config / app/globals.css   # Tailwind v4 CSS-first config
├── Dockerfile              # multi-stage; node:24
├── src/
│   ├── app/                # App Router
│   │   ├── (auth)/         # login, invite, verify, reset
│   │   ├── (app)/          # authenticated shell
│   │   │   ├── dashboard/
│   │   │   ├── workspaces/[workspaceId]/
│   │   │   ├── projects/[projectId]/
│   │   │   │   ├── documents/        # list, upload, [documentId], versions, trash
│   │   │   │   ├── chat/[threadId]/
│   │   │   │   └── settings/
│   │   │   └── settings/            # org, profile, usage
│   │   ├── (admin)/        # super-admin console
│   │   ├── api/            # route handlers (proxy/SSE bridge if needed)
│   │   └── layout.tsx      # providers (Query, theme), error boundary
│   ├── components/         # ui/ (shadcn), features/, layout/
│   ├── features/           # feature modules: auth, documents, upload, chat, admin
│   │   └── <feature>/{api.ts, hooks.ts, components/, store.ts, types.ts}
│   ├── lib/                # api-client (fetch wrapper w/ auth refresh), sse.ts, ws.ts, utils
│   ├── stores/             # zustand slices (ui, theme, chat-composer)
│   └── hooks/              # shared hooks (useAuth, useOrg, useToast)
└── tests/                  # vitest + testing-library; playwright e2e
```

## Data fetching & auth
- Central `apiClient` (fetch) attaches the access token, transparently refreshes on 401 via the
  rotating refresh cookie, and retries once; on refresh failure → redirect to login.
- TanStack Query wraps all reads; mutations invalidate the relevant keys.
- SSR/RSC reads use a server-side fetch with cookies forwarded.

## Key screens (components)
Auth (login/invite/verify/reset) · Dashboard (usage + recent activity, RSC) · Workspaces/Projects
navigation · Documents table (`@tanstack/react-table`, filters via nuqs) · Upload dropzone
(presigned multipart, parallel parts, progress) · Ingestion status card (SSE-driven pills) ·
Document detail + version history + source viewer (PDF/slide/media with citation deep-links) ·
Chat (thread list, streaming message renderer, tool-step panel, citation chips, sources panel,
tabular result table/chart) · Admin console (orgs table, org detail, impersonation banner) ·
Settings (members/roles, org, profile, theme).

## Realtime rendering
- **Chat streaming:** SSE reader appends tokens to the active assistant message; tool steps
  render as they arrive; citations resolve to a sources panel.
- **Ingestion progress:** subscribe per-document (WS/SSE); update the status pill + progress in
  the Query cache so lists and detail stay consistent.

## Accessibility, theming, responsiveness
- Radix/shadcn primitives give ARIA + keyboard support; visible focus rings; live regions for
  streaming completion and status changes.
- Tailwind v4 `dark:` variants; theme persisted in Zustand + `prefers-color-scheme` fallback.
- Responsive layouts (sidebar collapses to a drawer on mobile).

## Quality
- TS strict; ESLint + Prettier; component tests (Vitest + Testing Library); e2e (Playwright)
  covering login → upload → chat.
