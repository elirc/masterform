# Good First Tickets

Eighteen junior-scoped tickets spread across the repo. Each is small-blast-radius, follows an existing pattern, and is testable. Before starting any: read the anchors, run the neighboring tests, branch. These are *training tickets* — some would be accepted upstream, some are for practice; the "why a maintainer might reject" line tells you which.

Template fields are abbreviated; expand any ticket into the full template from the curriculum root prompt when you work it.

---

## Ticket 1: Add context to the pipeline failure log
Difficulty: Easy — 1h. Skills: structured logging.
Story: As an operator, I want pipeline send failures to log `surveyId` and `event` so I can correlate lost events.
Anchors: `apps/web/app/lib/pipelines.ts:22-24` (bare error log); style reference `apps/web/app/api/(internal)/pipeline/route.ts:174-176`.
Plan: change `.catch((error) => logger.error(error, "..."))` to include `{ surveyId, event, environmentId }` context object. Add/extend a colocated test asserting the logger call (mock `@formbricks/logger`).
Could go wrong: logging the full `response` object (PII!) — log IDs only.
Review questions: does the logger's first-arg convention (object vs error) match house style?
Rejection risk: low — pure observability win.
Interview story potential: "I improved failure observability on an at-most-once event path" → leads into the outbox conversation.

## Ticket 2: Unit-test `getCountry` header precedence
Difficulty: Easy — 1h. Skills: vitest, HTTP headers.
Anchors: `apps/web/app/api/v2/client/[environmentId]/responses/route.ts:37-41`.
Plan: the function isn't exported — either test through the route or export it; prefer exporting a pure helper into `lib/` matching neighbors. Cases: CF wins over Vercel; CloudFront fallback; none → undefined.
Rejection risk: refactor-for-testability must stay tiny or it becomes a drive-by.
Interview story: "testing pure logic extracted from a route handler."

## Ticket 3: Document the pipeline contract
Difficulty: Easy — 2h. Skills: technical writing, types-as-docs.
Anchors: `apps/web/app/lib/pipelines.ts`, `apps/web/app/api/(internal)/pipeline/types/pipelines.ts` (ZPipelineInput).
Plan: JSDoc on `sendToPipeline` stating: delivery is at-most-once, auth is CRON_SECRET, dates serialize to strings (consumer re-hydrates, `pipeline/route.ts:39-42`). No behavior change.
Rejection risk: low; docs PRs need to be *accurate* — cite lines in the PR body.
Interview story: "I documented an implicit contract I had to reverse-engineer."

## Ticket 4: Return `Retry-After` header on rate-limited storage uploads
Difficulty: Medium — 3h. Skills: HTTP semantics, wrapper code.
Anchors: `apps/web/modules/core/rate-limit/rate-limit.ts:83-86` (retryAfter already computed); `apps/web/app/lib/api/with-api-logging.ts` (where 429s are built — verify exact spot before starting).
Plan: thread `retryAfter` into the 429 response as a standard header. Test: simulate limit exceeded, assert header.
Could go wrong: v1/v2/v3 wrappers differ — scope to one wrapper and say so in the PR.
Interview story: "small API-correctness fix touching a cross-cutting wrapper safely."

## Ticket 5: Add a missing test for single-use URL parsing
Difficulty: Medium — 3h. Skills: security-adjacent testing.
Anchors: `apps/web/app/api/client/[environmentId]/responses/lib/single-use.ts:14-80` (+ its existing test file if present — check first).
Plan: table-driven tests: missing suId, missing suToken, malformed URL, valid encrypted pair. Use the repo's crypto helpers to build valid fixtures.
Interview story: "I wrote adversarial tests around token validation."

## Ticket 6: Improve `checkSurveyValidity` error for paused surveys
Difficulty: Easy — 2h. Skills: API error design, i18n awareness.
Anchors: `apps/web/app/api/v2/client/[environmentId]/responses/lib/utils.ts:22-26` — every non-inProgress status returns the same "not accepting submissions".
Plan: include `status` in the error details object (already returns surveyId). Widget copy stays unchanged; this only helps API consumers debug.
Rejection risk: check the public-API stability policy — additive detail fields are usually safe.

## Ticket 7: Storybook story for a survey-ui component
Difficulty: Easy — 2h. Skills: component isolation.
Anchors: `apps/storybook/` (find an existing story as template), `packages/survey-ui/`.
Plan: pick one un-storied component (verify by grep), add a story with 2–3 states.
Interview story: "component-driven development in a monorepo."

