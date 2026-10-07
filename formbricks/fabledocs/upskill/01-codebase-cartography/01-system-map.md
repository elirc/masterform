# System Map

## Shape: pnpm + Turborepo monorepo

Workspaces (`pnpm-workspace.yaml`): `apps/*` + `packages/*`. Task graph in `turbo.json`; root scripts in `package.json:12-47`.

```
formbricks/
├── apps/
│   ├── web/          # THE product: Next.js App Router — dashboard UI, all APIs, server actions, pipeline
│   └── storybook/    # component workbench for shared UI
├── packages/
│   ├── database/     # Prisma schema (1070 lines, 36 models), migrations (packages/database/migration/), zod json-types
│   ├── types/        # shared Zod schemas + TS types (the contract library)
│   ├── cache/        # Redis client + CacheService.withCache + typed cache-key registry
│   ├── storage/      # S3 client: presigned uploads, streams, deletion
│   ├── surveys/      # the survey RENDERER (Preact) → built to UMD/ESM, copied to apps/web/public/js/
│   ├── survey-ui/    # UI primitives for the renderer
│   ├── js-core/      # the embeddable browser SDK (@formbricks/js): CommandQueue, widget loader, user/attributes
│   ├── email/        # transactional email templates + sending
│   ├── logger/       # structured logger
│   ├── i18n-utils/   # translation scanning/generation (lingo.dev pipeline)
│   ├── ai / vite-plugins / config-eslint / config-prettier / config-typescript
├── docs/             # Mintlify PRODUCT docs site (not this curriculum)
├── docker/ charts/   # deploy collateral (Docker, Helm)
└── openapi.yml       # generated management-API spec
```

## Inside apps/web

```
apps/web/
├── app/                     # Next.js App Router
│   ├── (app)/environments/[environmentId]/...   # the authenticated dashboard
│   ├── (auth)/              # login/signup pages
│   ├── s/[surveyId]         # public link-survey page (via modules/survey/link)
│   ├── c/                   # contact-scoped survey links
│   ├── api/
│   │   ├── (internal)/pipeline/   # the async fan-out consumer (Flow 2)
│   │   ├── client/[environmentId]/ + v1|v2 client/   # PUBLIC widget APIs (Flows 1,3,7)
│   │   ├── v1|v2|v3/management/   # API-key management APIs (Flow 5)
│   │   └── auth/ billing/ webhooks/ health/
│   └── lib/                 # app-scoped helpers (api response builders, pipelines.ts)
├── modules/                 # feature modules (the real business logic)
│   ├── survey/{editor,list,link,follow-ups,templates,...}
│   ├── auth/                # NextAuth config, login/signup logic
│   ├── ee/                  # ENTERPRISE: license-check, quotas, audit-logs, sso, teams, contacts, billing, 2FA
│   ├── api/v2/              # v2 API framework (wrappers, auth)
│   ├── core/rate-limit/     # Redis Lua rate limiter
│   ├── storage/             # upload orchestration over packages/storage
│   ├── analysis/ environments/ organization/ projects/ integrations/ ui/
├── lib/                     # legacy-ish shared services (survey/service.ts, response/service.ts, constants.ts, action-client)
├── playwright/              # e2e specs
└── locales/                 # i18n message catalogs
```

Rule of thumb: **new feature code lives in `modules/<feature>/` (components + `actions.ts` + `lib/` + `types/`), older shared services live in `lib/`**, and `app/` holds routes that delegate quickly. You'll see both generations; prefer the `modules/` style for new work (matches `AGENTS.md`).

## Runtime topology (who runs where)

```
customer website ──loads── packages/js-core (SDK) ──injects── packages/surveys bundle (widget)
      │                                                      │
      │  GET /api/v1/client/{env}/environment  (config, cached)
      │  POST /api/v2/client/{env}/responses   (answers)
      ▼                                                      ▼
apps/web (Next.js server) ──POST /api/pipeline──▶ same app (webhooks, emails, integrations)
      │                │
   Postgres          Redis            S3 (presigned direct-from-browser uploads)
   (Prisma)          (cache + rate limit)
```

Four JS runtimes exist: Node server (routes/actions), browser dashboard (React 19), customer-page browser (js-core + Preact widget — *not* React-the-dashboard's React), and edge-ish CDN caching in front. Knowing which runtime a file targets is the first question to ask of any bug.

## Ownership map

| Area | Owner (directory) | Public interface |
| --- | --- | --- |
| Survey CRUD + editor | `apps/web/modules/survey/editor` | server actions (`actions.ts`) |
| Response ingestion | `apps/web/app/api/v2/client/.../responses` | public REST |
| Widget config | `.../environment` route + `packages/cache` | public REST (cached) |
| Async fan-out | `apps/web/app/api/(internal)/pipeline` | internal REST (CRON_SECRET) |
| AuthN | `apps/web/modules/auth` (NextAuth) | session cookies; `/api/auth/*` |
| AuthZ | `apps/web/lib/utils/action-client/` + per-route checks | in-process |
| Machine API | `apps/web/app/api/v1|v2|v3/management` | REST + `openapi.yml` |
| Persistence | `packages/database` | Prisma client |
| Cache/rate limit | `packages/cache` + `modules/core/rate-limit` | `withCache`, `checkRateLimit` |
| Files | `packages/storage` + `modules/storage` | presigned URLs |
| Embeddable SDK | `packages/js-core` (+ `packages/surveys`) | npm package + UMD script |
| Enterprise | `apps/web/modules/ee/*` | license-gated internals |

Public vs private: everything under `app/api/client` and `app/api/v*/` is a *published contract* (customers depend on it — see `openapi.yml`); `modules/*/lib/` internals can change freely; `packages/js-core` and the widget bundle are *the most public* surface of all — they run on customer sites and version independently in npm.

Drill: without looking, place these five files in the tree: the Lua rate limiter, the Prisma schema, the survey renderer's response queue, `sendToPipeline`, `authOptions`. Then verify. (All five paths are in this file.)
