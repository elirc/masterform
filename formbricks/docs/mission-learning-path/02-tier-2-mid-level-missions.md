# Tier 2 Mid-Level Missions

### Mission 9: State Has a Home and a Reason
**Tier:** Mid-Level  
**Time Estimate:** 45 minutes  
**Goal:** Separate local UI state, persistent UI preference, and server data cache.  
**The Concept:** The survey dashboard has three desks: filters on your desk, saved preferences in a drawer, and server data in the records system.  
**Design Intent Before You Read the Code:** State should live where its lifecycle belongs; wrong homes cause flicker, stale data, or accidental global coupling.  
**Find It In The Code:** Open `apps/web/modules/survey/list/components/survey-list.tsx:51-105`, `apps/web/modules/survey/list/hooks/use-surveys.ts:19-51`.

```tsx
// apps/web/modules/survey/list/components/survey-list.tsx:51-87
const [surveyFilters, setSurveyFilters] = useState<TSurveyOverviewFilters>(initialFilters); // Component state.
const [isFilterInitialized, setIsFilterInitialized] = useState(false); // Guards first fetch.
localStorage.setItem(FORMBRICKS_SURVEYS_FILTERS_KEY_LS, JSON.stringify(normalizedFilters)); // Persistent preference.
```

**The Aha Moment:** State is not just data; it is data plus lifecycle.  
**Socratic Checkpoint:** Which state is persisted? Which state gates fetching? Which state comes from server? Which state is derived? Which bug appears if fetching starts too early?  
**How to self-grade:** Strong answers name local state, localStorage, React Query, `useMemo`, and premature fetch risk.  
**Connects To:** Mission 10 and Mission 14.

### Mission 10: The Custom Hook Ecosystem
**Tier:** Mid-Level  
**Time Estimate:** 45 minutes  
**Goal:** Explain how hooks hide complexity without hiding ownership.  
**The Concept:** Hooks are reusable survey operations routines: "load list," "delete survey," "close when clicking outside."  
**Design Intent Before You Read the Code:** A hook should own one reusable behavior and expose a small API.  
**Find It In The Code:** Open `apps/web/modules/survey/list/hooks/use-surveys.ts:8-51`, `apps/web/modules/survey/list/hooks/use-delete-survey.ts:7-33`, `apps/web/lib/utils/hooks/useClickOutside.ts:4-35`.

```ts
// apps/web/modules/survey/list/hooks/use-delete-survey.ts:10-33
return useMutation({
  mutationFn: async ({ surveyId }) => deleteSurvey(surveyId), // Server mutation.
  onMutate: async ({ surveyId }) => { /* optimistic cache update */ },
  onError: (_error, _variables, context) => { /* rollback */ },
  onSettled: async () => { await queryClient.invalidateQueries({ queryKey: surveyKeys.lists() }); },
});
```

**The Aha Moment:** A custom hook is a boundary around timing, cleanup, or cache rules.  
**Socratic Checkpoint:** Which hook fetches pages? Which hook mutates and rolls back? Which hook cleans listeners? What does each return? What should not go inside these hooks?  
**How to self-grade:** Strong answers mention `useInfiniteQuery`, `useMutation`, cleanup in `useEffect`, return shape, and single responsibility.  
**Connects To:** Mission 11 and Mission 23.

### Mission 11: Side Effects Are Promises to the System
**Tier:** Mid-Level  
**Time Estimate:** 40 minutes  
**Goal:** Identify effects and their cleanup obligations.  
**The Concept:** Every side effect is like opening a survey booth: if you set it up, you must know when to close it.  
**Design Intent Before You Read the Code:** Effects should synchronize with external systems and clean up event listeners.  
**Find It In The Code:** Open `apps/web/lib/utils/hooks/useClickOutside.ts:8-35`, `apps/web/modules/survey/list/components/survey-list.tsx:55-87`.

```ts
// apps/web/lib/utils/hooks/useClickOutside.ts:26-35
document.addEventListener("mousedown", validateEventStart); // External system subscription.
document.addEventListener("click", listener);
return () => {
  document.removeEventListener("mousedown", validateEventStart); // Cleanup mirrors setup.
  document.removeEventListener("click", listener);
};
```

**The Aha Moment:** Effects are correct only when their setup and cleanup tell the same story.  
**Socratic Checkpoint:** What external systems are touched? What cleanup exists? Why track `startedInside`? What dependency could make the effect rerun? What warning would React give for missing cleanup?  
**How to self-grade:** Strong answers mention document listeners, localStorage, dependency arrays, cleanup symmetry, and stale handler risks.  
**Connects To:** Mission 19.

