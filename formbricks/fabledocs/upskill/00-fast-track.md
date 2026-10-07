# Fast Track — One Weekend in Formbricks

Goal: by Sunday night you can run the app, trace two end-to-end flows aloud, and have made one safe change with a passing test. This is the minimum viable "I know this codebase" state.

## 1. Install and run (Saturday morning)

All commands are from the repo root. They come from `package.json:12-47` and `AGENTS.md` — marked __inferred__ (read from the repo's own scripts and docs, not executed during authoring; this machine had no running Docker daemon).

```bash
pnpm install              # inferred — installs the whole workspace
pnpm db:up                # inferred — docker compose: Postgres + Redis (+ MailHog etc. per docker-compose.dev.yml)
pnpm db:migrate:dev       # inferred — applies Prisma migrations (packages/database/migration/)
pnpm go                   # inferred — db:up + all dev servers via turbo (or: pnpm dev)
```

Open http://localhost:3000, create an account (locally, email verification is relaxed — check `apps/web/lib/constants.ts` for the env flags), create an organization, project, and your first survey.

If something fails: check Docker is running, then check `.env` — the setup script `scripts/setup-dev-env.sh` (wired as `pnpm dev:setup`) generates one.

Run one unit suite to prove the toolchain works:

```bash
pnpm test --filter=@formbricks/cache    # inferred — vitest for the Redis cache package
```

## 2. The first 10 files to open, in order

| # | File | Why |
| --- | --- | --- |
| 1 | `AGENTS.md` | The maintainers' own map: structure, commands, conventions. Read it fully. |
| 2 | `packages/database/schema.prisma:344-421` | The `Survey` model — the product's center of gravity. Note `blocks`, `endings`, `singleUse`, `autoComplete` are Json columns. |
| 3 | `packages/database/schema.prisma:158-190` | The `Response` model. Find `@@unique([surveyId, singleUseId])` — a DB-enforced invariant you'll meet again. |
| 4 | `apps/web/app/api/v2/client/[environmentId]/responses/route.ts` | The public response-submission endpoint. The whole request lifecycle in one file. |
| 5 | `apps/web/app/lib/pipelines.ts` | 25 lines that define the async architecture: an HTTP self-call with a shared secret. |
| 6 | `apps/web/app/api/(internal)/pipeline/route.ts` | Where webhooks, integrations, notification emails, and follow-ups fan out. |
| 7 | `apps/web/lib/utils/action-client/index.ts` | How every server action gets auth: `actionClient` → `authenticatedActionClient`. |
| 8 | `apps/web/lib/utils/action-client/action-client-middleware.ts:94-121` | `checkAuthorizationUpdated` — the authorization model (org roles vs project-team permissions). |
| 9 | `apps/web/app/api/v1/client/[environmentId]/environment/lib/environmentState.ts` | The Redis `withCache` pattern and the SDK's config payload. |
| 10 | `packages/js-core/src/index.ts` | The public SDK surface your customers embed — `setup`, `setUserId`, `track` all funnel into a `CommandQueue`. |

## 3. Trace two flows (Saturday afternoon)

Do these with the code open. Full trace tables live in [01-codebase-cartography/05-key-flows.md](01-codebase-cartography/05-key-flows.md).

**Flow A — a link-survey response is submitted:**
browser widget (`packages/surveys/src/lib/response-queue.ts`) → `POST /api/v2/client/{environmentId}/responses` (`apps/web/app/api/v2/client/[environmentId]/responses/route.ts:204-278`) → validation (`.../responses/lib/utils.ts:13-73`: environment scoping, status, single-use, reCAPTCHA) → `createResponseWithQuotaEvaluation` in one Prisma transaction (`.../responses/lib/response.ts:21-45`) → `sendToPipeline` fire-and-forget (`route.ts:245-259`).

Pause-and-predict before you look: what stops a response being submitted to a survey in a *different* environment? (Answer: `checkSurveyValidity` line 18.) What happens if the same single-use link is used twice? (Answer: DB unique constraint → `UniqueConstraintError` → 409.)

**Flow B — what happens after the response arrives:**
`POST /api/pipeline` authenticated by `CRON_SECRET` header (`apps/web/app/api/(internal)/pipeline/route.ts:31-36`) → find matching webhooks (`:82-93`) → deliver with SSRF-validated, DNS-pinned fetch and 5s timeout (`:96-177`) → on `responseFinished`: integrations, notification emails, follow-ups, autoComplete check (`:179-318`).

Pause-and-predict: if a customer's webhook endpoint is down, does the response still get saved? Does the webhook ever retry? (Saved: yes — it was committed before the pipeline call. Retry: no — failures are logged and dropped. That asymmetry is a senior-level observation; remember it.)

## 4. One small safe change (Sunday)

Pick one:

- Add a unit test for `getCountry` in `apps/web/app/api/v2/client/[environmentId]/responses/route.ts:37-41` (header precedence: CF → Vercel → CloudFront). Colocate as `*.test.ts` like the repo does.
- Improve one log message in `apps/web/app/lib/pipelines.ts:22-24` to include `surveyId` and `event` context (match the structured-logging style in `apps/web/app/api/(internal)/pipeline/route.ts:174-176`).

Run the neighboring tests before and after (`pnpm test --filter=@formbricks/web -- <path>` — __inferred__; check `apps/web/package.json` scripts for the exact test invocation). Don't commit to main; branch.

## 5. Teach-back (Sunday night)

Explain Flow A aloud in under three minutes as if an interviewer asked "walk me through how a form submission works in a codebase you know." Must include: where validation happens, where authorization/tenancy is enforced, what's inside the DB transaction, and what's deliberately *outside* it (the pipeline). Record yourself. If you said "and then it just saves it," redo it.

## What the fast track skips

Auth internals (NextAuth + SSO + 2FA), the survey editor's state management, contacts/segments, quotas' concurrency story, billing/EE licensing, i18n tooling, storage signed-URL flow, and all of the drills. That's what the other eight modules are for.
