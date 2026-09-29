# Reference Artifacts

### Mission 25: Write the Docs That Don't Exist

**Tier:** Senior  
**Time Estimate:** 90 minutes  
**Goal:** Convert code-reading into reusable operating references for future work.  
**The Concept:** The best product owners do not only understand the survey system; they leave trail markers for the next person under pressure.  
**Design Intent Before You Read the Code:** Workflow docs should be actionable, line-backed, and tied to how this repo actually works.  
**Find It In The Code:** Use the references below as your source map.

```text
package.json:18-31                                  # Core scripts.
apps/web/app/(app)/environments/[environmentId]/surveys/page.tsx:1-4
apps/web/modules/survey/list/page.tsx:23-59
apps/web/modules/survey/list/components/survey-list.tsx:40-244
apps/web/app/api/v3/surveys/route.ts:20-82
apps/web/modules/survey/list/lib/survey-page.ts:204-267
packages/database/schema.prisma:344-417
```

**The Aha Moment:** Documentation is engineering when it changes how safely people act.  
**Socratic Checkpoint:** What workflow causes most mistakes? Which line ranges prove your claims? What should be a checklist instead of prose? What belongs in onboarding versus review? What should you avoid documenting because it will drift?  
**How to self-grade:** Strong answers produce action-oriented docs that cite files, mention risks, and help someone make a change without guessing.  
**Connects To:** Future feature work in `docs/user-story-build-path/`.

## Doc 1: Junior Onboarding Checklist

- Install dependencies with `pnpm install`; root scripts define workspace commands (`package.json:18-31`).
- Run `pnpm dev:setup` to create `.env` and generate required secrets (`package.json:46-48`, `scripts/setup-dev-env.sh:126-161`).
- Start local services with `pnpm db:up`; Docker provides Postgres, MailHog, Valkey, and RustFS (`package.json:24-25`, `docker-compose.dev.yml:1-76`).
- Apply database setup with `pnpm db:migrate:dev` and optionally seed with `pnpm db:seed` (`package.json:20-23`, `packages/database/package.json:36-44`).
- First PR checklist: identify route file, module file, API/service file, schema/type file, and test file before editing.

## Doc 2: Architecture Guide for New Engineers

Navigate by ownership:

- URL and layout: `apps/web/app`, such as survey list route and layout (`apps/web/app/(app)/environments/[environmentId]/surveys/page.tsx:1-4`, `apps/web/app/(app)/environments/[environmentId]/surveys/layout.tsx:1-8`).
- Feature behavior: `apps/web/modules`, such as survey list server page and client list (`apps/web/modules/survey/list/page.tsx:23-59`, `apps/web/modules/survey/list/components/survey-list.tsx:40-244`).
- Shared backend utilities: `apps/web/lib` and API libraries, such as safe actions and v3 wrapper (`apps/web/lib/utils/action-client/index.ts:14-65`, `apps/web/app/api/v3/lib/api-wrapper.ts:345-423`).
- Persistence: `packages/database`, with Prisma schema and client (`packages/database/schema.prisma:4-17`, `packages/database/src/client.ts:1-20`).

## Doc 3: Code Review Checklist

- Does every database read scope by tenant boundary, such as `environmentId` for surveys (`apps/web/modules/survey/list/lib/survey-page.ts:155-164`)?
- Does every v3 route use wrapper response helpers and auth where needed (`apps/web/app/api/v3/surveys/route.ts:20-82`, `apps/web/app/api/v3/lib/response.ts:62-149`)?
- Does UI text use `t()` and dates use locale-aware helpers (`apps/web/modules/survey/list/components/survey-card.tsx:31-45`, `apps/web/modules/survey/list/components/survey-card.tsx:82-87`)?
- Does client server-state mutation update or invalidate React Query cache (`apps/web/modules/survey/list/hooks/use-delete-survey.ts:10-33`)?
- Do tests cover the changed layer (`apps/web/app/api/v3/surveys/route.test.ts:91-372`, `apps/web/modules/survey/list/lib/survey-page.test.ts:49-334`)?

