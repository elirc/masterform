# Tier 3 Senior Missions

### Mission 18: Reverse-Engineer the Architecture Decisions
**Tier:** Senior  
**Time Estimate:** 60 minutes  
**Goal:** Infer why the code is shaped this way.  
**The Concept:** Architecture decisions are the survey platform's operating policy: routes are signs, modules are teams, wrappers are standardized intake.  
**Design Intent Before You Read the Code:** Look for repeated structure, not isolated files.  
**Find It In The Code:** Open `apps/web/app/(app)/environments/[environmentId]/surveys/page.tsx:1-4`, `apps/web/app/api/v3/lib/api-wrapper.ts:345-423`, `apps/web/modules/survey/list/lib/survey-page.ts:17-94`.

```ts
// apps/web/modules/survey/list/lib/survey-page.ts:17-43
const SURVEY_LIST_CURSOR_VERSION = 1 as const; // Cursor format can evolve.
const ZSurveyListPageCursor = z.union([ZDateCursor, ZNameCursor, ZRelevanceCursor]); // Different sorts need different cursor shapes.
```

**The Aha Moment:** Good architecture is repeated evidence of a team solving the same problem once.  
**Socratic Checkpoint:** Why delegate route files? Why wrap v3 routes? Why version cursors? Why use module folders? Why expose `workspaceId` over `environmentId`?  
**How to self-grade:** Strong answers infer maintainability, consistency, evolvability, feature cohesion, and product vocabulary.  
**Connects To:** Mission 24.

### Mission 19: Find the Bugs Before They Happen
**Tier:** Senior  
**Time Estimate:** 55 minutes  
**Goal:** Predict failure modes from code shape.  
**The Concept:** Senior debugging is reading the survey room before anyone trips: where are the wet floors?  
**Design Intent Before You Read the Code:** Look for duplicated branches, broad abstractions, lifecycle edges, and stale cache risks.  
**Find It In The Code:** Open `apps/web/modules/ui/components/survey-status-indicator/index.tsx:13-98`, `apps/web/app/api/v3/lib/api-wrapper.ts:345-423`, `apps/web/modules/survey/list/hooks/use-delete-survey.ts:10-33`.

```tsx
// apps/web/modules/ui/components/survey-status-indicator/index.tsx:15-98
if (tooltip) {
  // One rendering branch.
} else {
  // A second rendering branch with similar status logic; future statuses can drift.
}
```

**The Aha Moment:** Bugs often hide where a concept is represented twice.  
**Socratic Checkpoint:** Where is duplicate status rendering? Where is wrapper blast radius high? Where can optimistic UI lie? Where can auth leak existence? Where can localStorage corrupt state?  
**How to self-grade:** Strong answers name duplicated rendering, broad wrapper, mutation rollback, generic 403, and parse/remove invalid filters.  
**Connects To:** Mission 20 and Mission 23.

### Mission 20: The Bug Injection Challenge
**Tier:** Senior  
**Time Estimate:** 60 minutes  
**Goal:** Design tests for user-visible symptoms without modifying production code.  
**The Concept:** Injecting bugs mentally is like stress-testing a survey before launch.  
**Design Intent Before You Read the Code:** Start with symptom, then find the narrowest test that would fail.  
**Find It In The Code:** Open `apps/web/modules/survey/list/lib/survey-page.test.ts:49-334`, `apps/web/app/api/v3/surveys/route.test.ts:91-372`.

```ts
// apps/web/modules/survey/list/lib/survey-page.test.ts:85-155
test("uses a stable updatedAt order with a next cursor", async () => {
  // Test should prove order, cursor, and next-page behavior together.
});
```

**The Aha Moment:** A good bug test describes the broken user promise, not implementation trivia.  
**Socratic Checkpoint:** What symptom proves cursor instability? What route test proves unauthorized access? What UI test proves bad status display? What test proves API envelope stability? What test proves rollback?  
**How to self-grade:** Strong answers pair each symptom with one narrow file and assertion.  
**Connects To:** Mission 23.

### Mission 21: Performance X-Ray
**Tier:** Senior  
**Time Estimate:** 50 minutes  
**Goal:** Identify good and risky data-access patterns.  
**The Concept:** Performance is how quickly the survey office can pull the right records without rummaging through every cabinet.  
**Design Intent Before You Read the Code:** Prefer scoped queries, minimal selects, grouped counts, cursor pagination, and parallel reads.  
**Find It In The Code:** Open `apps/web/app/api/v3/surveys/route.ts:49-58`, `apps/web/modules/survey/list/lib/survey-page.ts:204-267`, `apps/web/modules/survey/list/lib/survey-record.ts:23-55`, `apps/web/modules/survey/list/lib/survey.ts:29-67`.

