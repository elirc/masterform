# Mid-Level Engineer Guide

## Architecture Diagram

```text
Browser
  |
  | /environments/:environmentId/surveys
  v
Next App Router page
  apps/web/app/(app)/environments/[environmentId]/surveys/page.tsx:1-4
  |
  v
Server page loads project/auth/locale
  apps/web/modules/survey/list/page.tsx:23-59
  |
  v
Client component with filters + React Query
  apps/web/modules/survey/list/components/survey-list.tsx:40-244
  apps/web/modules/survey/list/hooks/use-surveys.ts:8-51
  |
  | GET /api/v3/surveys?workspaceId=...
  v
Route handler + wrapper
  apps/web/app/api/v3/surveys/route.ts:20-82
  apps/web/app/api/v3/lib/api-wrapper.ts:345-423
  |
  v
Workspace auth + pagination service
  apps/web/app/api/v3/lib/auth.ts:79-122
  apps/web/modules/survey/list/lib/survey-page.ts:412-440
  |
  v
Prisma/Postgres
  packages/database/schema.prisma:344-417
  packages/database/schema.prisma:147-190
```

## Type System Deep Dive

The survey overview filter contract starts as Zod, then becomes TypeScript through inference (`apps/web/modules/survey/list/types/survey-overview.ts:1-39`). This matters because the same shape controls local storage normalization, query-string generation, React Query keys, and backend filter parsing (`apps/web/modules/survey/list/components/survey-list.tsx:55-87`, `apps/web/modules/survey/list/lib/v3-surveys-client.ts:38-79`, `apps/web/modules/survey/list/hooks/use-surveys.ts:19-40`).

The database row contract is safer than a raw Prisma model because `surveySelect` defines the exact columns and relations loaded for the list, and `TSurveyRow` is inferred from that select (`apps/web/modules/survey/list/lib/survey-record.ts:5-22`). If a UI asks for a field not selected there, TypeScript should complain before runtime.

```ts
// apps/web/modules/survey/list/lib/survey-record.ts:5-22
export const surveySelect = {
  id: true,
  createdAt: true,
  updatedAt: true,
  name: true,
  creator: { select: { name: true } }, // Pull only the nested field the card needs.
  status: true,
  singleUse: true,
  environmentId: true,
} satisfies Prisma.SurveySelect; // Compile-time check against Prisma's legal select shape.

export type TSurveyRow = Prisma.SurveyGetPayload<{ select: typeof surveySelect }>;
```

## State Management Deep Dive

Survey list state has three homes:

- URL route state: `environmentId` comes from the App Router param and is passed by the server page (`apps/web/modules/survey/list/page.tsx:17-28`, `apps/web/modules/survey/list/page.tsx:48-58`).
- UI preference state: filters live in React state and local storage (`apps/web/modules/survey/list/components/survey-list.tsx:51-87`).
- Server data state: React Query owns pages, loading, errors, and cursor pagination (`apps/web/modules/survey/list/hooks/use-surveys.ts:25-51`).

```tsx
// apps/web/modules/survey/list/hooks/use-surveys.ts:25-51
const query = useInfiniteQuery({
  queryKey, // Includes workspaceId, limit, and filters, so cache entries match the view.
  initialPageParam: null as string | null,
  enabled, // Lets the component delay fetching until local-storage filters are resolved.
  placeholderData: keepPreviousData, // Keeps old rows visible while filters refetch.
  queryFn: ({ pageParam, signal }) =>
    listSurveys({ workspaceId, limit, cursor: pageParam, includeTotalCount: pageParam === null, filters, signal }),
  getNextPageParam: (lastPage) => lastPage.meta.nextCursor ?? undefined,
});
```

## API Contract Map

- `GET /api/v3/surveys`: auth mode `both`, query parsed by `parseV3SurveysListQuery`, response envelope `{ data, meta }` through `successListResponse` (`apps/web/app/api/v3/surveys/route.ts:20-82`, `apps/web/app/api/v3/lib/response.ts:136-149`).
- `DELETE /api/v3/surveys/[surveyId]`: auth mode `both`, params validated with `z.cuid2()`, permission requires `readWrite`, response envelope `{ data: { id } }` (`apps/web/app/api/v3/surveys/[surveyId]/route.ts:10-72`).
- `POST /api/v2/client/[environmentId]/responses`: public client response endpoint, validates environment and body, checks survey rules, creates response, sends pipeline events (`apps/web/app/api/v2/client/[environmentId]/responses/route.ts:46-80`, `apps/web/app/api/v2/client/[environmentId]/responses/route.ts:204-278`).

## Component Interaction Map

`SurveysPage` fetches server-side project/auth/locale data and renders `SurveysList` (`apps/web/modules/survey/list/page.tsx:23-59`). `SurveysList` controls filters, fetches pages, computes display states, renders `SurveyFilters`, and maps each row to `SurveyCard` (`apps/web/modules/survey/list/components/survey-list.tsx:40-244`). `SurveyCard` delegates status display to `SurveyStatusIndicator`, type display to `SurveyTypeIndicator`, and menu actions to `SurveyDropDownMenu` (`apps/web/modules/survey/list/components/survey-card.tsx:10-13`, `apps/web/modules/survey/list/components/survey-card.tsx:57-115`).