### Mission 12: The Full API Contract
**Tier:** Mid-Level  
**Time Estimate:** 60 minutes  
**Goal:** Map request parsing, auth, response envelope, and errors for v3 survey list.  
**The Concept:** The API contract is the survey office counter: requests need ID, permission, form fields, and a standard receipt.  
**Design Intent Before You Read the Code:** Contract consistency lets UI clients, API clients, tests, and logs agree.  
**Find It In The Code:** Open `apps/web/modules/survey/list/lib/v3-surveys-client.ts:38-121`, `apps/web/app/api/v3/surveys/route.ts:20-82`, `apps/web/app/api/v3/lib/response.ts:62-149`.

```ts
// apps/web/app/api/v3/lib/response.ts:136-149
export function successListResponse(data, meta, options) {
  const headers = { "Content-Type": "application/json", "Cache-Control": options?.cache ?? "private, no-store" };
  return Response.json({ data, meta }, { status: 200, headers }); // Standard list envelope.
}
```

**The Aha Moment:** A reliable API is a shape, not just a URL.  
**Socratic Checkpoint:** What query params does the client send? What error helper handles invalid params? What success shape returns? Where is cache controlled? What carries request id?  
**How to self-grade:** Strong answers cite search params, `problemBadRequest`, `{ data, meta }`, no-store cache, and `X-Request-Id`.  
**Connects To:** Mission 13 and Mission 14.

### Mission 13: The Middleware Chain
**Tier:** Mid-Level  
**Time Estimate:** 55 minutes  
**Goal:** Explain request classification and wrapper responsibilities.  
**The Concept:** Requests pass through a check-in desk before reaching survey staff: identify route type, authenticate, rate-limit, validate, audit.  
**Design Intent Before You Read the Code:** Cross-cutting concerns should be centralized enough to be consistent but visible enough to debug.  
**Find It In The Code:** Open `apps/web/app/middleware/endpoint-validator.ts:7-35`, `apps/web/app/api/v3/lib/api-wrapper.ts:345-423`.

```ts
// apps/web/app/api/v3/lib/api-wrapper.ts:368-413
const authResult = await authenticateV3RequestOrRespond(req, auth, requestId, instance); // Who are you?
const parsedInputResult = await parseV3Input(req, props, schemas, requestId, instance); // Is your form valid?
const rateLimitResponse = await applyV3RateLimitOrRespond({ authentication, enabled: rateLimit, config, requestId, log }); // Too much?
const response = await handler({ req, authentication, parsedInput, requestId, instance }); // Actual route work.
return ensureRequestIdHeader(response, requestId); // Traceability.
```

**The Aha Moment:** Middleware is the system's habit of asking the same questions every time.  
**Socratic Checkpoint:** How are v3 survey routes classified? Which auth modes exist? Where is rate limit applied? Where is audit queued? Where is request id guaranteed?  
**How to self-grade:** Strong answers mention `AuthenticationMethod.Both`, `TV3AuthMode`, `applyV3RateLimitOrRespond`, `queueV3AuditLog`, and `ensureRequestIdHeader`.  
**Connects To:** Mission 22.

### Mission 14: End-to-End Feature Trace
**Tier:** Mid-Level  
**Time Estimate:** 75 minutes  
**Goal:** Trace survey list from URL to rendered cards.  
**The Concept:** A survey list is a production report: choose workspace, filter forms, ask records, return rows, display them.  
**Design Intent Before You Read the Code:** Each layer should transform just enough and pass a clearer contract to the next.  
**Find It In The Code:** Open `apps/web/app/(app)/environments/[environmentId]/surveys/page.tsx:1-4`, `apps/web/modules/survey/list/page.tsx:23-59`, `apps/web/modules/survey/list/components/survey-list.tsx:89-244`, `apps/web/app/api/v3/surveys/route.ts:20-82`, `apps/web/modules/survey/list/lib/survey-page.ts:412-440`.

```tsx
// apps/web/modules/survey/list/components/survey-list.tsx:190-224
{surveys.map((survey) => (
  <SurveyCard key={survey.id} survey={survey} environmentId={environment.id} deleteSurvey={handleDeleteSurvey} />
))}
{hasNextPage && <Button onClick={() => fetchNextPage()} loading={isFetchingNextPage}>Load more</Button>}
```

