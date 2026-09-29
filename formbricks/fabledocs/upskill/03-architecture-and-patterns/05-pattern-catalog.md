# Pattern Catalog

Sixteen patterns this repo actually uses, each with real anchors. The goal is *recognition*: seeing the shape in any codebase and knowing its failure modes. Interview vocabulary is flagged per card.

---

## Pattern 1: Layered validation funnel

Problem it solves: rejecting bad requests as cheaply and as early as possible, with precise errors.
General shape: format → existence → tenancy → business rules → per-field rules, each layer only running if the previous passed.
Real example: `apps/web/app/api/v2/client/[environmentId]/responses/route.ts:204-239` (Zod parse → survey fetch → `checkSurveyValidity` → answer validation).
Second example: `apps/web/app/api/v1/client/[environmentId]/storage/route.ts:33-85`.
Why this implementation works: each layer returns a typed `Response` early; the happy path reads top-to-bottom with no nesting.
Failure modes: layers in the wrong order (expensive reCAPTCHA before cheap format checks); a layer silently skipped on one of several code paths.
Use it when: any public write endpoint. Avoid it when: a single Zod schema can express everything — don't hand-roll layers for pure shape checks.
Interview angle: "Where do you validate input?" — the answer is "at every trust boundary, ordered by cost."
Drill: reorder the checks in Flow 1 by execution cost and confirm the repo already did.

## Pattern 2: Database-enforced idempotency (unique constraint as invariant)

