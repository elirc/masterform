# Junior Developer Onboarding User Stories

This file contains 20 contribution stories designed to train a junior engineer on this codebase. Stories 1-10 are guided follow-along tickets with detailed implementation plans. Stories 11-20 are intentionally higher level so the junior developer has room to investigate, design, and make decisions independently.

The stories are ordered to move from small UI changes to full-stack work. The goal is not just to ship changes. The goal is to learn how Formbricks organizes product routes, feature modules, shared UI, server actions, route handlers, API clients, React Query, Prisma-backed services, tests, and localization.

## Guided Stories 1-10

These first ten stories include detailed implementation plans. A junior developer should be able to follow them while still reading the code carefully and making small engineering decisions.

## Story 1: Replace The Missing Creator Dash With A Clear Label

**Difficulty:** Easy  
**Estimated Time:** 1 hour  
**Primary Learning Goal:** Learn a small client component, typed props, i18n, and safe null handling.

**The Story:** As a workspace member viewing the surveys list, I want surveys without a known creator to show a clear label so that the row looks intentional instead of incomplete.

**Acceptance Criteria:**

- [ ] If `survey.creator` exists, the survey card still shows the creator name.
- [ ] If `survey.creator` is `null`, the survey card shows a translated fallback such as "Unknown creator" or "System".
- [ ] The fallback text fits inside the existing creator column and does not change the row layout.
- [ ] The new text is added to `apps/web/locales/en-US.json`.

**Files You'll Likely Touch:**

- `apps/web/modules/survey/list/components/survey-card.tsx` - renders creator name in the survey list card.
- `apps/web/locales/en-US.json` - stores the English i18n key.

**Detailed Implementation Plan:**

1. Open `apps/web/modules/survey/list/components/survey-card.tsx`.
2. Find the creator rendering near the bottom of the card body. It currently renders `survey.creator ? survey.creator.name : "-"`.
3. Confirm `SurveyCard` already calls `const { t } = useTranslation();`. You do not need to import a new translation hook.
4. Choose a key name that follows the repo pattern, for example `environments.surveys.unknown_creator`.
5. Replace the `"-"` fallback with `t("environments.surveys.unknown_creator")`.
6. Open `apps/web/locales/en-US.json`.
7. Add the new key under the most appropriate existing `environments.surveys` section. If the exact nesting is hard to find, search for nearby keys such as `new_survey`, `no_surveys_found`, or `no_surveys_created_yet`.
8. Run a quick search to make sure the key is spelled exactly the same in both files.
9. Manually inspect the row structure in `SurveyCard` and confirm you did not alter the grid column count.
10. If you run tests, prefer a targeted lint/type check or existing survey list tests. This is a tiny UI text change, so a full test suite is not necessary unless your PR process requires it.

**Learning Checkpoints:**

- Explain why the component should not guess a creator from the logged-in user.
- Explain why the fallback text belongs in `en-US.json`.
- Explain how `TSurveyListItem` makes `creator` nullable.

**Common Mistakes:**

- Adding raw English text directly in JSX.
- Adding a translation key in the wrong namespace and mistyping it in the component.
- Changing the grid layout for a text-only change.

## Story 2: Add A Draft Status Tooltip

**Difficulty:** Easy  
**Estimated Time:** 1.5 hours  
**Primary Learning Goal:** Learn conditional rendering, shared UI components, and i18n.

**The Story:** As a survey builder, I want the draft status icon to explain itself on hover so that I can quickly understand which surveys still need work.

**Acceptance Criteria:**

- [ ] `SurveyStatusIndicator` shows explanatory tooltip text for `draft` when `tooltip` is enabled.
- [ ] Existing tooltip behavior for `inProgress`, `paused`, and `completed` remains unchanged.
- [ ] Non-tooltip draft rendering still uses the pencil icon.
- [ ] Tooltip text uses `t()`.

**Files You'll Likely Touch:**

- `apps/web/modules/ui/components/survey-status-indicator/index.tsx` - shared status icon component.
- `apps/web/locales/en-US.json` - translation text for the draft tooltip.

**Detailed Implementation Plan:**

1. Open `apps/web/modules/ui/components/survey-status-indicator/index.tsx`.
2. Read the whole component before editing. Notice there are two branches:
   - `if (tooltip)` renders status icons inside `TooltipProvider`, `Tooltip`, `TooltipTrigger`, and `TooltipContent`.
   - The `else` branch renders status icons without tooltip content.