```ts
// apps/web/app/api/v3/surveys/route.ts:49-58
const surveyPagePromise = getSurveyListPage(...); // Main list read.
const totalCountPromise = includeTotalCount ? getSurveyCount(...) : Promise.resolve(null); // Optional count.
const [surveyPage, totalCount] = await Promise.all([surveyPagePromise, totalCountPromise]); // Parallel.
```

**The Aha Moment:** Performance is often visible in query shape before you run a profiler.  
**Socratic Checkpoint:** Where is count parallelized? Where is cursor pagination used? Where are counts grouped? Where does offset pagination remain? Which index supports survey list order?  
**How to self-grade:** Strong answers cite `Promise.all`, `limit + 1`, `groupBy`, older `skip`, and `@@index([environmentId, updatedAt])`.  
**Connects To:** Mission 22.

### Mission 22: The Security Audit
**Tier:** Senior  
**Time Estimate:** 65 minutes  
**Goal:** Audit auth, authorization, public write validation, and error leakage.  
**The Concept:** Security is the survey booth checking badges, validating forms, and not announcing private records to strangers.  
**Design Intent Before You Read the Code:** Authentication proves identity; authorization proves access; validation protects state; generic errors reduce leakage.  
**Find It In The Code:** Open `apps/web/modules/auth/lib/authOptions.ts:190-260`, `apps/web/app/api/v3/lib/auth.ts:79-122`, `apps/web/app/api/v2/client/[environmentId]/responses/route.ts:46-138`, `apps/web/app/api/v3/lib/response.ts:85-112`.

```ts
// apps/web/app/api/v3/lib/auth.ts:99-110
const context = await resolveV3WorkspaceContext(workspaceId); // Resolve internal ids.
const permission = keyAuth.environmentPermissions.find((p) => p.environmentId === context.environmentId);
if (!permission || !apiKeyPermissionAllows(permission.permission, minPermission)) {
  return problemForbidden(requestId, "You are not authorized to access this resource", instance);
}
```

**The Aha Moment:** The safest route is the one that validates identity, scope, and shape before doing useful work.  
**Socratic Checkpoint:** Where is password CPU DoS reduced? Where is API-key scope checked? Where does public response shape validate? Where are not-found details risky? Why use generic 403?  
**How to self-grade:** Strong answers cite password length, environment permission, Zod/body validation, `problemNotFound` warning, and resource leakage.  
**Connects To:** Mission 23.

### Mission 23: Write the Test That Doesn't Exist
**Tier:** Senior  
**Time Estimate:** 70 minutes  
**Goal:** Design a missing high-value test without violating repo test strategy.  
**The Concept:** A test is a rehearsal of the user's promise before launch day.  
**Design Intent Before You Read the Code:** Unit-test logic and routes; use Playwright for React component workflows.  
**Find It In The Code:** Open `playwright.config.ts:12-36`, `vitest.workspace.ts:1-1`, `apps/web/modules/survey/list/hooks/use-delete-survey.ts:10-33`.

```ts
// apps/web/modules/survey/list/hooks/use-delete-survey.ts:12-31
onMutate: async ({ surveyId }) => { /* remove from cache immediately */ },
onError: (_error, _variables, context) => { /* restore previous cache */ },
onSettled: async () => { /* invalidate survey list queries */ },
```

**The Aha Moment:** Test at the lowest layer that proves the promise, unless the promise is visual workflow.  
**Socratic Checkpoint:** Would you use Vitest or Playwright for localStorage filters? Which hook test proves rollback? Which route test proves 403? What fixture data is needed? What assertion proves the user promise?  
**How to self-grade:** Strong answers choose Playwright for component workflow, Vitest for hooks/routes, and define setup/action/assertion clearly.  
**Connects To:** Mission 24.

### Mission 24: The Git History Tells a Story
**Tier:** Senior  
**Time Estimate:** 45 minutes  
**Goal:** Infer system evolution from realistic commit messages and current code.  
**The Concept:** Git history is the survey product's diary; every fix reveals a pressure point.  
**Design Intent Before You Read the Code:** Use history to understand why constraints exist before changing them.  
**Find It In The Code:** Open `apps/web/modules/survey/list/lib/survey-page.ts:17-94`, `apps/web/app/api/v3/lib/response.ts:1-4`, `apps/web/modules/auth/lib/authOptions.ts:203-235`.

```ts
// apps/web/app/api/v3/lib/response.ts:1-4
// V3 API response helpers — RFC 9457 Problem Details and list envelope.
// This comment implies a deliberate contract standardization.
```

**The Aha Moment:** Before removing complexity, ask what incident or product need probably created it.  
**Socratic Checkpoint:** What story does cursor versioning tell? What story does problem-json tell? What story does control-hash auth tell? What story does route delegation tell? What story does React Query tell?  
**How to self-grade:** Strong answers connect code shape to past scale, API consistency, security hardening, maintainability, and client interactivity.  
**Connects To:** Mission 25.

