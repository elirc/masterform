# Senior Engineer Guide

## Architectural Critique

- Scalability: 4/5. The survey list uses cursor pagination and scoped queries (`apps/web/modules/survey/list/lib/survey-page.ts:204-223`), while older offset helpers still exist and should not be copied for large lists (`apps/web/modules/survey/list/lib/survey.ts:29-67`).
- TypeScript discipline: 4/5. Zod inference, Prisma select payloads, and generic API wrapper types are strong (`apps/web/modules/survey/list/types/survey-overview.ts:1-39`, `apps/web/modules/survey/list/lib/survey-record.ts:5-22`, `apps/web/app/api/v3/lib/api-wrapper.ts:28-58`).
- Separation of concerns: 4/5. Route files delegate to modules, and API cross-cutting concerns live in v3 wrapper (`apps/web/app/(app)/environments/[environmentId]/surveys/page.tsx:1-4`, `apps/web/app/api/v3/lib/api-wrapper.ts:345-423`).
- Testability: 4/5. Survey list route, cursor service, and hooks have direct tests (`apps/web/app/api/v3/surveys/route.test.ts:91-372`, `apps/web/modules/survey/list/lib/survey-page.test.ts:49-334`, `apps/web/modules/survey/list/hooks/use-surveys.test.ts:20-184`).
- Maintainability: 3/5. Some components duplicate state/rendering branches, especially status indicator tooltip and non-tooltip modes (`apps/web/modules/ui/components/survey-status-indicator/index.tsx:13-98`).
- Security posture: 4/5. Auth validates password length, constant-time password verification, account active state, and API-key permission scoping (`apps/web/modules/auth/lib/authOptions.ts:190-260`, `apps/web/app/api/v3/lib/auth.ts:95-110`).
- Performance: 4/5. The list endpoint runs page and count in parallel and avoids count on subsequent pages (`apps/web/app/api/v3/surveys/route.ts:49-58`, `apps/web/modules/survey/list/hooks/use-surveys.ts:30-36`).

## Performance Audit

Finding: `getSurveys` still exposes offset pagination with `skip`, which is risky for large survey sets (`apps/web/modules/survey/list/lib/survey.ts:29-67`). Prefer the cursor path used by v3 list.

```ts
// Prefer the cursor pattern in apps/web/modules/survey/list/lib/survey-page.ts:204-223
return prisma.survey.findMany({
  where: buildBaseWhere(environmentId, filterCriteria, { ...cursorWhere }), // Environment scoped.
  select: surveySelect, // Minimal selected fields.
  orderBy: getSurveyOrderBy(sortBy), // Stable sort.
  take: limit + 1, // One extra row tells us if there is a next page.
});
```

Finding: response counts are grouped by survey ids after page rows load, avoiding per-card count queries (`apps/web/modules/survey/list/lib/survey-record.ts:23-55`). Preserve this pattern.

## Security Audit

Finding: v3 API keys are checked against environment permission after workspace context resolution (`apps/web/app/api/v3/lib/auth.ts:95-110`). Any new v3 workspace route should call `requireV3WorkspaceAccess` after wrapper authentication.

```ts
// Pattern from apps/web/app/api/v3/surveys/route.ts:36-48
const authResult = await requireV3WorkspaceAccess(authentication, workspaceId, "read", requestId, instance);
if (authResult instanceof Response) return authResult; // Generic 403 avoids resource existence leaks.
const { environmentId } = authResult; // Internal id is safe after authorization.
```

Finding: credentials auth limits password length before bcrypt work and verifies a control hash even when the user is missing to reduce enumeration timing risk (`apps/web/modules/auth/lib/authOptions.ts:203-235`). Do not "simplify" that block.

## TypeScript Discipline Review

The codebase is strongest where runtime schemas and static types share a source, as in survey overview filters (`apps/web/modules/survey/list/types/survey-overview.ts:1-39`). It is weaker where complex wrappers need `any`, such as NextAuth wrapping around callback params (`apps/web/app/api/auth/[...nextauth]/route.ts:35-43`). That `any` may be pragmatic, but changes around it deserve tests because type safety is lower.

## Custom Abstractions Inventory

- `withV3ApiWrapper`: auth, validation, rate limit, audit, request id, error handling (`apps/web/app/api/v3/lib/api-wrapper.ts:345-423`).
- `authenticatedActionClient`: safe-action wrapper that attaches authenticated user to action context (`apps/web/lib/utils/action-client/index.ts:51-65`).
- `withAuditLogging`: used around authenticated server actions like survey creation/editing (`apps/web/modules/survey/components/template-list/actions.ts:42-91`, `apps/web/modules/survey/editor/actions.ts:188-330`).
- `surveySelect` plus mappers: list-query projection and transformation (`apps/web/modules/survey/list/lib/survey-record.ts:5-55`).
- `SurveysQueryClientProvider`: route-local React Query client for survey list routes (`apps/web/app/(app)/environments/[environmentId]/surveys/query-client-provider.tsx:1-10`).