3. In the tooltip trigger area, confirm that draft currently renders an icon in the trigger branch.
4. In the tooltip content area, look for the conditional text for `inProgress`, `paused`, and `completed`.
5. Add a `draft` branch in the tooltip content area. Keep the structure similar to the other statuses.
6. Use `PencilIcon` for draft so tooltip and non-tooltip draft visuals agree.
7. Add a translation key such as `common.survey_draft` or `common.draft_survey`. Prefer an existing `common.*` naming pattern if one exists nearby.
8. Add the new key to `apps/web/locales/en-US.json`.
9. Check that you did not remove or change the non-tooltip draft branch.
10. If you want an optional cleanup, make a note that this component could eventually use a status config object, but do not refactor in this story unless the change stays very small.

**Learning Checkpoints:**

- Explain why duplicated status branches can drift over time.
- Explain when a shared UI component should receive props versus reading global state.
- Explain why all user-facing tooltip text must be translated.

**Common Mistakes:**

- Adding tooltip text only for the trigger but not the content.
- Changing icon sizes for one status and making the row visually uneven.
- Refactoring the whole component during a small onboarding ticket.

## Story 3: Make The Filtered Empty State More Helpful

**Difficulty:** Easy-Medium  
**Estimated Time:** 2 hours  
**Primary Learning Goal:** Learn UI state branching and existing survey list behavior.

**The Story:** As a workspace member filtering surveys, I want the empty state to tell me when no surveys match my filters so that I know I should adjust filters rather than create a new survey.

**Acceptance Criteria:**

- [ ] When filters are active and there are no matching surveys, the empty state says no surveys match the current filters.
- [ ] When there are no surveys at all and no filters are active, the existing template empty state still appears for users who can create surveys.
- [ ] Read-only users with no surveys and no filters still see the read-only empty state.
- [ ] All new text is translated.

**Files You'll Likely Touch:**

- `apps/web/modules/survey/list/components/survey-list.tsx` - owns survey list states.
- `apps/web/locales/en-US.json` - new empty-state text.

**Detailed Implementation Plan:**

1. Open `apps/web/modules/survey/list/components/survey-list.tsx`.
2. Find these derived values:
   - `hasAppliedFilters`
   - `showInitialLoading`
   - `showTemplateEmptyState`
   - `showReadOnlyEmptyState`
3. Find the default `surveyContent` block that currently renders the generic no-surveys-found state.
4. Add a new derived boolean such as `showFilteredEmptyState`.
5. The boolean should be true when:
   - the page is not loading,
   - there is no API error,
   - `surveys.length === 0`,
   - `hasAppliedFilters` is true.
6. Add a new branch before the generic no-surveys content or replace the default `surveyContent` when `showFilteredEmptyState` is true.
7. Keep the existing `showTemplateEmptyState` and `showReadOnlyEmptyState` branches before the main list body. They represent different product states.
8. Add translated text, for example `environments.surveys.no_surveys_match_filters`.
9. Verify mentally that the branch does not run during initial localStorage initialization.
10. If you want a small manual QA path, test with a name filter that cannot match any survey.

**Learning Checkpoints:**

- Explain the difference between "no surveys exist" and "no surveys match filters."
- Explain why `isFilterInitialized` matters.
- Explain why the template empty state should not appear for filtered results.

**Common Mistakes:**

- Showing the filtered empty state while data is still loading.
- Removing the template empty state by accident.
- Adding a clear-filters button before understanding how filters are normalized.

## Story 4: Show An Active Filter Count

**Difficulty:** Medium  
**Estimated Time:** 3 hours  
**Primary Learning Goal:** Learn derived state, normalization utilities, and small utility tests.

**The Story:** As a workspace member, I want to see how many survey filters are active so that I understand when my survey list is narrowed.

**Acceptance Criteria:**

- [ ] The survey list UI shows an active filter count when one or more filters are active.
- [ ] The count is hidden or visually zero when no filters are active.
- [ ] The count is based on normalized filters, not raw localStorage data.
- [ ] A utility test covers the count behavior if you add a new utility.

**Files You'll Likely Touch:**

- `apps/web/modules/survey/list/components/survey-list.tsx` - computes normalized filters and renders filter UI.
- `apps/web/modules/survey/list/components/survey-filters.tsx` - likely place to display the count.
- `apps/web/modules/survey/list/lib/utils.ts` - existing filter helper utilities.
- `apps/web/modules/survey/list/lib/utils.test.ts` - tests for filter utility behavior.
- `apps/web/locales/en-US.json` - translation text.

