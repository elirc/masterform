# Verification Log

Running record of what was inspected while authoring this curriculum. Date: 2026-07-11. Environment: Windows 11, app directory `formbricks/` (no `.git` of its own at the time, so git history was NOT used; all claims come from the working tree).

> **Status note (2026-10-06, frozen record).** Kept as written for the 2026-07-11 pass. The repository now has its own git history (a single snapshot commit, so still no upstream archaeology). A static re-check on 2026-10-06 (no installs, Docker, builds or tests) confirmed the inventory below still holds: 15 packages, 36 Prisma models in a 1,070-line schema with `Response` at :158 and `Survey` at :344, 378 colocated `*.test.ts(x)` files in `apps/web`, 20 workflow files, 23 Playwright `*.spec.ts` files (the "14+" below is a floor), and the "full" file sizes for the v2 responses route, pipeline route, storage route, action client and rate limiter. Sampled fast-track anchors (geo headers at `responses/route.ts:37-41`, `POST` at :204, pipeline send at :245, cron-secret check at `(internal)/pipeline/route.ts:31-36`, webhook `.catch` at :174-176, `sendToPipeline` catch at `pipelines.ts:22-24`) still point at the described code. One correction: the root `package.json` `scripts` block ends at L47, so `package.json:12-53` was narrowed to `:12-47` across the curriculum.

## Commands run (verified)

| Command | Result |
| --- | --- |
| `ls` on repo root, `apps/`, `packages/`, `docs/` | Confirmed monorepo layout: apps/web, apps/storybook, 15 packages, Mintlify docs site in docs/ |
| `cat package.json`, `cat pnpm-workspace.yaml`, `cat vitest.workspace.ts` | Confirmed scripts (db:up, dev, go, test, test:e2e, fb-migrate-dev, i18n) and workspace globs |
| `grep "^model " packages/database/schema.prisma` | 36 Prisma models enumerated; schema is 1070 lines |
| `ls packages/database/migration` | Migration directory exists (named `migration`, not `migrations`), timestamped folders from 2023-03 onward |
| `find apps/web/app/api/...` | Mapped v1/v2/v3 + client + (internal) route groups |
| `find apps/web -name "*.test.ts*" \| wc -l` | 378 colocated test files in apps/web alone |
| `ls apps/web/playwright` | 14+ Playwright specs (survey, signup, onboarding, storage-smoke, follow-up, …) |
| `ls .github/workflows` | 20 workflows incl. test.yml, e2e.yml, lint.yml, sonarqube.yml, docker builds |

## Commands NOT run (inferred only)

`pnpm install`, `pnpm db:up`, `pnpm dev`, `pnpm test`, `pnpm test:e2e`, `pnpm lint`, `pnpm build`, `pnpm db:migrate:dev`. Reason: authoring environment had no running Docker daemon and the task is docs-only. All are documented in `package.json:12-47` and `AGENTS.md` and are labeled __inferred__ wherever cited.

## Files read in full or in large part (anchors written from these reads)

