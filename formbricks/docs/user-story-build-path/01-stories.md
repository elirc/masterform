# User Stories

## Story 1: Make Read-Only Draft Surveys More Obvious
**Difficulty:** Easy  
**Estimated Time:** 1 hour  
**Skills You'll Practice:** React props, conditional rendering, i18n  
**The Story:** As a read-only workspace member, I want draft surveys to visibly communicate that I cannot open them so that I understand why the card is not clickable.  
**Acceptance Criteria:**
- [ ] Draft survey cards for read-only users show a translated visual label.
- [ ] Non-read-only users see no new read-only label.
- [ ] Published/read-only survey cards still link to summary pages.
**Files You'll Likely Touch:** `apps/web/modules/survey/list/components/survey-card.tsx` because it already branches on `isDraftAndReadOnly`; `apps/web/locales/en-US.json` because user-facing text must be translated.  
**High-Level Implementation Plan:**
1. In `SurveyCard`, extend the `isDraftAndReadOnly` branch around `apps/web/modules/survey/list/components/survey-card.tsx:55-115`.
2. Add a small translated label near the status or name.
3. Add the English translation key in `apps/web/locales/en-US.json`.
**Tips:**
- `SurveyCard` already computes `isDraftAndReadOnly`.
- Use `t()` from `useTranslation()` as existing labels do.
- Keep the card grid stable; avoid changing column counts.
**What Could Go Wrong:**
- Label text could overflow the grid.
- You could accidentally remove the read-only no-link behavior.
**Stretch Goal:** Add a tooltip explaining that the user needs write access.  
**Connects To:** Story 2, because both modify survey list row display.

## Story 2: Show "No Creator" With a Better Label
**Difficulty:** Easy  
**Estimated Time:** 1 hour  
**Skills You'll Practice:** UI polish, i18n, null handling  
**The Story:** As a workspace member, I want surveys without a creator to show a meaningful label so that the list feels intentional rather than broken.  
**Acceptance Criteria:**
- [ ] Surveys with `creator: null` show a translated fallback label.
- [ ] Surveys with creators still show the creator name.
- [ ] The fallback does not break truncation or row layout.
**Files You'll Likely Touch:** `apps/web/modules/survey/list/components/survey-card.tsx` because it currently renders `survey.creator ? survey.creator.name : "-"`; `apps/web/locales/en-US.json` for the label.  
**High-Level Implementation Plan:**
1. Replace the `"-"` fallback near `apps/web/modules/survey/list/components/survey-card.tsx:88-90`.
2. Add a translation key such as `environments.surveys.unknown_creator`.
3. Verify cards with and without creators.
**Tips:**
- Follow existing `t("common...")` usage in `SurveyCard`.
- Do not infer a creator from user/session; the API contract says creator can be null.
- Keep text inside the existing overflow classes.
**What Could Go Wrong:**
- Missing translation key will render a raw key.
- A too-long fallback can truncate awkwardly.
**Stretch Goal:** Add a muted style to distinguish system/imported surveys.  
**Connects To:** Story 3, because both strengthen status/metadata clarity.

## Story 3: Add Tooltip Text For Draft Status
**Difficulty:** Easy  
**Estimated Time:** 1.5 hours  
**Skills You'll Practice:** Component branching, tooltip UI, i18n  
**The Story:** As a survey builder, I want draft status icons to explain themselves so that I can scan survey readiness quickly.  
**Acceptance Criteria:**
- [ ] Tooltip mode for `draft` shows translated explanatory text.
- [ ] Non-tooltip mode still shows the pencil icon.
- [ ] Existing statuses keep their current tooltip behavior.
**Files You'll Likely Touch:** `apps/web/modules/ui/components/survey-status-indicator/index.tsx` because tooltip/non-tooltip status rendering lives there; `apps/web/locales/en-US.json` for the text.  
**High-Level Implementation Plan:**
1. Inspect tooltip branch around `apps/web/modules/ui/components/survey-status-indicator/index.tsx:15-71`.
2. Add a draft tooltip branch using the existing `PencilIcon`.
3. Add a translation key for draft explanation.
**Tips:**
- The non-tooltip branch already handles draft at `apps/web/modules/ui/components/survey-status-indicator/index.tsx:91-95`.
- Avoid duplicating more markup than necessary.
- Keep icon sizes consistent with other statuses.
**What Could Go Wrong:**
- Draft icon could differ between tooltip and non-tooltip modes.
- Tooltip content could miss `t()` and violate i18n rules.
**Stretch Goal:** Refactor the component to use a status config map.  
**Connects To:** Story 4, because status display feeds list filtering UX.