**Detailed Implementation Plan:**

1. Open `apps/web/modules/survey/list/lib/utils.ts`.
2. Read the existing functions around filter normalization and `hasActiveSurveyFilters`.
3. Decide whether to extend `hasActiveSurveyFilters` or add a new function such as `getActiveSurveyFilterCount`.
4. Use `initialFilters` as the baseline for "not active."
5. Count active concepts, not raw array lengths blindly:
   - non-empty name search counts as one active filter,
   - status choices count as one active filter group or individual statuses, depending on the UX you want,
   - type choices count similarly,
   - non-default sort counts if you decide sorting is a filter-like narrowing signal.
6. Add tests in `utils.test.ts`. Include at least:
   - initial filters returns `0`,
   - name filter returns `1`,
   - status and type together return the expected count,
   - invalid or channel-specific values are normalized before counting if applicable.
7. Open `survey-list.tsx` and compute the count from `normalizedFilters`.
8. Pass the count to `SurveyFilters` if that component owns the visible filter controls.
9. Open `survey-filters.tsx` and add a small translated count label.
10. Add translation keys to `en-US.json`.
11. Run the targeted utility test if possible, for example the package's Vitest command or a focused test invocation.

**Learning Checkpoints:**

- Explain why normalized filters are safer than raw filters.
- Explain the difference between a utility function and component-specific logic.
- Explain how you decided what counts as one filter.

**Common Mistakes:**

- Counting default values as active filters.
- Forgetting to update tests when changing utility behavior.
- Putting all count logic directly in JSX, making it hard to test.

## Story 5: Add A Manual Refresh Button To The Survey List

**Difficulty:** Medium  
**Estimated Time:** 3 hours  
**Primary Learning Goal:** Learn React Query refetching and loading states.

**The Story:** As a workspace member, I want to manually refresh the survey list so that I can confirm recent survey changes without reloading the whole page.

**Acceptance Criteria:**

- [ ] A refresh button appears near the survey filters or page header.
- [ ] Clicking it calls the existing React Query `refetch`.
- [ ] The button shows a loading state while the refresh is running.
- [ ] Existing error retry behavior continues to work.
- [ ] Text or tooltip uses translations.

**Files You'll Likely Touch:**

- `apps/web/modules/survey/list/components/survey-list.tsx` - receives `refetch` from `useSurveys`.
- `apps/web/modules/ui/components/button/index.tsx` - existing shared button supports loading.
- `apps/web/locales/en-US.json` - refresh label or tooltip.

**Detailed Implementation Plan:**

1. Open `apps/web/modules/survey/list/components/survey-list.tsx`.
2. Find the destructuring result from `useSurveys`. It already pulls out values like `error`, `fetchNextPage`, `isLoading`, `refetch`, `surveys`, and `totalCount`.
3. Check whether the query object exposes an appropriate loading flag for refetching, such as `isFetching` or similar from TanStack Query. If it is not already destructured, add it.
4. Decide where the refresh control belongs. The safest first version is near `SurveyFilters`, inside the existing page content area.
5. Import a refresh icon from `lucide-react`, such as `RefreshCwIcon`, if available in the installed icon package.
6. Add a `Button` with:
   - `variant="secondary"` or another existing variant,
   - `size="sm"`,
   - `loading={isFetching && !isFetchingNextPage}` or another condition that does not confuse load-more pagination with manual refresh,
   - `onClick={() => refetch()}`.
7. Add a translated label or accessible text. If the button is icon-only, make sure it has an `aria-label`.
8. Do not create a new fetch function. The learning goal is to reuse the existing React Query source of truth.
9. Confirm the error branch still renders the existing "try again" button and still calls `refetch`.
10. Manually QA by clicking refresh with existing filters applied.

**Learning Checkpoints:**

- Explain why `refetch` is better than directly calling `listSurveys` from the component.
- Explain why refresh loading and load-more loading are not the same state.
- Explain what React Query cache key is being refreshed.

**Common Mistakes:**

- Creating a second data-fetching path and bypassing React Query.
- Showing the button before filters are initialized and causing a confusing request.
- Using a raw English `aria-label`.

## Story 6: Add A Copy Survey ID Menu Action

**Difficulty:** Medium  
**Estimated Time:** 3 hours  
**Primary Learning Goal:** Learn dropdown actions, browser APIs, and toast/error feedback.

**The Story:** As a support engineer, I want to copy a survey ID from the survey list so that I can quickly reference it in debugging or customer support.

**Acceptance Criteria:**

