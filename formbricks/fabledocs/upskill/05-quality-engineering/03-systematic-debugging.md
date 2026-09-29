# Systematic Debugging

The method: **reproduce → narrow → hypothesize → test cheaply → fix the root cause → add regression coverage.** Never skip step 1; never "fix" at step 3. Below, the method applied to five realistic scenarios in this codebase, with the actual tools named.

Tools of this repo:
- Structured logs via `@formbricks/logger` (pino-style; grep for `logger.error` context objects).
- Sentry (`sentry.server.config.ts`, breadcrumbs in `rate-limit.ts:101-108`) for unexpected errors.
- Prisma query logging: enable via Prisma client log option / `DEBUG` env when reproducing DB questions locally.
- Vitest colocated tests (`*.test.ts` next to source) for cheap hypothesis tests — often faster than clicking through the UI.
- Playwright specs + traces for UI flows (`apps/web/playwright/`).
- Browser DevTools: network tab for client-API calls; Application → IndexedDB (`offline-storage.ts`) for queued responses.
- MailHog or equivalent from `docker-compose.dev.yml` for asserting emails locally (verify the service name in that file).

---

## Scenario 1: "Customer says their webhook never fired for some responses"

Reproduction: create a webhook to a request-bin, submit a response, confirm the normal case works. Then reproduce *absence*: you can't directly — so reproduce the conditions (restart the app mid-submission; make the webhook endpoint hang >5s).
First question: did the event reach the pipeline route at all, or die between response-commit and fan-out?
Narrowing path:
1. Grep logs for `Error sending event to pipeline` (`apps/web/app/lib/pipelines.ts:23`) around the response's `createdAt`. If present → the self-call failed (WEBAPP_URL wrong? pod died?).
2. If absent, grep for `Webhook call to ... failed` (`pipeline/route.ts:175`). If present → delivery failed: timeout (5s, :105-115), SSRF validation rejection, or target 4xx/5xx.
3. If neither log exists → the process likely died before/at the unawaited `sendToPipeline` (`responses/route.ts:245`) — check deploy/restart timestamps against the gap.
Useful probes: request-bin timestamps; `AbortSignal.timeout` produces a distinguishable error name (`TimeoutError`) in the logs.
Likely root causes: at-most-once pipeline (design), webhook target slow, SSRF validation rejecting a newly-internal DNS record.
Regression test to add: none can force delivery — that's the lesson. Add an alert on log pattern instead, and cite this incident in the outbox proposal (senior project 2).
Senior lesson: some bugs are architecture. The fix isn't a patch, it's a design conversation with evidence.
Interview version: narrate exactly this — "I proved the event was lost between commit and fan-out by log absence, then explained why no code fix exists without changing delivery semantics."

## Scenario 2: "I edited a survey question but the widget still shows the old one"