## Story 4: Add A Visible Active Filter Count
**Difficulty:** Medium  
**Estimated Time:** 3 hours  
**Skills You'll Practice:** Derived state, component props, filter UX  
**The Story:** As a workspace member, I want to see how many survey filters are active so that I know when the list is narrowed.  
**Acceptance Criteria:**
- [ ] The survey list header or filter area shows a translated active filter count when filters are applied.
- [ ] The count is hidden or zero when no filters are active.
- [ ] Name, status, type, and non-default sort choices are counted consistently.
**Files You'll Likely Touch:** `apps/web/modules/survey/list/components/survey-list.tsx` because it computes `hasAppliedFilters`; `apps/web/modules/survey/list/lib/utils.ts` because filter normalization and active checks live there; `apps/web/modules/survey/list/components/survey-filters.tsx` because it owns filter UI; `apps/web/locales/en-US.json`.  
**High-Level Implementation Plan:**
1. Add or extend a utility near `hasActiveSurveyFilters` in `apps/web/modules/survey/list/lib/utils.ts`.
2. Pass the count from `SurveysList` to `SurveyFilters`.
3. Render a compact translated count near existing filter controls.
4. Add unit tests for the utility if one exists beside `utils.test.ts`.
**Tips:**
- `SurveysList` already computes `normalizedFilters`.
- Use `initialFilters` from `apps/web/modules/survey/list/lib/constants.ts` as baseline.
- Keep `currentProjectChannel` normalization in mind.
**What Could Go Wrong:**
- Counting raw filters before normalization can overstate active filters.
- Sort defaults can differ by channel if normalization changes allowed types.
**Stretch Goal:** Add a one-click "clear all" action next to the count.  
**Connects To:** Story 5, because it prepares you to improve empty states.

## Story 5: Improve Filtered Empty State
**Difficulty:** Medium  
**Estimated Time:** 3 hours  
**Skills You'll Practice:** Conditional rendering, UX states, i18n  
**The Story:** As a workspace member, I want a clearer empty state when filters return no surveys so that I know I should adjust filters instead of creating a survey.  
**Acceptance Criteria:**
- [ ] When filters are active and no surveys match, the empty state says the list is filtered.
- [ ] The existing template empty state still appears only when there are zero surveys and no filters.
- [ ] Read-only empty state remains unchanged for zero surveys without filters.
**Files You'll Likely Touch:** `apps/web/modules/survey/list/components/survey-list.tsx` because it computes empty states; `apps/web/modules/survey/list/components/survey-filters.tsx` if adding a clear action; `apps/web/locales/en-US.json`.  
**High-Level Implementation Plan:**
1. Inspect `showTemplateEmptyState`, `showReadOnlyEmptyState`, and default `surveyContent`.
2. Add a filtered-empty branch before the generic no-surveys message.
3. Optionally call `setSurveyFilters(initialFilters)` for a clear button.
**Tips:**
- `hasAppliedFilters` already exists in `SurveysList`.
- Do not break `TemplateContainerWithPreview`; it is the no-survey creation path.
- Keep all visible text translated.
**What Could Go Wrong:**
- Filtered empty state could appear during initial loading if `isFilterInitialized` is ignored.
- Clear action could write invalid state to localStorage if not normalized.
**Stretch Goal:** Include active filter count from Story 4.  
**Connects To:** Story 6, because clearer filtered state pairs with API filter behavior.