- `AGENTS.md` (maintainer conventions, build/cache warnings for packages/surveys)
- `packages/database/schema.prisma` — model list; `Response` :158-190; `Survey` :344-421
- `apps/web/app/api/v2/client/[environmentId]/responses/route.ts` (full, 279 lines)
- `apps/web/app/api/v2/client/[environmentId]/responses/lib/utils.ts` (full, 74 lines)
- `apps/web/app/api/v2/client/[environmentId]/responses/lib/response.ts` :1-60
- `apps/web/app/lib/pipelines.ts` (full, 26 lines)
- `apps/web/app/api/(internal)/pipeline/route.ts` (full, 348 lines)
- `apps/web/app/api/client/[environmentId]/responses/lib/single-use.ts` :1-80
- `apps/web/app/api/v1/client/[environmentId]/environment/route.ts` :1-80
- `apps/web/app/api/v1/client/[environmentId]/environment/lib/environmentState.ts` (full, 72 lines)
- `apps/web/app/api/v1/client/[environmentId]/storage/route.ts` (full, 114 lines)
- `apps/web/app/api/v1/auth.ts` :1-60
- `apps/web/lib/utils/action-client/index.ts` (full, 66 lines)
- `apps/web/lib/utils/action-client/action-client-middleware.ts` :1-121
- `apps/web/modules/survey/editor/actions.ts` :1-120, :250-340
- `apps/web/modules/auth/lib/authOptions.ts` :168-310 (CredentialsProvider authorize)
- `apps/web/modules/core/rate-limit/rate-limit.ts` (full, 136 lines)
- `apps/web/modules/core/rate-limit/rate-limit-configs.ts` :1-40
- `apps/web/modules/ee/quotas/lib/evaluation-service.ts` :1-50
- `apps/web/modules/ee/quotas/lib/utils.ts` :100-180
- `apps/web/modules/survey/follow-ups/lib/follow-ups.ts` :1-70 (280 total)
- `apps/web/modules/storage/service.ts` :1-30
- `packages/cache/src/client.ts` :1-80; `packages/cache/src/cache-keys.ts` :1-60; `packages/cache/src/service.ts` (grep: `withCache` at :241)
- `packages/js-core/src/index.ts` :1-60; `packages/js-core/src/lib/common/command-queue.ts` :1-50
- `packages/surveys/src/lib/response-queue.ts` :1-80

## Claims verified

- Environment scoping on public response submission: `checkSurveyValidity` rejects `survey.environmentId !== environmentId` (`.../responses/lib/utils.ts:18-20`); same check in storage route (`.../storage/route.ts:76-80`).
- Single-use idempotency enforced at DB layer: `@@unique([surveyId, singleUseId])` (`schema.prisma:186`).
- Response + quota evaluation share one interactive Prisma transaction (`.../responses/lib/response.ts:24-41`).
- Pipeline auth = `x-api-key === CRON_SECRET` (`pipeline/route.ts:34`); `sendToPipeline` is invoked without `await` in the v2 responses route (`route.ts:245-259`) and swallows fetch errors (`pipelines.ts:22-24`).
- Webhook SSRF defense: URL validation + DNS-pinned dispatcher + `redirect: "manual"` + 5s AbortSignal timeout (`pipeline/route.ts:96-177`).
- Rate limiter is an atomic Redis Lua INCR+EXPIRE, fails open on Redis errors or when Redis absent (`rate-limit.ts:26-33,112-134`).
- Login `authorize` applies IP rate limiting, caps password at 128 chars, uses a control hash for constant-time user-enumeration defense, and consumes 2FA backup codes (`authOptions.ts:190-310`).
- Environment state is Redis-cached 60s with matching CDN cache headers, and the SDK is told to re-fetch after 1h (`environmentState.ts:68-70`, `environment/route.ts:60-75`).
- No Redis cache invalidation call was found in survey/environment write paths (searched `invalidateCache|\.del\(` across `apps/web/lib/survey/service.ts`, `apps/web/lib/environment/`, and `packages/cache/src/service.ts` greps) — the env-state cache appears TTL-only. Labeled "investigate" where taught.

## Uncertainties / not covered

- Quota counting (`quotas/lib/utils.ts:135-176`) is count-then-insert inside a transaction; whether Postgres default READ COMMITTED allows limit overshoot under concurrency is **plausible but not proven by test** — taught as "possible risk / investigate," and the survey-level `autoComplete` check in `pipeline/route.ts:273-301` has the same shape.
- Did not execute the app, DB migrations, or any test suite; all runtime behavior claims are from code reading.
- Not covered in depth: `packages/ai`, `apps/storybook`, Storybook/Chromatic CI, helm charts, `apps/web/modules/ee/sso` internals, `apps/web/vendor`, lingodotdev i18n pipeline internals, `docs/` site content, `charts/`.
- Pre-existing learning material exists in `docs/` (`architectural-cartographer/`, `mission-learning-path/`, `user-story-build-path/`, `junior-dev-onboarding-user-stories.md`). Untouched and not relied upon.
- Line numbers checked against the working tree on 2026-07-11; they will drift as the repo changes — search for symbols if an anchor misses.