Problem it solves: "at most one response per single-use link" must hold even under concurrent duplicate submissions.
General shape: encode the invariant as a DB unique index; treat the violation error as a domain outcome, not a crash.
Real example: `packages/database/schema.prisma:186` (`@@unique([surveyId, singleUseId])`) + the 409 mapping in `apps/web/app/api/v2/client/[environmentId]/responses/route.ts:180-182`.
Second example: `Response.displayId @unique` (`schema.prisma:184`) — one response per display.
Why this implementation works: application-level `SELECT then INSERT` has a race window; the constraint is race-proof because the DB serializes it.
Failure modes: nullable columns in unique composites (Postgres allows many NULLs — here that's *intended*: non-single-use responses all have `singleUseId = NULL`); forgetting to catch the violation and returning 500.
Use it when: any "exactly one of X per Y" rule. Avoid it when: the rule is soft (limits, quotas) — see Pattern 12's race instead.
Interview angle: *idempotency* and *invariant* — say "I push uniqueness invariants into the database because application checks race."
Drill: find one more `@@unique` in the schema and state the business rule it encodes.

## Pattern 3: Fire-and-forget HTTP self-call as event bus

Problem it solves: decoupling response creation from slow side effects (webhooks, emails) without adding queue infrastructure.
General shape: after commit, POST the event to an internal route on yourself, authenticated by a shared secret; don't await; log failures.
Real example: `apps/web/app/lib/pipelines.ts:5-25` (producer) + `apps/web/app/api/(internal)/pipeline/route.ts:31-36` (consumer auth).
Second example: No second producer found — `sendToPipeline` is the single entry point (grep it).
Why this implementation works: zero new infrastructure; keeps p99 of the write path low; horizontal scale for free (any pod can consume).
Failure modes: **at-most-once** — process death or network blip loses the event silently; no backpressure; no replay; serverless platforms may kill the unawaited fetch.
Use it when: events are advisory (analytics). Avoid it when: consumers depend on delivery (customer webhooks arguably do) — then you need an outbox table or a queue.
Interview angle: *reliability*, *at-most-once vs at-least-once*. This pattern is the seed of the system-design conversation in [08-interview-prep/04](../08-interview-prep/04-system-design-from-this-repo.md).
Drill: write the 5-line pseudocode for converting this to a transactional outbox (insert event row in the Flow-1 transaction; poller delivers).

## Pattern 4: Read-through cache with TTL-only invalidation

Problem it solves: an endpoint hit by every widget page-load can't afford a DB round trip each time.
General shape: `withCache(fn, key, ttl)` — try Redis, miss → run fn → store; writes never touch the cache.
Real example: `apps/web/app/api/v1/client/[environmentId]/environment/lib/environmentState.ts:22-70` (60s TTL) + `packages/cache/src/service.ts:241` (`withCache`).
Second example: license status caching in `apps/web/modules/ee/license-check/` (see `createCacheKey.license.*`, `packages/cache/src/cache-keys.ts:31-37`).
Why this implementation works: bounded staleness with zero invalidation code to get wrong; Redis outage degrades to direct DB reads (fail-open).
Failure modes: users perceive "my change didn't apply" during the TTL; stacking more caches on top (CDN + client) multiplies worst-case staleness; side effects inside the cached fn run only on miss (`environmentState.ts:29-56` knows this and tolerates it).
Use it when: high-read, tolerance for seconds of staleness. Avoid it when: reads must reflect writes immediately — then add key deletion on write, and accept the complexity.
Interview angle: *consistency* vs *latency*; "how do you invalidate?" → "TTL-only is a legitimate answer when you can bound staleness."
Drill: compute worst-case staleness for a survey pause (CDN 60s + Redis 60s + SDK 1h) and say which layer you'd fix first.

## Pattern 5: Typed cache-key registry

Problem it solves: ad-hoc string keys collide, typo, and can't be audited.
General shape: one module exports `createCacheKey.resource.subresource(id)` returning a branded type; nothing else may build keys.
Real example: `packages/cache/src/cache-keys.ts:17-50` (`fb:env:{id}:state`, `fb:rate_limit:{ns}:{id}:{window}`), branded `CacheKey` type.
Second example: every call site — e.g. `environmentState.ts:68`, `rate-limit.ts:37`.
Why this implementation works: the branded type makes raw strings a compile error; the registry is a complete inventory of what's cached (audit in one file).
Failure modes: the registry growing "custom" escape hatches (`createCacheKey.custom`, :50+) that reintroduce ad-hoc keys.
Use it when: >2 cache keys exist. Avoid it when: never, really — this one generalizes.
Interview angle: shows *ownership* thinking — one module owns the key namespace.
Drill: add (on paper) a key for `survey:{id}:responseCount` following the house style.

## Pattern 6: Atomic rate limiting via Lua

Problem it solves: INCR-then-EXPIRE as two calls races across pods; windows could live forever.
General shape: single Lua script does INCR, sets EXPIRE only when count==1, returns `[count, allowed]` atomically.
Real example: `apps/web/modules/core/rate-limit/rate-limit.ts:46-66`; window key includes the window start (`cache-keys.ts:44-46`) so windows are fixed, not sliding.
Second example: config table in `rate-limit-configs.ts:1-36` — per-namespace budgets (login 10/15min, uploads 5/min).
Why this implementation works: Redis executes Lua atomically — no race regardless of pod count.
Failure modes: **fail-open** (`rate-limit.ts:112-134`): Redis down → unlimited requests; fixed windows allow 2× burst at window edges.
Use it when: multi-instance deployments. Avoid it when: single process — an in-memory counter is simpler.
Interview angle: classic "design a rate limiter" question with a production-grade answer in hand: fixed vs sliding window, atomicity, fail-open vs fail-closed per namespace.
Drill: modify (on paper) the Lua script to a sliding window with two keys. What did it cost you?

## Pattern 7: Server-action middleware chain (authn → input → authz → audit)

Problem it solves: every mutation needs the same safety rails; copy-pasting them guarantees one gets forgotten.
General shape: composable client: base error handling → session middleware → per-action `inputSchema` → explicit `checkAuthorizationUpdated` → `withAuditLogging` wrapper.
Real example: `apps/web/lib/utils/action-client/index.ts:14-65` + `apps/web/modules/survey/editor/actions.ts:250-267`.
Second example: any action file — grep `authenticatedActionClient` (100+ hits).
Why this implementation works: the unsafe parts (raw `actionClient`) are visibly different from the safe default; audit context rides the ctx object.
Failure modes: authz is still *per-action manual* (`checkAuthorizationUpdated` call) — a new action can forget it and be authenticated-but-unauthorized; OR-semantics of the access array read wrong.
Use it when: Next.js server actions exist at all. Avoid it when: n/a — but know tRPC/middleware equivalents.
Interview angle: *boundary* and *contract* — "mutations pass through a typed, audited pipeline."
Drill: write the fake diff of an action missing `checkAuthorizationUpdated` and the review comment that catches it (this is review kata #3).

## Pattern 8: Dual permission systems joined by OR

Problem it solves: org-wide roles (owner/manager) are too coarse for per-project collaboration; teams need scoped grants.
General shape: authorization passes if ANY grant in an access list passes: org-role grant OR project-team permission (weighted read < readWrite < manage).
Real example: `apps/web/lib/utils/action-client/action-client-middleware.ts:21-121` (`TAccess`, weights :39-48, loop :105-119).
Second example: `updateSurveyAction`'s access array (`editor/actions.ts:256-266`).
Why this implementation works: additive grants are easy to reason about ("who can? anyone matching any row").
Failure modes: additive systems can't express *deny*; forgetting that `billing` roles exist (check `TOrganizationRole`) when listing roles; weight tables silently diverging from the enum.
Use it when: B2B multi-tenant with teams. Avoid it when: you need deny rules or attribute-based conditions — then move to a policy engine.
Interview angle: *authorization* design — RBAC vs ReBAC; be able to sketch this repo's model in 60 seconds.
Drill: draw the grant-evaluation flowchart for `updateSurveyAction` for (a) an org owner, (b) a team contributor with `read`, (c) a billing-role user.

## Pattern 9: SSRF triple defense for user-supplied URLs

Problem it solves: customer-configured webhook URLs are attacker-controlled network destinations inside your VPC.
General shape: (1) validate/resolve the URL against private ranges, (2) pin the resolved IP into the HTTP dispatcher so DNS can't rebind between check and use, (3) refuse redirects, (4) bound with a timeout.
Real example: `apps/web/app/api/(internal)/pipeline/route.ts:96-177` (comments explain the TOCTOU explicitly) + `apps/web/lib/utils/validate-webhook-url.ts`.
Second example: escape hatch `DANGEROUSLY_ALLOW_WEBHOOK_INTERNAL_URLS` (`route.ts:101`) — the loud env-var naming is itself a pattern.
Why this implementation works: each layer closes the previous one's bypass: validation alone loses to rebinding; pinning alone loses to redirects.
Failure modes: forgetting one layer; validating but then fetching a *different* URL string; allowing `http://` to internal load balancers via 30x.
Use it when: any server-side fetch of user-supplied URLs (webhooks, importers, link previews). Avoid it when: never — always do this.
Interview angle: SSRF is a top security-round topic; this is a quotable production defense including the *named* TOCTOU concept.
Drill: explain aloud why `redirect: "manual"` is load-bearing even after IP pinning.

## Pattern 10: Result types instead of thrown errors at package boundaries

Problem it solves: shared packages can't know how callers want failures handled; exceptions lose type information.
General shape: `Result<T, E> = {ok:true,data} | {ok:false,error}` returned from package APIs; callers must branch.
Real example: `packages/cache/src/client.ts:11-58` (`Result<RedisClient, CacheError>` with error codes); `packages/storage` service returns `Result` consumed at `apps/web/app/api/v1/client/[environmentId]/storage/route.ts:95-106`.
Second example: `apps/web/modules/survey/follow-ups/lib/follow-ups.ts` (`Result`, `err`, typed `FollowUpSendError` codes).
Why this implementation works: the type system forces error handling at the call site; error *codes* (not messages) let callers map to HTTP statuses.
Failure modes: mixing paradigms (some paths throw `UniqueConstraintError` classes — see `.../responses/route.ts:176-182`); double-handling; `ok()`-wrapping everything including real bugs.
Use it when: package/API boundaries with multiple callers. Avoid it when: deep inside one module — local throws are fine.
Interview angle: TS-specific "how do you model errors?" — contrast exceptions, Result, and discriminated unions. [08-interview-prep/01](../08-interview-prep/01-js-ts-node-deep-dive.md) Q8.
Drill: refactor (on paper) `sendToPipeline` to return `Result<void, PipelineError>` and see what the callers would have to do.

## Pattern 11: Zod schema as the single contract (`ZSurvey` everywhere)

Problem it solves: survey structure lives in Json columns — the DB can't type it, so something else must.
General shape: one Zod schema per domain object in `@formbricks/types`, used for API input parsing, server-action input, and (via Prisma json-types generator) DB payload typing.
Real example: `apps/web/modules/survey/editor/actions.ts:250` (`.inputSchema(ZSurvey)`); `packages/database/schema.prisma:357-370` json-doc comments (`/// [SurveyBlocks]`) wiring generated types; `packages/database/zod/` directory.
Second example: `ZResponseInputV2` (`apps/web/app/api/v2/client/[environmentId]/responses/types/response.ts`).
Why this implementation works: schema-first means client, server, and DB agree by construction; parse-don't-validate at every boundary.
Failure modes: schema drift vs stored Json (old rows predate new schema rules — migrations must backfill or schemas must stay backward-compatible); giant schemas get slow to parse on hot paths.
Use it when: semi-structured domain data in Json columns. Avoid it when: the data is relational — don't Json what you'll need to query/aggregate.
Interview angle: *contract* — "where is your source of truth for data shape?" Also the classic "Json column vs normalized tables" tradeoff, which this repo answers with "Json + Zod + app-level validation."
Drill: find where `blocks` (Json in DB) gets validated on write and typed on read; write the chain as arrows.

## Pattern 12: Transactional check-then-act (quota counting) — the anti-pattern to recognize

Problem it solves (intends to): enforce response quotas ("max N screened-in responses").
General shape: inside a transaction: count matching rows → compare to limit → insert/update accordingly.
Real example: `apps/web/modules/ee/quotas/lib/utils.ts:135-176` (groupBy count :135-151, compare :155-164, act :172-177), called from the Flow-1 transaction (`.../responses/lib/response.ts:24-41`).
Second example: `autoComplete` in `apps/web/app/api/(internal)/pipeline/route.ts:273-301` (count → maybe close survey).
Why this implementation is risky: under Postgres READ COMMITTED, two concurrent transactions can both count `limit-1` and both proceed — quota overshoot. **Possible risk / investigate**: I did not run a concurrency test; an advisory lock or serializable isolation may exist elsewhere (none found).
Use the honest frame: a transaction gives you *atomicity* of your own writes, not *isolation* from concurrent counts. These are different words for a reason.
Interview angle: this distinction (atomicity ≠ isolation) is a favorite senior filter question. Fixes to name: `SELECT ... FOR UPDATE` on a parent row, advisory lock keyed by quotaId, a counter column with `UPDATE ... WHERE count < limit` returning affected rows, or serializable + retry.
Drill: write the two-transaction interleaving that overshoots a quota of 100 by one. Then pick a fix and state its lock-contention cost.

## Pattern 13: Client-side durable queue (offline responses)

Problem it solves: survey answers on flaky mobile networks must not be lost.
General shape: enqueue updates in memory + IndexedDB; a per-survey module-level lock ensures one sender; retry with delay; remove from storage only after server ack.
Real example: `packages/surveys/src/lib/response-queue.ts:36-53` (module-level locks with the *why* in comments), `:55-80` (queue + `pendingDbIds` mapping), `offline-storage.ts` (IndexedDB CRUD).
Second example: `packages/js-core/src/lib/common/command-queue.ts:25-50` — same family: serialize SDK commands so `setup` completes before `track`.
Why this implementation works: survives React remounts (locks are module-level, not instance-level — the comment at :36-40 explains a real bug class); at-least-once from the client side pairs with Pattern 2's server-side dedupe.
Failure modes: at-least-once without a server idempotency key duplicates responses (note: only single-use surveys get the unique constraint — worth investigating how open surveys dedupe retries); IndexedDB unavailable in private browsing.
Use it when: user data + unreliable networks. Avoid it when: data is trivially refetchable.
Interview angle: "How do you handle offline?" — answer with queue + persistence + locking + idempotent server, and name which piece lives where in this repo.
Drill: trace one response through queue → IndexedDB → send fail → retry → ack → cleanup, citing functions.

## Pattern 14: Feature gating by license (EE boundary)

Problem it solves: one codebase ships open-source core + paid enterprise features.
General shape: EE code physically isolated under `modules/ee/` (with its own LICENSE); runtime checks like `getIsContactsEnabled(organizationId)` guard entry points; limits (not just features) also gate.
Real example: `apps/web/app/api/v2/client/[environmentId]/responses/route.ts:82-96` (contacts gate); `apps/web/app/api/v1/client/[environmentId]/storage/route.ts:82-85` (upload-size gate); `apps/web/modules/ee/license-check/`.
Second example: follow-ups permission check `getSurveyFollowUpsPermission` (`modules/survey/follow-ups/lib/utils.ts`).
Why this implementation works: directory boundary makes license scope auditable; org-scoped checks cache well.
Failure modes: gate checked in UI but not API (the real test is always the API); EE checks adding latency to hot public paths (each is a cache/DB hop — see Flow 1 step 3).
Use it when: open-core products. Avoid it when: a build-time split (separate packages) is feasible and cleaner.
Interview angle: *boundary* + business awareness — knowing why the code is shaped by the business model reads as senior.
Drill: pick any EE feature and find its (a) directory, (b) runtime gate, (c) what happens on the free tier.

## Pattern 15: Route wrapper for cross-cutting API concerns

Problem it solves: logging, error reporting, and rate limiting repeated in every route handler.
General shape: `withV1ApiWrapper({handler, customRateLimitConfig})` — handler returns `{response, error?}`; wrapper logs, reports, rate-limits.
Real example: `apps/web/app/api/v1/client/[environmentId]/storage/route.ts:29-113` (note `customRateLimitConfig` at :112); wrapper in `apps/web/app/lib/api/with-api-logging.ts`.
Second example: v2's equivalent `apps/web/modules/api/v2/auth/api-wrapper.ts`.
Why this implementation works: the `{response, error}` return contract lets the wrapper distinguish expected 4xx (no alert) from 5xx (report) — see `storage/route.ts:98-105` choosing which shape to return.
Failure modes: wrapper drift between API versions (v1/v2/v3 each have one); handlers bypassing the wrapper entirely.
Use it when: >3 routes share concerns. Avoid it when: a framework middleware already covers it.
Interview angle: *observability* — "how do you make sure every endpoint is logged and rate-limited?" Answer: make the safe path the easy path.
Drill: list what the wrapper does that a naive try/catch wouldn't (error classification, structured context, rate limit, Sentry).

## Pattern 16: Pre-compiled widget bundle served from the app

Problem it solves: the survey renderer must run on arbitrary customer sites — it can't be a Next.js page.
General shape: `packages/surveys` builds (Vite) to UMD+ESM; the bundle is copied into `apps/web/public/js/`; the app and SDK load it from there; `AGENTS.md` documents the cache-busting workflow.
Real example: `AGENTS.md` ("Survey Packages Build & Cache" section); `packages/surveys/vite.config.*`; `packages/js-core/src/lib/survey/widget.ts:301` (`preloadSurveysScript`).
Second example: No second bundle — but `apps/web/vendor/` holds other vendored client code.
Why this implementation works: one artifact serves the link-survey page, the in-app widget, and the SDK; versioned with the app deployment.
Failure modes: the triple cache trap (turbo build cache + public/ copy + browser cache) — AGENTS.md's `--force` + hard-refresh instructions exist because people lost hours here; source edits silently not taking effect.
Use it when: shipping embeddable JS. Avoid it when: internal-only components — just import them.
Interview angle: *build boundary* — knowing that "the app imports from `dist/`, not source" explains a whole class of "my change does nothing" bugs; a great debugging-round anecdote.
Drill: from `AGENTS.md`, write the exact command sequence to see a `packages/surveys` change live, and *why each step exists*.