- [ ] Each survey card menu includes a "Copy survey ID" action.
- [ ] Clicking the action copies `survey.id` to the clipboard.
- [ ] The user gets success feedback.
- [ ] Clipboard failure shows a translated error message or fallback.
- [ ] The action works for read-only users because it does not mutate data.

**Files You'll Likely Touch:**

- `apps/web/modules/survey/list/components/survey-dropdown-menu.tsx` - survey card menu actions.
- `apps/web/modules/survey/list/components/survey-card.tsx` - passes survey data to the menu.
- `apps/web/locales/en-US.json` - labels and feedback messages.

**Detailed Implementation Plan:**

1. Open `apps/web/modules/survey/list/components/survey-dropdown-menu.tsx`.
2. Read the current menu actions before editing. Identify how actions are grouped, how disabled actions are represented, and how translated labels are used.
3. Confirm the component receives the full `survey` object or at least `survey.id`.
4. Add a new menu item labeled with a key like `environments.surveys.copy_survey_id`.
5. Implement a click handler:
   - call `navigator.clipboard.writeText(survey.id)`,
   - await the promise,
   - show success feedback using the existing toast pattern in this file or nearby modules.
6. If `navigator.clipboard` is unavailable or throws, show a translated error toast.
7. Do not gate this action behind `isSurveyCreationDeletionDisabled`; copying an ID should remain available to read-only users.
8. Add translation keys for label, success, and failure.
9. Manually inspect keyboard behavior: menu item should be reachable like other dropdown items.
10. If existing tests cover this component or menu behavior, update them. If not, note that a Playwright smoke test would be appropriate for clipboard behavior.

**Learning Checkpoints:**

- Explain why copy ID is a read-only action.
- Explain why clipboard APIs are asynchronous.
- Explain how the menu currently handles destructive versus non-destructive actions.

**Common Mistakes:**

- Disabling the copy action for read-only users.
- Not handling clipboard rejection.
- Adding support/debugging text without translation.

## Story 7: Add A Unit Test For Survey List Search Param Building

**Difficulty:** Medium  
**Estimated Time:** 3 hours  
**Primary Learning Goal:** Learn small utility tests and API-client contracts.

**The Story:** As a maintainer, I want survey list URL parameters to be tested so that filter changes do not silently break the v3 survey list API.

**Acceptance Criteria:**

- [ ] `buildSurveyListSearchParams` has tests for workspace id, limit, sort, name, status, and type filters.
- [ ] The test covers `includeTotalCount=false`.
- [ ] The test covers cursor inclusion.
- [ ] Existing tests still pass.

**Files You'll Likely Touch:**

- `apps/web/modules/survey/list/lib/v3-surveys-client.ts` - function under test.
- `apps/web/modules/survey/list/lib/v3-surveys-client.test.ts` - existing test file for search params.

**Detailed Implementation Plan:**

1. Open `apps/web/modules/survey/list/lib/v3-surveys-client.ts`.
2. Read `buildSurveyListSearchParams` from top to bottom.
3. Open `apps/web/modules/survey/list/lib/v3-surveys-client.test.ts`.
4. Read the existing tests. Do not duplicate cases that already exist unless you are strengthening assertions.
5. Add a test case that passes:
   - a fake `workspaceId`,
   - a numeric `limit`,
   - a `cursor`,
   - `includeTotalCount: false`,
   - a filter object with `name`, multiple statuses, multiple types, and a non-default `sortBy`.
6. Assert the serialized search params contain:
   - `workspaceId`,
   - `limit`,
   - `cursor`,
   - `includeTotalCount=false`,
   - `sortBy`,
   - `filter[name][contains]`,
   - repeated `filter[status][in]`,
   - repeated `filter[type][in]`.
7. Use `searchParams.getAll(...)` for repeated params rather than checking the raw string order.
8. Run the targeted test if possible.
9. If a test fails because normalization changes values, inspect `normalizeSurveyFilters` before changing assertions.
10. Leave production code unchanged unless the test reveals a real bug.

**Learning Checkpoints:**

- Explain why repeated query params need `getAll`.
- Explain how frontend API client tests protect backend route expectations.
- Explain why test assertions should avoid depending on query param order.

**Common Mistakes:**

- Testing the full query string and making the test order-sensitive.
- Recreating normalization logic inside the test.
- Modifying production code just to satisfy an unclear test.

## Story 8: Add A Route Test For V3 Survey Delete Unauthorized Access