Reproduction: edit + publish a survey, reload the embedding page within a minute.
First question: which cache layer is serving stale — CDN, Redis, or the SDK's in-browser state?
Narrowing path:
1. `curl -H 'Accept: application/json' <app>/api/v1/client/<envId>/environment` directly (bypasses SDK): stale? → server-side (Redis 60s TTL, `environmentState.ts:69`) or CDN (`s-maxage=60`, `environment/route.ts:66-74`). Check the `age`/`cf-cache-status` response headers to split those two.
2. Fresh from curl but stale in browser → SDK holds state until `expiresAt` (+1h, `environment/route.ts:63`) — check IndexedDB/localStorage config in DevTools.
3. Also confirm the edit actually persisted: DB row `updatedAt` via Prisma studio (`pnpm --filter @formbricks/database db:studio` if wired — verify script name first).
Likely root causes: working-as-designed TTL staleness; or (if developing the widget) the `packages/surveys` build cache — the bundle in `apps/web/public/js/` is stale (AGENTS.md's `--force` rebuild + hard refresh).
Regression coverage: a test asserting TTL values match documented numbers, so a future TTL change is a *decision*, not an accident.
Senior lesson: "stale" is not one bug — enumerate the layers, bisect with curl before touching code.
Interview version: perfect "debug a caching issue" narrative — three layers, header-based bisection.

## Scenario 3: "Duplicate responses appearing for a single-use survey"

Reproduction: submit the same single-use link twice fast (two tabs, submit near-simultaneously).
First question: is dedupe failing at the DB constraint, or are these actually *different* singleUseIds?
Narrowing path:
1. Query the two Response rows: identical `singleUseId`? Two rows with the same non-null `(surveyId, singleUseId)` is impossible (`schema.prisma:186`) — if you see it, the ids differ; diff them.
2. Different ids → where did the second id come from? `validateSingleUseResponseInput` extracts suId from the *URL the client reports* (`single-use.ts:43-80`) — a client can mint URLs. Check whether `isEncrypted` is on (`schema.prisma:395`): unencrypted mode trusts the suId format.
3. Same id, one row, but user *saw* two success screens → client retry got a 409 `UniqueConstraintError` (`responses/route.ts:180-182`) that the widget rendered as success? Read `response-queue.ts` error handling for the 409 path.
Useful probes: Prisma studio on Response; widget network tab for the second POST's status code.
Likely root causes: unencrypted single-use mode (documented tradeoff), or client retry UX, not a server race — the constraint has that covered.
Regression test: vitest on the 409 mapping; e2e double-submit asserting one row.
Senior lesson: start from the invariant ("the DB makes true duplicates impossible") and let it eliminate half the hypothesis space.
Interview version: "I used a database constraint as a proof to bisect the bug space" — memorable line.

## Scenario 4: "Login is suddenly slow (3–4s) for everyone"

Reproduction: time `authorize` locally — where do the seconds go?
First question: CPU (bcrypt) or I/O (DB/Redis)?
Narrowing path:
1. Wrap timings around the three costly steps in `authOptions.ts:190-235`: `applyIPRateLimit` (Redis), `prisma.user.findUnique` (DB), `verifyPassword` (bcrypt CPU).
2. Rate limiter slow → is Redis reachable? `connectTimeout: 3000` in `packages/cache/src/client.ts:24-26` means a *down-but-routable* Redis can add seconds before fail-open kicks in. Check `Redis client error` logs.
3. bcrypt slow → did the hash cost factor change? Constant across users though — wouldn't be "suddenly."
4. DB slow → check pool exhaustion (long interactive transactions from Flow 1 under load competing for connections).
Likely root cause (most instructive): Redis half-down adding connect-timeout latency to every fail-open call — fail-open protects *availability*, not *latency*.
Regression coverage: a timeout budget test or a metric/alert on rate-limit check duration.
Senior lesson: fail-open designs need latency bounds, not just error handling.
Interview version: great systems-thinking story — "the fallback path was correct but slow, and slowness is an outage."

## Scenario 5: "A follow-up email went to the wrong address"

Reproduction: build a survey with a follow-up whose `to` is a question id; answer with an email; confirm routing.
First question: was `to` a direct address or a response-data lookup?
Narrowing path:
1. Read the follow-up's config (Prisma studio, `SurveyFollowUp.action.properties`).
2. Direct address → check `evaluateFollowUp`'s first branch (`follow-ups.ts:33-53`): the zod `z.email()` gate. Wrong address here means wrong *config* — user error or editor bug; go check the editor's save validation.
3. Question-id path → `response.data[to]` (:30) — if the question id changed after the follow-up was configured (question deleted/re-created in the editor), `to` points at a stale id. What does the code do? (:56-62 — returns an error result; but check the *editor* prevents dangling ids — that's the real bug surface.)
Useful probes: MailHog locally; the `FollowUpSendError` codes and the rate limiter (`follow-ups.ts` imports `applyRateLimit` — 50/hr, `rate-limit-configs.ts:23`).
Likely root causes: referential integrity between editor-time config and response-time data — Json columns have no FK (Pattern 11's cost, live).
Regression test: editor-level validation test that removing a question referenced by a follow-up warns/blocks.
Senior lesson: when config references data by id across Json documents, *something* must own referential integrity — find out what, and if nothing does, that's the root cause.
Interview version: "dangling reference across two Json documents" — a data-modeling debugging story interviewers rarely hear and remember.

---

## The habit to build

Before touching code, write down: (1) the exact observable symptom, (2) the two most-likely layers, (3) the cheapest experiment that splits them. In interviews, *say* this structure aloud — the interviewer is grading the search strategy, not the bug. Timed versions of scenarios 1–3 live in [08-interview-prep/05-debugging-and-code-review-rounds.md](../08-interview-prep/05-debugging-and-code-review-rounds.md).