## Testing Assessment

Existing tests cover route authorization/contract, cursor behavior, and client hook behavior (`apps/web/app/api/v3/surveys/route.test.ts:91-372`, `apps/web/modules/survey/list/lib/survey-page.test.ts:49-334`, `apps/web/modules/survey/list/hooks/use-surveys.test.ts:20-184`). A risky behavior worth adding is that corrupted local-storage filters should be removed before fetching.

Runnable test sketch:

```tsx
// Suggested location: apps/web/modules/survey/list/components/survey-list.test.tsx
// This is intentionally not committed because project guidance prefers not testing .tsx with Vitest.
// Use Playwright instead for component workflows, per AGENTS testing guidance.
test("removes invalid stored survey filters before fetching", async ({ page }) => {
  await page.addInitScript(() => {
    localStorage.setItem("formbricks-surveys-filters", "{not-json");
  });
  await page.goto("/environments/<envId>/surveys");
  await expect(page.getByText("Surveys")).toBeVisible();
  const value = await page.evaluate(() => localStorage.getItem("formbricks-surveys-filters"));
  expect(value).not.toBe("{not-json");
});
```

The reason this is Playwright-shaped is that repository guidance says not to write Vitest tests for `.tsx` components; React components are covered by Playwright E2E tests.

## Bug Injection Exercise

1. Symptom: The dashboard shows duplicate or missing surveys after clicking "Load more." Test scenario: seed surveys with identical `updatedAt` values and verify stable id tie-breaker; inspect cursor logic (`apps/web/modules/survey/list/lib/survey-page.ts:108-153`).
2. Symptom: A read-only user can open a draft survey edit URL from a list card. Test scenario: render a draft survey as read-only and assert no link wraps the card (`apps/web/modules/survey/list/components/survey-card.tsx:94-115`).
3. Symptom: API clients receive `200` with internal `environmentId`. Test scenario: route test should assert serializer exposes `workspaceId` and omits internal fields (`apps/web/app/api/v3/surveys/serializers.ts:11-18`, `apps/web/app/api/v3/surveys/route.test.ts:325-354`).
4. Symptom: Survey responses are accepted for the wrong environment. Test scenario: submit a response where `environmentId` and survey environment differ; inspect `checkSurveyValidity` path from response route (`apps/web/app/api/v2/client/[environmentId]/responses/route.ts:98-138`).
5. Symptom: A new v3 route returns inconsistent error shapes. Test scenario: force validation failure and assert `application/problem+json` body (`apps/web/app/api/v3/lib/response.ts:26-94`).

## Git History Learning Exercise

1. `feat: add cursor pagination to v3 survey list (#xxxx)` implies offset listing was not enough for scale and route contract changed around `nextCursor`.
2. `fix: hide resource existence in v3 workspace auth (#xxxx)` implies prior routes may have leaked 404/403 distinctions; review `problemForbidden` use (`apps/web/app/api/v3/lib/auth.ts:70-75`).
3. `chore: centralize v3 response envelopes (#xxxx)` implies clients needed consistent `{ data, meta }` and problem details (`apps/web/app/api/v3/lib/response.ts:136-173`).
4. `fix: preserve previous survey data during filter refetch (#xxxx)` implies UX flicker or loading regression; inspect `keepPreviousData` (`apps/web/modules/survey/list/hooks/use-surveys.ts:25-40`).
5. `security: harden credentials authorization timing (#xxxx)` implies enumeration or CPU DoS concern; inspect password length and control hash logic (`apps/web/modules/auth/lib/authOptions.ts:203-235`).

## If I Owned This Codebase

- Consolidate survey status rendering: effort S, impact M. Remove duplicate branches in `SurveyStatusIndicator` (`apps/web/modules/ui/components/survey-status-indicator/index.tsx:13-98`).
- Add a v3 route template/generator: effort M, impact M. New routes should consistently use wrapper, auth, response helpers, and tests (`apps/web/app/api/v3/lib/api-wrapper.ts:345-423`).
- Retire or clearly mark offset list helpers: effort M, impact H. Avoid future large-list regressions from copying `skip` patterns (`apps/web/modules/survey/list/lib/survey.ts:29-67`).
- Document workspace/environment naming transition: effort S, impact M. v3 surfaces `workspaceId` while database still uses project/environment models (`apps/web/app/api/v3/lib/auth.ts:30-68`, `packages/database/schema.prisma:571-653`).
- Add Playwright coverage for survey list local-storage filters and deletion rollback: effort M, impact M. Current hook tests cover behavior, but user-visible flow needs browser proof (`apps/web/modules/survey/list/hooks/use-delete-survey.ts:10-33`).