**Difficulty:** Medium  
**Estimated Time:** 4 hours  
**Primary Learning Goal:** Learn route tests, auth mocking, and resource-leak protection.

**The Story:** As a security-conscious maintainer, I want delete-survey authorization failures tested so that users cannot delete surveys outside their workspace.

**Acceptance Criteria:**

- [ ] The v3 delete survey route returns a forbidden response when workspace authorization fails.
- [ ] The test verifies `deleteSurvey` is not called on authorization failure.
- [ ] The response shape matches the v3 problem response style.
- [ ] Existing delete success behavior remains unchanged.

**Files You'll Likely Touch:**

- `apps/web/app/api/v3/surveys/[surveyId]/route.ts` - route under test.
- `apps/web/app/api/v3/surveys/[surveyId]/route.test.ts` - existing route tests.

**Detailed Implementation Plan:**

1. Open `apps/web/app/api/v3/surveys/[surveyId]/route.ts`.
2. Trace the route:
   - validates `surveyId`,
   - loads survey,
   - calls `requireV3WorkspaceAccess`,
   - sets audit metadata,
   - calls `deleteSurvey`.
3. Open `apps/web/app/api/v3/surveys/[surveyId]/route.test.ts`.
4. Study how the test file mocks:
   - session or API key auth,
   - `getSurvey`,
   - `requireV3WorkspaceAccess`,
   - `deleteSurvey`.
5. Add or extend a test where:
   - `getSurvey` returns a valid survey,
   - `requireV3WorkspaceAccess` returns a `Response` with forbidden status,
   - the route returns that response,
   - `deleteSurvey` is not called.
6. Assert the HTTP status is `403`.
7. Parse the JSON body and assert it has v3 problem fields such as `title`, `status`, `code`, or `requestId`, matching existing helper behavior.
8. Run the route test file if possible.
9. If mocks are difficult, copy the style from existing tests in the same file rather than inventing a new mocking setup.
10. Do not change production code unless the route actually deletes before authorization.

**Learning Checkpoints:**

- Explain why the route loads the survey before workspace access.
- Explain why failed authorization should not reveal too much about resource existence.
- Explain why asserting `deleteSurvey` is not called is important.

**Common Mistakes:**

- Only checking status code and forgetting to verify no mutation happened.
- Mocking too much and no longer testing the actual route behavior.
- Returning `404` for an auth-sensitive resource.

## Story 9: Add A Small "Last Updated" Helper Test

**Difficulty:** Medium  
**Estimated Time:** 3 hours  
**Primary Learning Goal:** Learn date display rules and utility testing.

**The Story:** As a survey dashboard user, I want updated times to stay locale-aware so that the list remains understandable across languages.

**Acceptance Criteria:**

- [ ] A targeted test covers the helper used for survey card relative updated time or date display.
- [ ] The test does not depend on the machine's browser locale.
- [ ] The implementation continues to use shared date/time helpers rather than `toLocaleString` directly in the component.

**Files You'll Likely Touch:**

- `apps/web/modules/survey/list/components/survey-card.tsx` - uses date helpers.
- `apps/web/lib/time.ts` and `apps/web/lib/time.test.ts` - relative time helper and tests.
- `apps/web/lib/utils/datetime.ts` and related tests if present - date display helper.

**Detailed Implementation Plan:**

1. Open `apps/web/modules/survey/list/components/survey-card.tsx`.
2. Find the date helpers used for `createdAt` and `updatedAt`.
3. Open the helper files, likely `apps/web/lib/time.ts` and `apps/web/lib/utils/datetime.ts`.
4. Open existing tests, especially `apps/web/lib/time.test.ts`.
5. Identify one behavior relevant to survey cards:
   - relative update text for a known timestamp,
   - date formatting for a known locale,
   - fallback behavior if a locale is invalid.
6. Add a test using explicit input dates and explicit locale values.
7. Avoid relying on "now" unless the test already uses fake timers.
8. If fake timers are already used in nearby tests, follow the existing setup and cleanup pattern.
9. Run the targeted test if possible.
10. Do not change `SurveyCard` unless you discover it bypasses shared helpers.

**Learning Checkpoints:**

- Explain why components should not call `new Intl.DateTimeFormat()` directly here.
- Explain the difference between locale and time zone.
- Explain how fake timers prevent flaky tests.

**Common Mistakes:**

- Writing a test that passes only in your local timezone.
- Using browser default locale implicitly.
- Testing the exact wording of relative time too rigidly if the helper intentionally localizes text.

## Story 10: Add A "Clear Filters" Action To The Survey List