## Ticket 8: Add index usage comment or missing index — responses by environment
Difficulty: Medium — 3h (investigation-heavy). Skills: query planning.
Anchors: `packages/database/schema.prisma:187-189` (existing Response indexes + the maintainers' habit of commenting *why*); response list queries in `apps/web/lib/response/service.ts`.
Plan: find one frequent query whose WHERE/ORDER BY isn't covered; propose the index in a migration via `pnpm fb-migrate-dev` (inferred command, root `package.json:38`). If all are covered, write up the proof instead — that's a valid outcome.
Rejection risk: indexes without EXPLAIN evidence get rejected; bring numbers from a seeded local DB.
Interview story: strong one — "I proposed/rejected an index with EXPLAIN evidence."

## Ticket 9: Harden `convertDatesInObject` with a unit test for the skip-set
Difficulty: Easy — 2h.
Anchors: `apps/web/lib/time.ts` (the helper), `apps/web/app/api/(internal)/pipeline/route.ts:39-42` (the skip-set call).
Plan: test that fields in the Set (contactAttributes, variables, data, meta) are NOT date-coerced while siblings are.
Interview story: "serialization boundaries and defensive tests."

## Ticket 10: Add `locale` fallback test for follow-up emails
Difficulty: Medium — 3h.
Anchors: `apps/web/modules/survey/follow-ups/lib/email.ts`, `follow-ups.ts:21-70`.
Plan: verify behavior when a recipient user has no locale — what renders? Add the test documenting whichever behavior exists; fix only if it crashes.
Interview story: "i18n edge cases in transactional email."

## Ticket 11: Type-tighten a `catch (error: any)` to `unknown`
Difficulty: Easy — 1-2h. Skills: TS narrowing.
Anchors: `apps/web/app/api/v1/auth.ts:55` (`handleErrorResponse = (error: any)`); grep `error: any` for more.
Plan: switch to `unknown` + proper narrowing (`instanceof`, `in` checks). One file per PR.
Rejection risk: near zero if the diff stays surgical.
Interview story: "incremental type-safety hardening."

## Ticket 12: Add OPTIONS CORS test for the responses route
Difficulty: Easy — 2h.
Anchors: `apps/web/app/api/v2/client/[environmentId]/responses/route.ts:194-202`.
Plan: assert status, cache-control header value, and body shape. Mirrors existing route tests.

## Ticket 13: Extract magic TTL numbers to named constants
Difficulty: Easy — 2h.
Anchors: `environmentState.ts:69` (`60 * 1000`), `environment/route.ts:63` (1h expiresAt), cache headers :66-74.
Plan: `ENVIRONMENT_STATE_TTL_MS`, `SDK_REFETCH_INTERVAL_MS` in the lib file — the three layers' relationship becomes greppable. No behavior change; tests unchanged.
Interview story: small, but feeds the "three-layer cache" narrative you'll tell anyway.

## Ticket 14: Rate-limit config for a missed namespace
Difficulty: Medium — 3h (investigation).
Anchors: `apps/web/modules/core/rate-limit/rate-limit-configs.ts:1-36`; grep `applyRateLimit` call sites.
Plan: find one sensitive server action without a limiter (email-sending and account actions are covered — verify what isn't, e.g. invite creation). Add config + call following `forgotPassword`'s example.
Rejection risk: product owners own the budgets — propose numbers, ask in the PR.
Interview story: "abuse-surface analysis across an app."

## Ticket 15: Playwright: assert survey-closed message
Difficulty: Medium — 4h. Skills: e2e.
Anchors: `apps/web/playwright/survey.spec.ts` (patterns/fixtures), `surveyClosedMessage` field (`schema.prisma:383`).
Plan: e2e that closes a survey then loads the link page and asserts the closed message renders. Reuse existing fixtures.
Interview story: "e2e coverage for a state machine edge."

## Ticket 16: Widget: surface quotaFull to the embedding page
Difficulty: Medium — 4h. Skills: SDK events.
Anchors: `packages/surveys/src/lib/response-queue.ts:27` (`onQuotaFull` callback exists); check what js-core exposes publicly (`packages/js-core/src/index.ts`).
Plan: if not already public, wire an event/callback so host pages can react when a quota ends a survey. Docs-only if it exists.
Rejection risk: public SDK surface — needs maintainer buy-in; write the proposal issue first (see 07-03).

## Ticket 17: Add a health-check assertion for Redis to /health
Difficulty: Medium — 3h.
Anchors: `apps/web/app/api/v2/health/route.ts` (see also `apps/web/app/health`), `packages/cache/src/client.ts` (isReady/isOpen checks).
Plan: report cache reachability in the health payload (degraded, not failing — remember fail-open philosophy). Test both states with a mocked client.
Interview story: "operational readiness — I made a silent dependency visible."

## Ticket 18: Fix-or-document the `slug` uniqueness error path
Difficulty: Medium — 3h.
Anchors: `schema.prisma:414` (`slug String? @unique` on Survey), slug module `apps/web/modules/survey/slug/`.
Plan: submit a duplicate slug via the editor — what does the user see? If a raw Prisma error leaks, map it to a friendly validation message following the `UniqueConstraintError` pattern (`.../responses/route.ts:180-182`). If handled, add the missing test.
Interview story: "constraint-violation UX — DB errors as domain errors."

---

Every ticket: run `pnpm lint` and the targeted tests (__inferred__ commands — see [09-reference/command-cheatsheet.md](../09-reference/command-cheatsheet.md)) before opening the PR, and write the PR body per [07-career-and-collaboration/02-writing-prs-and-rfcs.md](../07-career-and-collaboration/02-writing-prs-and-rfcs.md).