**The Aha Moment:** Full-stack tracing is following one piece of data until it becomes pixels.  
**Socratic Checkpoint:** Where does `environmentId` start? Where does `workspaceId` go? Where is auth checked? Where does Prisma run? Where is `responseCount` attached?  
**How to self-grade:** Strong answers cite each layer in order and include `getResponseCountsBySurveyIds`.  
**Connects To:** Mission 15 and Mission 21.

### Mission 15: Read the Diff Like an Engineer
**Tier:** Mid-Level  
**Time Estimate:** 50 minutes  
**Goal:** Practice reviewing a realistic survey-status change.  
**The Concept:** A diff is a proposed change to the survey operation manual; you must check every affected desk.  
**Design Intent Before You Read the Code:** Status changes touch schema, filters, icons, labels, API tests, and E2E behavior.  
**Find It In The Code:** Open `packages/database/schema.prisma:225-230`, `apps/web/modules/survey/list/components/survey-card.tsx:32-45`, `apps/web/modules/ui/components/survey-status-indicator/index.tsx:13-98`, `apps/web/app/api/v3/surveys/route.test.ts:283-354`.

```tsx
// apps/web/modules/survey/list/components/survey-card.tsx:32-45
switch (survey.status) {
  case "inProgress": return t("common.in_progress"); // New statuses need labels.
  case "completed": return t("common.completed");
  case "draft": return t("common.draft");
  case "paused": return t("common.paused");
}
```

**The Aha Moment:** Review the surface area of a concept, not just the edited lines.  
**Socratic Checkpoint:** What files must change for a new status? What test catches serialization? What UI branch can drift? What translation rule applies? What database migration might be needed?  
**How to self-grade:** Strong answers list schema/types/UI/i18n/tests/API and mention duplicated indicator branches.  
**Connects To:** Mission 19.

### Mission 16: Composition Over Inheritance
**Tier:** Mid-Level  
**Time Estimate:** 35 minutes  
**Goal:** Identify how the UI composes small pieces.  
**The Concept:** Survey rows are assembled from specialized instruments: status icon, type indicator, menu, link, date formatter.  
**Design Intent Before You Read the Code:** Composition keeps components focused and replaceable.  
**Find It In The Code:** Open `apps/web/modules/survey/list/components/survey-card.tsx:10-13`, `apps/web/modules/survey/list/components/survey-card.tsx:57-115`, `apps/web/modules/ui/components/survey-status-indicator/index.tsx:13-98`.

```tsx
// apps/web/modules/survey/list/components/survey-card.tsx:74-112
<SurveyStatusIndicator status={survey.status} /> // Status display.
<SurveyTypeIndicator type={survey.type} />       // Type display.
<SurveyDropDownMenu survey={survey} ... />       // Actions.
```

**The Aha Moment:** Composition turns one complicated card into several understandable contracts.  
**Socratic Checkpoint:** Which child owns status? Which child owns actions? Which child owns navigation? What remains in the parent? What would inheritance make harder here?  
**How to self-grade:** Strong answers name component responsibilities and parent orchestration.  
**Connects To:** Mission 18.

### Mission 17: TypeScript's Hidden Work
**Tier:** Mid-Level  
**Time Estimate:** 45 minutes  
**Goal:** See how TypeScript prevents invisible field drift.  
**The Concept:** TypeScript is the survey office checklist that notices a missing column before the report goes out.  
**Design Intent Before You Read the Code:** Inferred Prisma and Zod types keep selected data, parsed data, and UI expectations aligned.  
**Find It In The Code:** Open `apps/web/modules/survey/list/lib/survey-record.ts:5-22`, `apps/web/app/api/v3/lib/api-wrapper.ts:34-58`.

```ts
// apps/web/app/api/v3/lib/api-wrapper.ts:34-58
export type TV3ParsedInput<S extends TV3Schemas | undefined> = S extends object
  ? { [K in keyof S as NonNullable<S[K]> extends TV3Schema ? K : never]: z.infer<NonNullable<S[K]>> }
  : Record<string, never>; // Handler input follows provided schemas.
```

**The Aha Moment:** Advanced TypeScript is most valuable when it removes manual synchronization.  
**Socratic Checkpoint:** What does `satisfies` protect? What does `z.infer` protect? What does generic parsed input protect? Where is `any` still used? What test backs lower-typed code?  
**How to self-grade:** Strong answers cite Prisma select, Zod schemas, wrapper generics, NextAuth route `any`, and route tests.  
**Connects To:** Mission 23.