**Difficulty:** Medium  
**Estimated Time:** 4 hours  
**Primary Learning Goal:** Learn controlled state, localStorage persistence, and UX flow.

**The Story:** As a workspace member, I want to clear all survey filters with one click so that I can quickly return to the full survey list.

**Acceptance Criteria:**

- [ ] A clear-filters action appears only when filters are active.
- [ ] Clicking it resets filters to `initialFilters`.
- [ ] Local storage is updated so the reset persists.
- [ ] The survey list refetches with cleared filters.
- [ ] Text is translated.

**Files You'll Likely Touch:**

- `apps/web/modules/survey/list/components/survey-list.tsx` - owns `surveyFilters` state.
- `apps/web/modules/survey/list/components/survey-filters.tsx` - likely owns visible filter controls.
- `apps/web/modules/survey/list/lib/constants.ts` - exports `initialFilters`.
- `apps/web/modules/survey/list/lib/utils.ts` - active-filter helper.
- `apps/web/locales/en-US.json` - button text.

**Detailed Implementation Plan:**

1. Open `apps/web/modules/survey/list/components/survey-list.tsx`.
2. Confirm `surveyFilters` and `setSurveyFilters` are owned by this component.
3. Confirm `normalizedFilters` is written to localStorage in an effect after filters are initialized.
4. Decide whether the clear button belongs in `SurveyList` or inside `SurveyFilters`.
5. Prefer passing a callback into `SurveyFilters` if the filter controls live there.
6. Add a callback such as `handleClearFilters`:
   - call `setSurveyFilters(initialFilters)`,
   - do not manually call localStorage unless the existing effect does not run in this path.
7. Show the button only when `hasAppliedFilters` is true.
8. Use `Button` with a translated label such as `common.clear_filters` or a more specific survey key.
9. Confirm React Query refetches because `normalizedFilters` changes and the query key changes.
10. Add or update utility tests only if you change filter helper behavior.
11. Manually QA:
   - apply a filter,
   - reload page and confirm it persists,
   - click clear,
   - reload again and confirm filters remain cleared.

**Learning Checkpoints:**

- Explain why changing filters changes the React Query key.
- Explain why localStorage should be synchronized through one effect, not several scattered writes.
- Explain why the clear button should be hidden when no filters are active.

**Common Mistakes:**

- Clearing visible UI but leaving localStorage stale.
- Calling `refetch` manually when the query key change already handles it.
- Resetting to a hand-written object instead of `initialFilters`.

## Independent Stories 11-20

These ten stories intentionally use high-level plans. The junior developer should investigate the codebase, identify exact implementation details, and ask focused questions only after attempting the trace.

## Story 11: Add A Survey List "Updated By" Column

**Difficulty:** Medium  
**Estimated Time:** 5 hours  
**Primary Learning Goal:** Trace a field from database query to API response to UI.

**The Story:** As a workspace member, I want to see who last updated a survey so that I know who to ask about recent changes.

**Acceptance Criteria:**

- [ ] Survey list rows show last updater when available.
- [ ] The field is omitted or shows a translated fallback when unavailable.
- [ ] API response does not expose unnecessary user fields.
- [ ] Tests cover serialization or mapping changes.

**Files You'll Likely Touch:**

- `packages/database/schema.prisma`
- `apps/web/modules/survey/list/lib/survey-record.ts`
- `apps/web/app/api/v3/surveys/serializers.ts`
- `apps/web/modules/survey/list/types/survey-overview.ts`
- `apps/web/modules/survey/list/components/survey-card.tsx`

**High-Level Implementation Plan:**

1. Investigate whether the database already tracks last updater or only creator.
2. If the field exists, add it to the list select and mapping path.
3. If it does not exist, write down the migration/design implications before coding.
4. Update API serialization and frontend type contracts.
5. Render the value with a translated fallback and add tests.

## Story 12: Add A Survey Type Filter Shortcut

**Difficulty:** Medium  
**Estimated Time:** 4 hours  
**Primary Learning Goal:** Work with existing filter state and UI controls.

**The Story:** As a workspace member, I want a quick toggle for link surveys and app surveys so that I can narrow the list faster.

**Acceptance Criteria:**

- [ ] The shortcut updates the same filter state used by the existing survey filters.
- [ ] The selected shortcut persists through localStorage like other filters.
- [ ] The API request carries the expected `filter[type][in]` params.

**Files You'll Likely Touch:**