## Doc 4: Debugging Playbook

Survey dashboard blank:

1. Confirm route delegation (`apps/web/app/(app)/environments/[environmentId]/surveys/page.tsx:1-4`).
2. Check server page project lookup/auth/redirect (`apps/web/modules/survey/list/page.tsx:28-39`).
3. Check QueryClient provider (`apps/web/app/(app)/environments/[environmentId]/surveys/query-client-provider.tsx:1-10`).
4. Check filters and `enabled` state (`apps/web/modules/survey/list/components/survey-list.tsx:55-105`).
5. Inspect API URL generation (`apps/web/modules/survey/list/lib/v3-surveys-client.ts:38-79`).
6. Inspect API auth and route response (`apps/web/app/api/v3/surveys/route.ts:20-82`).

Response submission failing:

1. Validate environment id/body path (`apps/web/app/api/v2/client/[environmentId]/responses/route.ts:46-80`).
2. Validate survey rules (`apps/web/app/api/v2/client/[environmentId]/responses/route.ts:98-138`).
3. Inspect create and pipeline events (`apps/web/app/api/v2/client/[environmentId]/responses/route.ts:234-268`).

## Doc 5: Change Playbook

Branch -> inspect -> implement -> verify -> review:

1. Find route and feature module. Example: survey list route re-exports module page (`apps/web/app/(app)/environments/[environmentId]/surveys/page.tsx:1-4`).
2. Find type/schema contract. Example: overview filters and list item schema (`apps/web/modules/survey/list/types/survey-overview.ts:1-39`).
3. Find data path. Example: API client, route, and Prisma pagination (`apps/web/modules/survey/list/lib/v3-surveys-client.ts:81-121`, `apps/web/app/api/v3/surveys/route.ts:20-82`, `apps/web/modules/survey/list/lib/survey-page.ts:204-267`).
4. Implement smallest coherent change.
5. Run targeted tests first, then broader tests as risk requires (`package.json:27-31`).
6. Review against tenant scoping, i18n, dates, auth, caching, and tests.

## Doc 6: Senior Ownership Notes

Monitor these areas:

- v3 API wrapper blast radius (`apps/web/app/api/v3/lib/api-wrapper.ts:345-423`).
- Auth hardening assumptions (`apps/web/modules/auth/lib/authOptions.ts:203-260`).
- Cursor pagination correctness (`apps/web/modules/survey/list/lib/survey-page.ts:70-94`, `apps/web/modules/survey/list/lib/survey-page.ts:204-267`).
- Duplicated UI status logic (`apps/web/modules/ui/components/survey-status-indicator/index.tsx:13-98`).
- Public response validation and pipeline events (`apps/web/app/api/v2/client/[environmentId]/responses/route.ts:204-278`).

Improve carefully:

- Consolidate duplicated status rendering.
- Prefer cursor pagination over offset list helpers.
- Add workflow-level Playwright coverage for survey list filters and deletion.
- Keep generic auth failure messages for existence-sensitive resources.

## Doc 7: Interview Walkthrough

Practice answer:

"Formbricks is a Next.js App Router survey platform in a pnpm/Turbo monorepo. The main app is `apps/web`, and durable data lives behind Prisma in `packages/database` (`package.json:5-33`, `apps/web/package.json:1-23`, `packages/database/schema.prisma:4-17`). A representative flow is survey list: the route file delegates to a module page, the server page loads project/auth/locale, the client component initializes filters and React Query, the API client calls `/api/v3/surveys`, the v3 wrapper handles auth/rate-limit/audit, workspace access resolves environment context, Prisma reads cursor-paginated surveys, and the UI renders cards (`apps/web/app/(app)/environments/[environmentId]/surveys/page.tsx:1-4`, `apps/web/modules/survey/list/page.tsx:23-59`, `apps/web/modules/survey/list/components/survey-list.tsx:40-244`, `apps/web/app/api/v3/surveys/route.ts:20-82`, `apps/web/modules/survey/list/lib/survey-page.ts:204-267`)."