## Story 6: Add A Client-Side "Refresh Surveys" Control
**Difficulty:** Medium  
**Estimated Time:** 4 hours  
**Skills You'll Practice:** React Query, loading states, UI actions  
**The Story:** As a workspace member, I want to refresh the survey list manually so that I can confirm recent survey changes without reloading the page.  
**Acceptance Criteria:**
- [ ] A refresh button appears near survey filters.
- [ ] Clicking it calls the existing React Query refetch path.
- [ ] The button shows a loading state while refetching.
- [ ] Errors still show the existing retry UI.
**Files You'll Likely Touch:** `apps/web/modules/survey/list/components/survey-list.tsx` because it receives `refetch` and loading flags; `apps/web/modules/ui/components/button/index.tsx` if a variant is missing; `apps/web/locales/en-US.json`.  
**High-Level Implementation Plan:**
1. Use `refetch` from `useSurveys` around `apps/web/modules/survey/list/components/survey-list.tsx:89-105`.
2. Add a button near `SurveyFilters`.
3. Use query fetching state to show loading without blocking load-more.
4. Keep existing error branch intact.
**Tips:**
- `Button` already supports `loading`.
- Use an icon from `lucide-react` if available.
- Do not create a new fetch path; reuse React Query.
**What Could Go Wrong:**
- You might use `isLoading`, which only reflects first load, instead of refetch state.
- Refresh could run before filters are initialized.
**Stretch Goal:** Add a short toast on successful refresh.  
**Connects To:** Story 7, because both touch client/server state synchronization.

## Story 7: Add API Support For Excluding Completed Surveys
**Difficulty:** Medium  
**Estimated Time:** 5 hours  
**Skills You'll Practice:** API query params, Zod parsing, React Query keys, tests  
**The Story:** As a workspace member, I want an option to hide completed surveys so that I can focus on active work.  
**Acceptance Criteria:**
- [ ] UI can toggle "hide completed" in the survey filters.
- [ ] The API request carries the filter.
- [ ] The v3 survey list route applies the filter through existing filter criteria.
- [ ] Tests cover query param serialization and route behavior.
**Files You'll Likely Touch:** `apps/web/modules/survey/list/types/survey-overview.ts` for filter shape; `apps/web/modules/survey/list/lib/v3-surveys-client.ts` for query params; `apps/web/app/api/v3/surveys/parse-v3-surveys-list-query.ts` for backend parsing; `apps/web/modules/survey/list/components/survey-filters.tsx` for UI; `apps/web/app/api/v3/surveys/route.test.ts`.  
**High-Level Implementation Plan:**
1. Add the filter to `ZSurveyOverviewFilters`.
2. Update normalization and localStorage parsing utilities.
3. Serialize the query param in `buildSurveyListSearchParams`.
4. Parse it in the v3 query parser and translate it into filter criteria.
5. Add tests for serialization and route calls into `getSurveyListPage`.
**Tips:**
- Existing status filters may already express "not completed"; decide whether this is a shortcut or a real new filter.
- Keep query key stable by including the new filter.
- Use backend parsing tests before route tests.
**What Could Go Wrong:**
- LocalStorage with old filter shapes may fail to parse.
- API and UI may disagree on whether completed is hidden by status or separate flag.
**Stretch Goal:** Remember the toggle per workspace instead of globally.  
**Connects To:** Story 8, because it prepares you for schema-backed feature work.

## Story 8: Add A Survey Archive Flag
**Difficulty:** Hard  
**Estimated Time:** 10 hours  
**Skills You'll Practice:** Prisma migration, API filtering, UI status, tests  
**The Story:** As a workspace manager, I want to archive surveys so that old surveys can be hidden without deleting their responses.  
**Acceptance Criteria:**
- [ ] Surveys have a persisted archive flag.
- [ ] Archived surveys are hidden from the default survey list.
- [ ] A filter can show archived surveys.
- [ ] Archiving requires read-write project permission.
- [ ] Existing responses remain intact.
**Files You'll Likely Touch:** `packages/database/schema.prisma` for a new field; `apps/web/modules/survey/list/lib/survey-page.ts` for default filtering; `apps/web/app/api/v3/surveys/[surveyId]/route.ts` or a new route/action for archive mutation; `apps/web/modules/survey/list/components/survey-dropdown-menu.tsx` for menu action; `apps/web/modules/survey/list/types/survey-overview.ts`; tests in route and survey-page test files.  
**High-Level Implementation Plan:**
1. Add a Boolean field to `Survey` in Prisma and create a migration.
2. Update list query filtering to exclude archived by default.
3. Add API/action for archive/unarchive with `readWrite` permission.
4. Add UI menu option and confirmation copy.
5. Add tests for default exclusion, archived filter, and authorization.
**Tips:**
- Survey deletion already requires `readWrite` in v3 delete route.
- Preserve tenant scoping by `environmentId`.
- Do not delete responses; archive is not deletion.
**What Could Go Wrong:**
- Archived surveys could disappear from direct summary pages unintentionally.
- Counts could change if archive filtering is applied inconsistently.
**Stretch Goal:** Add archived count to filters.  
**Connects To:** Story 9, because both require full-stack persistence and analytics awareness.