- `apps/web/modules/survey/list/components/survey-filters.tsx`
- `apps/web/modules/survey/list/components/survey-list.tsx`
- `apps/web/modules/survey/list/lib/v3-surveys-client.ts`
- `apps/web/modules/survey/list/lib/utils.ts`

**High-Level Implementation Plan:**

1. Inspect current survey filter controls and type filter behavior.
2. Add a shortcut UI that updates the existing `surveyFilters.type` array.
3. Ensure normalization still works for each project channel.
4. Add or update tests around search param generation.

## Story 13: Add A Safer Confirmation Message For Delete Survey

**Difficulty:** Medium  
**Estimated Time:** 4 hours  
**Primary Learning Goal:** Learn destructive action UX and mutation flow.

**The Story:** As a workspace manager, I want the delete survey confirmation to mention response deletion risk so that I understand the impact before confirming.

**Acceptance Criteria:**

- [ ] Delete confirmation copy explicitly mentions survey responses.
- [ ] Copy is translated.
- [ ] Delete behavior and permissions remain unchanged.
- [ ] Read-only users still cannot delete surveys.

**Files You'll Likely Touch:**

- `apps/web/modules/survey/list/components/survey-dropdown-menu.tsx`
- `apps/web/modules/survey/list/hooks/use-delete-survey.ts`
- `apps/web/locales/en-US.json`

**High-Level Implementation Plan:**

1. Find the current delete confirmation modal/menu flow.
2. Update the warning copy without changing mutation behavior.
3. Check read-only disabled behavior.
4. Manually verify confirmation and cancellation paths.

## Story 14: Add API Docs Notes For V3 Survey List Filters

**Difficulty:** Medium  
**Estimated Time:** 4 hours  
**Primary Learning Goal:** Connect implementation to external API documentation.

**The Story:** As an API user, I want clear documentation for v3 survey list filters so that I can query surveys correctly.

**Acceptance Criteria:**

- [ ] Documentation lists supported query params.
- [ ] Documentation includes examples for name, status, type, cursor, and total count.
- [ ] Documentation matches actual parser behavior.

**Files You'll Likely Touch:**

- `docs/api-v3-reference`
- `apps/web/app/api/v3/surveys/parse-v3-surveys-list-query.ts`
- `apps/web/modules/survey/list/lib/v3-surveys-client.ts`

**High-Level Implementation Plan:**

1. Inspect existing v3 API docs structure.
2. Read the query parser and API client to confirm actual parameter names.
3. Add or update survey list filter documentation.
4. Include examples that match real query param names.

## Story 15: Add A Workspace Access Test For V3 Survey List API Keys

**Difficulty:** Medium-Hard  
**Estimated Time:** 5 hours  
**Primary Learning Goal:** Learn API-key authorization and route tests.

**The Story:** As an API maintainer, I want API-key workspace access tests to cover missing environment permission so that integrations cannot read surveys from unauthorized workspaces.

**Acceptance Criteria:**

- [ ] Route test covers API key with no permission for the requested workspace.
- [ ] Response is forbidden.
- [ ] Survey query service is not called.
- [ ] Existing session-auth tests still pass.

**Files You'll Likely Touch:**

- `apps/web/app/api/v3/surveys/route.test.ts`
- `apps/web/app/api/v3/lib/auth.ts`
- `apps/web/app/api/v3/lib/auth.test.ts`

**High-Level Implementation Plan:**

1. Read existing route tests for API-key behavior.
2. Add a failing permission scenario.
3. Assert no data-read service runs after authorization failure.
4. Keep the test focused on route behavior, not Prisma internals.

## Story 16: Add An "Open Public Link" Action For Link Surveys

**Difficulty:** Medium-Hard  
**Estimated Time:** 6 hours  
**Primary Learning Goal:** Learn product-domain conditionals and safe URL construction.

**The Story:** As a workspace member, I want to open a link survey's public URL from the survey list so that I can preview or share it quickly.

**Acceptance Criteria:**

- [ ] The action appears only for link surveys where a public link can be constructed.
- [ ] The action opens the correct public survey URL.
- [ ] App surveys do not show the action.
- [ ] Text is translated.

**Files You'll Likely Touch:**

- `apps/web/modules/survey/list/components/survey-dropdown-menu.tsx`
- `apps/web/modules/survey/list/components/survey-card.tsx`
- `apps/web/modules/survey/list/types/survey-overview.ts`
- `apps/web/lib/getPublicUrl.ts`

**High-Level Implementation Plan:**

