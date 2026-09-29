# Architectural Cartographer Journal

## First-Pass Mental Model

Formbricks is a pnpm/Turbo monorepo whose primary product app is a Next.js App Router application in `apps/web`; shared runtime packages live under `packages/*`; the database package owns Prisma schema, generated client setup, migrations, and seed scripts (`package.json:5-33`, `pnpm-workspace.yaml:1-7`, `apps/web/package.json:1-23`, `packages/database/package.json:32-49`). The product domain is survey experience management: organizations own projects, projects own environments, environments own surveys and contacts, and responses attach to surveys (`packages/database/schema.prisma:571-600`, `packages/database/schema.prisma:617-653`, `packages/database/schema.prisma:655-682`, `packages/database/schema.prisma:147-190`, `packages/database/schema.prisma:344-417`).

## Inspection Discoveries

- The app uses Next.js App Router conventions: route files and layouts live in `apps/web/app`, while feature implementation is usually delegated into `apps/web/modules` and `apps/web/lib` (`apps/web/app/(app)/environments/[environmentId]/surveys/page.tsx:1-4`, `apps/web/modules/survey/list/page.tsx:1-60`).
- The root layout installs Sentry and i18n providers around the entire app (`apps/web/app/layout.tsx:24-43`), while the authenticated app layout checks the NextAuth session and active user status before rendering product UI (`apps/web/app/(app)/layout.tsx:18-45`).
- Browser-side survey listing is powered by React Query, not server component refresh alone (`apps/web/app/(app)/environments/[environmentId]/surveys/layout.tsx:1-8`, `apps/web/app/(app)/environments/[environmentId]/surveys/query-client-provider.tsx:1-10`, `apps/web/modules/survey/list/hooks/use-surveys.ts:25-51`).
- The v3 survey list endpoint is intentionally browser-session or API-key accessible through a shared wrapper (`apps/web/app/api/v3/surveys/route.ts:20-82`, `apps/web/app/api/v3/lib/api-wrapper.ts:126-154`, `apps/web/app/api/v3/lib/auth.ts:79-122`).
- The newer survey list path uses cursor pagination in `survey-page.ts`, while an older cached helper still supports offset pagination in `survey.ts` (`apps/web/modules/survey/list/lib/survey-page.ts:70-94`, `apps/web/modules/survey/list/lib/survey-page.ts:204-267`, `apps/web/modules/survey/list/lib/survey.ts:29-67`).
- Setup is partly documented in the README, but the actionable local flow is reconstructed from scripts: create `.env`, fill generated secrets, start Docker services, migrate, seed, and run dev (`README.md:134-149`, `scripts/setup-dev-env.sh:126-161`, `docker-compose.dev.yml:1-76`, `package.json:21-31`, `packages/database/package.json:36-44`).

## Why These Teaching Anchors

- `apps/web/app/(app)/environments/[environmentId]/surveys/page.tsx:1-4` is tiny but important: it shows route files often delegate to modules.
- `apps/web/modules/survey/list/page.tsx:23-59` is the server-side bridge from URL params and auth to a client component.
- `apps/web/modules/survey/list/components/survey-list.tsx:40-244` is the main client experience: persisted filters, React Query, empty states, errors, pagination, and deletion.
- `apps/web/modules/survey/list/hooks/use-surveys.ts:8-51` shows state with a home and a reason.
- `apps/web/app/api/v3/surveys/route.ts:20-82` is the best simple-but-real backend route.
- `apps/web/app/api/v3/lib/api-wrapper.ts:345-423` is the shared API pipeline: request id, auth, validation, rate limit, audit, handler, response.
- `apps/web/modules/survey/list/lib/survey-page.ts:204-267` explains database pagination without offset.
- `packages/database/schema.prisma:344-417` gives the durable shape of a survey.
- `apps/web/modules/auth/lib/authOptions.ts:165-260` shows security-minded authentication.
- `apps/web/app/api/v2/client/[environmentId]/responses/route.ts:204-278` shows how public survey responses enter the system.

## Where A Junior Might Get Confused

- Route files are sometimes one-line re-exports, so the code that matters may be in `modules` rather than beside the URL file (`apps/web/app/(app)/environments/[environmentId]/surveys/page.tsx:1-4`).
- There are several API generations (`v1`, `v2`, `v3`), and they do not all use the same response envelope or wrapper (`apps/web/app/api/v3/lib/response.ts:136-173`, `apps/web/app/api/v2/client/[environmentId]/responses/route.ts:194-278`).
- `workspaceId` in v3 survey APIs currently resolves to environment/project/org context, so the name is a product abstraction over existing database tables (`apps/web/app/api/v3/lib/auth.ts:30-68`, `apps/web/app/api/v3/surveys/route.ts:36-48`).

## Where A Mid-Level Engineer Should Slow Down

- Authorization checks are layered: route wrapper authenticates, then the handler authorizes against workspace context (`apps/web/app/api/v3/lib/api-wrapper.ts:368-379`, `apps/web/app/api/v3/surveys/route.ts:36-48`).
- Cursor shape and sort order are validated before database access, which prevents mixed-sort pagination bugs (`apps/web/modules/survey/list/lib/survey-page.ts:70-94`).
- Deleting a survey updates the client optimistically and rolls back on mutation error (`apps/web/modules/survey/list/hooks/use-delete-survey.ts:10-33`).

## Where A Senior Engineer Should Be Skeptical

- `Button` forwards a `disabled` prop even when `asChild` renders a non-button element, so callers using `asChild` should verify the resulting element behavior (`apps/web/modules/ui/components/button/index.tsx:44-63`).
- `SurveyStatusIndicator` has duplicate conditional rendering paths for tooltip and non-tooltip modes, increasing drift risk (`apps/web/modules/ui/components/survey-status-indicator/index.tsx:13-98`).
- The v3 API wrapper handles a lot of cross-cutting concerns; that is powerful, but changes there have broad blast radius (`apps/web/app/api/v3/lib/api-wrapper.ts:345-423`).

## Checkpoint Method

For every checkpoint, write a short answer, then grade it against the rubric. Strong answers should name the exact file, describe ownership boundaries, mention at least one failure mode, and explain how the code protects or fails to protect the user.