## Story 9: Add Internal Notes To Survey Responses
**Difficulty:** Hard  
**Estimated Time:** 12 hours  
**Skills You'll Practice:** Data modeling, authenticated mutation, analysis UI, authorization  
**The Story:** As a workspace analyst, I want to add internal notes to individual survey responses so that my team can remember follow-up context.  
**Acceptance Criteria:**
- [ ] A response can have one or more internal notes.
- [ ] Notes are visible only to authenticated workspace users with access.
- [ ] Public response submission cannot set notes.
- [ ] Notes show author and timestamp using shared date formatting.
- [ ] Tests cover unauthorized access and successful note creation.
**Files You'll Likely Touch:** `packages/database/schema.prisma` for a response note model; response analysis components under `apps/web/app/(app)/environments/[environmentId]/surveys/[surveyId]/(analysis)/responses`; `apps/web/app/api/v3` or server actions for note mutations; `apps/web/lib/response/service.ts`; locale file; tests.  
**High-Level Implementation Plan:**
1. Model `ResponseNote` related to `Response` and `User`.
2. Add a service function scoped through survey/environment access.
3. Add an authenticated route or action requiring workspace access.
4. Render notes in response detail/modal UI.
5. Ensure public v2 response endpoint ignores notes entirely.
**Tips:**
- Response model already relates to survey and tags.
- Reuse `requireV3WorkspaceAccess` or action-client authorization patterns.
- User-facing timestamps need shared formatting helpers.
**What Could Go Wrong:**
- Notes could leak through public response APIs.
- Notes could be created for responses in another environment if scoping is weak.
**Stretch Goal:** Allow deleting your own note with audit logging.  
**Connects To:** Story 10, because notes introduce collaboration and ownership concerns.

## Story 10: Design A Cached Workspace Survey Overview
**Difficulty:** Expert  
**Estimated Time:** 20 hours  
**Skills You'll Practice:** architecture, caching, authorization, performance, API design  
**The Story:** As a workspace manager, I want a fast workspace overview with survey counts by status and recent response volume so that I can understand health at a glance.  
**Acceptance Criteria:**
- [ ] Overview includes counts by survey status and recent response count.
- [ ] Data is scoped to the authorized workspace/environment.
- [ ] Expensive aggregation uses the repo's cache utilities rather than Next `unstable_cache`.
- [ ] Cache keys use `createCacheKey.*` utilities.
- [ ] Mutations that affect overview data invalidate or refresh the relevant cache.
- [ ] Tests cover auth, aggregation correctness, and cache behavior.
**Files You'll Likely Touch:** `apps/web/app/api/v3` for a new overview route; `apps/web/app/api/v3/lib/auth.ts` for workspace access pattern; `packages/cache` or `apps/web/lib/cache` for cache utilities; `packages/database/schema.prisma` only if new persistence is required; survey list/dashboard UI under `apps/web/modules/survey/list`; mutation paths like `apps/web/modules/survey/editor/actions.ts` and response submission route for invalidation.  
**High-Level Implementation Plan:**
1. Design the response contract and route path.
2. Implement authorized aggregation by `environmentId`.
3. Add cache key and cache wrapper using existing cache utilities.
4. Invalidate on survey status changes and response creation.
5. Add UI overview panel above survey list.
6. Add route/service tests and a Playwright smoke path if UI is added.
**Tips:**
- Do not use Next `unstable_cache`; repo guidance forbids it.
- `Response` has indexes on `surveyId` and `createdAt`.
- `Survey` has `@@index([environmentId, updatedAt])`.
- Treat locale and time zone separately if showing dates.
**What Could Go Wrong:**
- Cache keys could leak data across workspaces if environment id is omitted.
- Counts could become stale if response creation does not invalidate.
- Aggregations could become slow if they scan all responses without indexed filters.
**Stretch Goal:** Add trend comparison versus the previous period.  
**Connects To:** This is the capstone: it uses routing, auth, API contracts, Prisma, caching, UI, tests, and review judgment.