1. Investigate how public survey links are constructed elsewhere.
2. Check what fields the survey list item exposes for link survey URLs.
3. Add only the minimum data needed to construct the URL.
4. Render a non-mutating menu action for link surveys only.

## Story 17: Add A Small Health Check Test

**Difficulty:** Medium  
**Estimated Time:** 3 hours  
**Primary Learning Goal:** Learn minimal route-handler testing.

**The Story:** As a maintainer, I want the health route tested so that uptime checks do not accidentally regress.

**Acceptance Criteria:**

- [ ] The test calls the health route `GET` handler.
- [ ] The response status is successful.
- [ ] The JSON body equals `{ status: "ok" }`.

**Files You'll Likely Touch:**

- `apps/web/app/health/route.ts`
- `apps/web/app/health/route.test.ts`

**High-Level Implementation Plan:**

1. Inspect existing route test style in nearby API tests.
2. Create a small Vitest route test.
3. Import and call `GET`.
4. Assert status and body.

## Story 18: Add A Storage Not Configured Hint To File Upload Survey UI

**Difficulty:** Hard  
**Estimated Time:** 8 hours  
**Primary Learning Goal:** Trace a feature from survey element to shared UI/toast behavior.

**The Story:** As a survey builder, I want a clear hint when file upload storage is not configured so that I know why file upload questions will not work.

**Acceptance Criteria:**

- [ ] File upload survey configuration surfaces a translated storage warning when storage is unavailable.
- [ ] Existing storage-not-configured toast behavior is reused where appropriate.
- [ ] The warning does not appear for unrelated question types.

**Files You'll Likely Touch:**

- `apps/web/modules/ui/components/storage-not-configured-toast/index.tsx`
- `apps/web/modules/ui/components/storage-not-configured-toast/lib/utils.tsx`
- Survey editor components under `apps/web/modules/survey/editor`
- Storage service files under `apps/web/modules/storage`

**High-Level Implementation Plan:**

1. Find how file upload questions are configured in the survey editor.
2. Find how storage availability is currently checked.
3. Reuse existing storage warning components/utilities.
4. Add a targeted UI state for file upload configuration only.

## Story 19: Add Response Count To Survey Template Empty State

**Difficulty:** Hard  
**Estimated Time:** 7 hours  
**Primary Learning Goal:** Understand empty states, server data, and product messaging.

**The Story:** As a new workspace user, I want the empty survey state to explain that response counts will appear after publishing so that I understand what the dashboard will become.

**Acceptance Criteria:**

- [ ] The zero-survey template view includes a short translated message about responses.
- [ ] The message appears only in the true zero-survey state, not filtered empty state.
- [ ] The template preview remains functional.

**Files You'll Likely Touch:**

- `apps/web/modules/survey/list/components/survey-list.tsx`
- `apps/web/modules/survey/templates/components/template-container.tsx`
- `apps/web/locales/en-US.json`

**High-Level Implementation Plan:**

1. Trace the `showTemplateEmptyState` branch.
2. Inspect `TemplateContainerWithPreview` props and layout.
3. Add the message at the correct layer without disturbing template creation.
4. Manually verify the no-survey and filtered-empty paths.

## Story 20: Add A Workspace Survey Overview Panel

**Difficulty:** Hard  
**Estimated Time:** 10-14 hours  
**Primary Learning Goal:** Design a small full-stack feature with performance and auth in mind.

**The Story:** As a workspace manager, I want a compact overview of survey counts by status so that I can understand workspace health before scanning individual surveys.

**Acceptance Criteria:**

- [ ] Overview shows counts for draft, in-progress, paused, and completed surveys.
- [ ] Counts are scoped to the current environment/workspace.
- [ ] The data path does not duplicate large list queries unnecessarily.
- [ ] Authorization matches the survey list route.
- [ ] Tests cover aggregation and unauthorized access.

**Files You'll Likely Touch:**

- `apps/web/modules/survey/list/page.tsx`
- `apps/web/modules/survey/list/components/survey-list.tsx`
- `apps/web/modules/survey/list/lib/survey.ts` or a new focused overview service
- `apps/web/app/api/v3/surveys` or a new v3 overview route
- `apps/web/app/api/v3/lib/auth.ts`
- `packages/database/schema.prisma`

**High-Level Implementation Plan:**

1. Decide whether overview data should load server-side with the page or through a v3 client endpoint.
2. Implement a scoped aggregation by `environmentId`.
3. Reuse existing authorization patterns.
4. Render a compact overview panel above the list.
5. Add tests for counts and authorization.