## Full-Stack Feature Trace

1. User opens `/environments/:environmentId/surveys`; the App Router route re-exports the module page (`apps/web/app/(app)/environments/[environmentId]/surveys/page.tsx:1-4`).
2. Server page resolves `params`, gets translation, loads the project, checks environment auth, handles billing redirect, loads locale, and passes props to `SurveysList` (`apps/web/modules/survey/list/page.tsx:23-59`).
3. `SurveysList` initializes filters from local storage, normalizes them, and enables fetching only after initialization (`apps/web/modules/survey/list/components/survey-list.tsx:55-105`).
4. `useSurveys` builds a stable query key and calls `listSurveys` with cursor and filters (`apps/web/modules/survey/list/hooks/use-surveys.ts:19-40`).
5. `listSurveys` serializes filters into `/api/v3/surveys` query params, sends a no-store GET, parses errors through v3 API error handling, and converts ISO dates back to `Date` objects (`apps/web/modules/survey/list/lib/v3-surveys-client.ts:30-121`).
6. The v3 route wrapper authenticates, validates optional schemas, applies rate limits, queues audit logs, and guarantees a request-id header (`apps/web/app/api/v3/lib/api-wrapper.ts:126-154`, `apps/web/app/api/v3/lib/api-wrapper.ts:345-423`).
7. The route authorizes workspace access, resolves `environmentId`, reads page rows and optional total count in parallel, serializes each item, and returns `{ data, meta }` (`apps/web/app/api/v3/surveys/route.ts:36-68`).
8. `getSurveyListPage` chooses relevance or standard pagination and wraps Prisma known errors as `DatabaseError` (`apps/web/modules/survey/list/lib/survey-page.ts:412-440`).
9. `findSurveyRows` scopes by `environmentId`, applies filter criteria, selects list fields, orders stably, and takes `limit + 1` for cursor detection (`apps/web/modules/survey/list/lib/survey-page.ts:204-223`).
10. The UI receives pages, flattens them, renders cards, and shows "Load more" if `hasNextPage` exists (`apps/web/modules/survey/list/components/survey-list.tsx:190-244`).

## Diff Reading Exercise

Hypothetical change: "Add an `archived` survey status to the dashboard."

Read the diff in this order:

1. Schema/type impact: survey status enum comes from shared survey types and Prisma status already lists `draft`, `inProgress`, `paused`, `completed` (`packages/database/schema.prisma:225-230`, `apps/web/modules/survey/list/types/survey-overview.ts:1-39`).
2. API filter impact: filter query params must accept and serialize the new status (`apps/web/modules/survey/list/lib/v3-surveys-client.ts:70-76`).
3. UI display impact: status label and icon must handle the new case (`apps/web/modules/survey/list/components/survey-card.tsx:32-45`, `apps/web/modules/ui/components/survey-status-indicator/index.tsx:13-98`).
4. Test impact: route and pagination tests should cover filter propagation and status serialization (`apps/web/app/api/v3/surveys/route.test.ts:283-299`, `apps/web/modules/survey/list/lib/survey-page.test.ts:78-155`).

Strong review comment example: "This adds `archived` to the filter UI but not to `SurveyStatusIndicator`, so archived surveys will render without an icon in both tooltip and non-tooltip branches. Please update `apps/web/modules/ui/components/survey-status-indicator/index.tsx:13-98` and add a component or E2E assertion."

## Non-Obvious Patterns

- Route files often re-export module pages to keep URL structure separate from feature implementation (`apps/web/app/(app)/environments/[environmentId]/surveys/page.tsx:1-4`).
- v3 uses a common wrapper because every API route needs the same request id, auth, validation, rate limiting, audit, and error response behavior (`apps/web/app/api/v3/lib/api-wrapper.ts:345-423`).
- Cursor pagination uses a versioned encoded JSON cursor rather than raw offsets to preserve stable ordering across changes (`apps/web/modules/survey/list/lib/survey-page.ts:17-43`, `apps/web/modules/survey/list/lib/survey-page.ts:70-94`).
- The list endpoint hides internal `environmentId` by serializing items with `workspaceId`, matching the product vocabulary in v3 (`apps/web/app/api/v3/surveys/serializers.ts:11-18`).

## Mid-Level Socratic Checkpoint

1. Why does `SurveysList` delay fetching until filters are initialized?
2. What would break if `surveySelect` removed `creator.name`?
3. Why does cursor pagination query `limit + 1` rows?
4. Why does the v3 wrapper authenticate before parsing handler-specific logic?
5. Where is workspace authorization enforced after authentication?
6. Why is `includeTotalCount` true only on the initial page?
7. How does optimistic deletion protect the UI from a failed delete?
8. Which test file should change if cursor behavior changes?

## How To Self-Grade

Strong answers separate concerns: URL route, server page, client state, data-fetching hook, API route, wrapper/auth, Prisma query, and UI rendering. They mention exact files, explain why each layer exists, and name a risk: stale filters, leaking unauthorized workspace existence, unstable pagination, or cache inconsististency after mutation.

