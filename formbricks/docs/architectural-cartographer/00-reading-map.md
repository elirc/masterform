# Reading Map

## Mental Model

Formbricks is a survey operations system: organizations hold workspaces/projects, projects hold production and development environments, environments hold surveys, contacts, actions, tags, webhooks, and API-key permissions, and responses flow back into analytics and integrations (`packages/database/schema.prisma:571-600`, `packages/database/schema.prisma:617-653`, `packages/database/schema.prisma:655-682`, `packages/database/schema.prisma:147-190`, `packages/database/schema.prisma:344-417`). The Next.js app exposes product routes in `apps/web/app`, moves feature implementation into `apps/web/modules`, shares backend services in `apps/web/lib`, and persists data through `@formbricks/database` (`apps/web/app/(app)/environments/[environmentId]/surveys/page.tsx:1-4`, `packages/database/src/client.ts:1-20`).

## Top 10 Files To Read In Order

1. `package.json:1-48`  
   Why now: learn the monorepo commands. Understand first: pnpm and Turbo run scripts across workspaces. Explain after reading: which command starts dev, db, tests, and build.

2. `.env.example:9-61`  
   Why now: learn runtime dependencies. Understand first: auth, encryption, SMTP, and Postgres are mandatory local concerns. Explain after reading: why `WEBAPP_URL`, `NEXTAUTH_URL`, `ENCRYPTION_KEY`, `NEXTAUTH_SECRET`, `CRON_SECRET`, and `DATABASE_URL` exist.

3. `docker-compose.dev.yml:1-76`  
   Why now: see local infrastructure. Understand first: the app expects Postgres, MailHog, Valkey, and object storage locally. Explain after reading: which ports support database, email, cache, and file uploads.

4. `apps/web/app/layout.tsx:24-43`  
   Why now: see the root shell. Understand first: all routes inherit this layout. Explain after reading: how locale, i18n, Sentry, and no-script warning wrap the app.

5. `apps/web/app/(app)/layout.tsx:18-45`  
   Why now: see authenticated app setup. Understand first: route groups do not appear in the URL. Explain after reading: how the app uses session and user status before rendering product UI.

6. `apps/web/app/(app)/environments/[environmentId]/surveys/page.tsx:1-4`  
   Why now: understand route delegation. Understand first: App Router `page.tsx` exposes a URL. Explain after reading: why this file is a pointer, not the feature implementation.

7. `apps/web/modules/survey/list/page.tsx:23-59`  
   Why now: see a server component loading domain data. Understand first: server code can call services before rendering. Explain after reading: how environment auth and project lookup become client props.

8. `apps/web/modules/survey/list/components/survey-list.tsx:40-244`  
   Why now: see a real client feature. Understand first: client components can use hooks and local storage. Explain after reading: how filters, loading, empty states, errors, and pagination fit together.

9. `apps/web/app/api/v3/surveys/route.ts:20-82`  
   Why now: trace the matching API. Understand first: Route Handlers export HTTP methods. Explain after reading: how parsing, authorization, database reads, count reads, and response shaping work.

10. `apps/web/modules/survey/list/lib/survey-page.ts:204-267`  
    Why now: learn persistence mechanics. Understand first: cursor pagination reads one extra row. Explain after reading: why `limit + 1`, `nextCursor`, and selected fields matter.

## Three Most Important Data Flows

1. Survey list dashboard: `apps/web/app/(app)/environments/[environmentId]/surveys/page.tsx:1-4` -> `apps/web/modules/survey/list/page.tsx:23-59` -> `apps/web/modules/survey/list/components/survey-list.tsx:89-124` -> `apps/web/modules/survey/list/hooks/use-surveys.ts:25-51` -> `apps/web/modules/survey/list/lib/v3-surveys-client.ts:81-121` -> `apps/web/app/api/v3/surveys/route.ts:20-82` -> `apps/web/modules/survey/list/lib/survey-page.ts:412-440`.

2. Survey creation/editing: `apps/web/modules/survey/components/template-list/actions.ts:42-91` creates surveys through an authenticated server action, while `apps/web/modules/survey/editor/actions.ts:188-248` and `apps/web/modules/survey/editor/actions.ts:250-330` update draft and published survey states.

3. Public response submission: `apps/web/app/api/v2/client/[environmentId]/responses/route.ts:46-80` validates input, `apps/web/app/api/v2/client/[environmentId]/responses/route.ts:98-138` validates submission against the survey, and `apps/web/app/api/v2/client/[environmentId]/responses/route.ts:204-278` creates the response and sends pipeline events.

## Pre-Reading Checklist

1. Can I explain what an organization, project, environment, survey, and response are from the schema?
2. Can I point to the command that starts Docker services?
3. Can I point to the command that starts the Next app?
4. Can I distinguish route files from feature modules?
5. Can I explain why some files start with `"use client"`?
6. Can I identify where a server action validates its input?
7. Can I identify where an API route validates authentication?
8. Can I explain the difference between session auth and API-key auth in v3 routes?
9. Can I name the database model that stores survey responses?
10. Can I find one test that proves survey listing behavior?

## Red Flags Checklist

- A query that fetches surveys without scoping by `environmentId`, because survey rows are environment-owned (`packages/database/schema.prisma:344-352`, `apps/web/modules/survey/list/lib/survey-page.ts:155-164`).
- A count query combined with offset pagination on large survey lists, because the newer list path uses cursor pagination (`apps/web/modules/survey/list/lib/survey-page.ts:70-94`, `apps/web/modules/survey/list/lib/survey-page.ts:204-223`).
- User-facing text not wrapped in `t()`, because visible strings in survey list components use `useTranslation` (`apps/web/modules/survey/list/components/survey-list.tsx:50-56`, `apps/web/modules/survey/list/components/survey-list.tsx:118-124`).
- A route returning raw errors rather than the v3 problem/success helpers (`apps/web/app/api/v3/lib/response.ts:62-94`, `apps/web/app/api/v3/lib/response.ts:136-173`).
- API-key access that skips environment permission checks (`apps/web/app/api/v3/lib/auth.ts:95-110`).
- A mutation that updates server state without invalidating or updating React Query cache (`apps/web/modules/survey/list/hooks/use-delete-survey.ts:10-33`).
- Date display using browser defaults instead of locale-aware helpers (`apps/web/modules/survey/list/components/survey-card.tsx:6-9`, `apps/web/modules/survey/list/components/survey-card.tsx:82-87`).
- A change to `withV3ApiWrapper` without API wrapper tests, because it gates auth, validation, rate limits, audit, and error handling (`apps/web/app/api/v3/lib/api-wrapper.ts:345-423`).
- A change to auth callbacks without considering sign-in audit logging (`apps/web/app/api/auth/[...nextauth]/route.ts:62-160`).

