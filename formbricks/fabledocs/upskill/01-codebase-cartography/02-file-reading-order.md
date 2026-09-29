# File Reading Order

Thirty files, ordered so each read pays into the next. Per file: why, what to look for, what to skip. Junior path = 1–14. Mid path = 1–24. Senior path = all + the "senior noticing" notes.

## Stage A — Orientation (everyone)

1. `AGENTS.md` — maintainer conventions. Look for: the surveys-bundle cache warning; caching rules ("no unstable_cache"); i18n rules. Skip nothing; it's short.
2. `package.json` (root) — scripts are the operational API. Look for: `go`, `db:*`, `fb-migrate-dev`, `i18n`. Skip devDependencies.
3. `pnpm-workspace.yaml` + `turbo.json` — workspace + task graph. Look for: `onlyBuiltDependencies` (why pnpm blocks postinstall scripts). Skip fine detail.
4. `packages/database/schema.prisma:582-745` — Environment, Project, Organization, Membership. Look for: the tenancy chain and `type` production/development on Environment. Skip field-level docs on first pass.
5. `packages/database/schema.prisma:344-421` — Survey. Look for: how much lives in Json columns; `@@index([environmentId, updatedAt])`. 
6. `packages/database/schema.prisma:158-190` — Response + Display (:239-262). Look for: `@@unique([surveyId, singleUseId])`, index comments.
7. `apps/web/lib/constants.ts` — env-var surface. Look for: `CRON_SECRET`, `ENCRYPTION_KEY`, `WEBAPP_URL`, rate-limit flags. Skip the long tail.

## Stage B — The write path (everyone)

8. `apps/web/app/api/v2/client/[environmentId]/responses/route.ts` — full read. Look for: the validation funnel order; the unawaited `sendToPipeline` (:245).
9. `apps/web/app/api/v2/client/[environmentId]/responses/lib/utils.ts` — `checkSurveyValidity`. Look for: tenancy check first; the input mutation at :33.
10. `apps/web/app/api/v2/client/[environmentId]/responses/lib/response.ts:21-45` — the transaction. Look for: `tx` threading.
11. `apps/web/app/lib/pipelines.ts` — all 26 lines. Look for: what happens on failure (only a log).
12. `apps/web/app/api/(internal)/pipeline/route.ts` — full read, slowly; it's the densest file in the repo. Look for: CRON_SECRET auth; SSRF comments (:96-115); `allSettled`; autoComplete check-then-act (:273-301).

## Stage C — Auth in both universes (everyone)

13. `apps/web/lib/utils/action-client/index.ts` — the action pipeline. Look for: what `authenticatedActionClient` adds.
14. `apps/web/lib/utils/action-client/action-client-middleware.ts` — `checkAuthorizationUpdated`. Look for: OR-semantics; the weight tables.

*Junior path checkpoint: you can now do Flows 1, 2, 4. Go do the drills in 04-code-reading-gym before continuing.*

## Stage D — Read path + caching (mid)

15. `apps/web/app/api/v1/client/[environmentId]/environment/route.ts` — cache headers. Look for: the three TTLs story.
16. `apps/web/app/api/v1/client/[environmentId]/environment/lib/environmentState.ts` — `withCache`. Look for: the hidden write (:29-56).
17. `packages/cache/src/cache-keys.ts` — key registry. Look for: the branded `CacheKey` type.
18. `packages/cache/src/client.ts` — singleton + reconnection. Look for: `globalThis` stash (dev hot-reload survival); fail behavior.
19. `apps/web/modules/core/rate-limit/rate-limit.ts` + `rate-limit-configs.ts` — Look for: Lua atomicity; fail-open; the budget table.

## Stage E — The distributed edge (mid)

20. `packages/js-core/src/index.ts` — SDK public API. Look for: everything funnels into CommandQueue; legacy-init shim.
21. `packages/js-core/src/lib/common/command-queue.ts` — Look for: why commands serialize.
22. `packages/surveys/src/lib/response-queue.ts` — Look for: module-level locks comment (:36-40); IndexedDB mapping.
23. `apps/web/modules/auth/lib/authOptions.ts:168-320` — Look for: control hash; backup-code consumption.
24. `apps/web/app/api/v1/client/[environmentId]/storage/route.ts` — Look for: presigned flow; license-based size limit.

*Mid path checkpoint: you can now do all seven flows and the pattern catalog will read as recognition, not news.*

## Stage F — Judgment material (senior)

25. `apps/web/modules/ee/quotas/lib/utils.ts:100-180` — Look for: count-then-act inside tx. Senior noticing: what isolation level would make this safe? What's the cheapest fix?
26. `apps/web/modules/ee/license-check/lib/utils.ts` — Look for: how licenses cache (`createCacheKey.license.*`) and what fails open vs closed. Senior noticing: EE checks on hot public paths cost latency — where would you memoize?
27. `apps/web/modules/survey/editor/actions.ts` — full read. Senior noticing: how many permission systems one mutation consults (org role, project team, license, follow-ups grandfathering).
28. `apps/web/modules/survey/editor/components/survey-editor.tsx:86-113` — Look for: `localSurvey` via `structuredClone(survey)` — client-state copy of server data; who wins on conflict? Senior noticing: last-write-wins editing, no versioning — a product-level decision hiding in a useState.
29. `apps/web/modules/survey/list/hooks/use-surveys.ts` — Look for: `useInfiniteQuery` + cursor pagination + `surveyKeys` key factory; `includeTotalCount` only on first page (cost control).
30. `apps/web/app/lib/api/with-api-logging.ts` — Look for: the `{response, error}` contract that decides Sentry-vs-silence. Senior noticing: error *classification* as an architectural concern.

## How to read (the meta-skill)

For every file: first the imports (dependencies = coupling), then exports (contract), then one happy path top-to-bottom, then one failure path. Write one sentence: "This file owns ___ and must never ___." If you can't fill the second blank, you haven't finished reading. This habit — stating the invariant — is precisely what interviewers probe with "walk me through code you've read recently."
