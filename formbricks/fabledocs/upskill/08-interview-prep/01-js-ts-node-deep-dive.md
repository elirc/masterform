# JS / TS / Node Deep-Dive Questions

Fifteen question cards. Practice: answer aloud in ≤90 seconds, *leading with the repo example*. Round: JS/TS deep-dive unless noted.

## Q1: What actually happens when you `await` inside a request handler — and when would you deliberately NOT await something?

What it's really testing: event loop + microtasks + the consequences of dangling promises.
Repo anchor: `apps/web/app/api/v2/client/[environmentId]/responses/route.ts:245-259` — `sendToPipeline(...)` called without `await` after the DB transaction.
Junior answer sounds like: "await pauses until the promise resolves."
Mid-level answer adds: awaiting suspends *this* handler (the thread keeps serving others); here the code skips `await` so the client gets its 200 before webhook fan-out — trading latency for delivery certainty.
Senior answer includes: the failure mode — in serverless, the runtime may freeze after the response, so unawaited work can silently die; unhandled rejections are only survivable because `sendToPipeline` ends in `.catch` (`app/lib/pipelines.ts:22-24`); the principled fix is an outbox, not `void promise`.
Likely follow-ups: "What's the difference between microtask and macrotask here?" "What does Node do with an unhandled rejection?"
Practice drill: trace what the pipeline call's failure looks like in logs vs what the user sees.

## Q2: `Promise.all` vs `Promise.allSettled` — when does the choice matter?

What it's really testing: async error semantics.
Repo anchor: `apps/web/app/api/(internal)/pipeline/route.ts:303-318` — webhook + email promises gathered with `allSettled`, rejected ones logged individually.
Junior: "all fails fast, allSettled doesn't."
Mid: here one dead customer webhook must not cancel the other nine webhooks or the notification emails — independent side effects ⇒ `allSettled`; dependent reads (`route.ts:181-184` uses `Promise.all` for integrations+count) ⇒ fail-fast is correct.
Senior: adds resource framing — each webhook already has its own 5s `AbortSignal.timeout` (:105-115), so the batch is bounded at max(individual timeouts), and notes `all` on the read pair is fine *because* a missing count makes the whole branch pointless.
Follow-ups: "How would you cap concurrency at 10 webhooks at a time?" (p-limit / manual worker pool.)
Drill: rewrite :304 with `Promise.all` and narrate the exact behavior change.

## Q3: How do you write a race-condition-free counter across multiple server instances?

What it's really testing: shared-state reasoning beyond one process.
Repo anchor: `apps/web/modules/core/rate-limit/rate-limit.ts:44-66` — Lua script doing INCR + conditional EXPIRE atomically in Redis.
Junior: "use a variable / a mutex."
Mid: in-process locks don't survive horizontal scaling; push the atomicity into the shared store — Redis INCR is atomic, and the Lua script makes INCR+EXPIRE one atomic unit.
Senior: names the residual issues — fixed windows allow edge bursts, fail-open on Redis loss (:129-134) is a deliberate availability choice, and contrasts with the DB-side version of the same problem (quota counting, `modules/ee/quotas/lib/utils.ts:135-176`) which is *not* race-free.
Follow-ups: "Sliding window?" "What if Redis is a cluster?"
Drill: explain why `INCR` then `EXPIRE` as two await-ed calls is buggy (crash between them ⇒ immortal key).

## Q4: What is a TS discriminated union and where does it beat exceptions?

What it's really testing: type-driven error design.
Repo anchor: `packages/cache/src/client.ts:11-58` returns `Result<RedisClient, CacheError>`; consumed with `.ok` narrowing. Also `TValidatedResponseInputResult` union in `apps/web/app/api/v2/client/[environmentId]/responses/route.ts:30-35` — either `{environmentId, responseInputData}` or `{response}` — narrowed by `"response" in validatedInput` (:208).
Junior: defines the syntax.
Mid: the compiler *forces* callers to handle failure; the `in`-operator narrowing at :208 is control-flow analysis doing exhaustiveness work exceptions can't.
Senior: discusses boundary placement — Result at package boundaries, throws for truly exceptional states (the same route still catches `UniqueConstraintError` class at :180) — and the cost of mixing both idioms in one codebase.
Follow-ups: "`unknown` vs `any` in the catch clause?" "How would `satisfies` help the Result helpers?"
Drill: write the narrowing for a 3-armed union without casts.

## Q5: Walk me through Zod's `safeParse` vs `parse` and where each belongs.

What it's really testing: parse-don't-validate; trust boundaries.
Repo anchor: `apps/web/app/api/v2/client/[environmentId]/responses/route.ts:50-60` (`ZEnvironmentId.safeParse` → 400 with details) vs `.inputSchema(ZSurvey)` on server actions (`modules/survey/editor/actions.ts:250`) where the framework handles the failure.
Junior: "safeParse doesn't throw."
Mid: at HTTP boundaries you own the error response, so `safeParse` + typed error details (`transformErrorToDetails`); inside next-safe-action the middleware owns it, so schema-declaration style.
Senior: notes the *output type* is the point — after parse, `TResponseInputV2` flows through the whole file with no re-checking; validation happens exactly once per boundary; and flags the performance cost of mega-schemas (ZSurvey) on hot paths.
Follow-ups: "How do you validate data leaving your system (webhook payloads)?"
Drill: find where the parsed type is *widened* back to any/unknown anywhere in the flow (you shouldn't find one — say why that matters).

## Q6: How does structuredClone/serialization bite you when data crosses process boundaries?

What it's really testing: JSON round-trip hazards (Dates, undefined, Maps).
Repo anchor: `apps/web/app/api/(internal)/pipeline/route.ts:39-42` — `convertDatesInObject(jsonInput, new Set([...]))` re-hydrating Dates after the HTTP self-call, because `sendToPipeline` JSON-stringified them (`app/lib/pipelines.ts:16-21`).
Junior: "JSON doesn't have dates."
Mid: the pipeline is an HTTP boundary inside one app — every Date became a string; the consumer must re-hydrate *selectively* (the Set names Json-ish fields to skip) or Zod validation would reject/mistype.
Senior: generalizes — any self-call/queue/cache serializes; the schema should own hydration (z.coerce.date()); and this is hidden coupling: producer and consumer must agree forever, which is what makes "just POST to yourself" less free than it looks.
Follow-ups: "What else does JSON.stringify drop?" (undefined, functions, BigInt throws, Map/Set become {}.)
Drill: list the fields in `TPipelineInput` that survive the round trip unchanged vs transformed.

## Q7: Explain module-level state in a bundled client library — when is it a feature?

What it's really testing: module systems, singleton lifecycle, closures.
Repo anchor: `packages/surveys/src/lib/response-queue.ts:36-53` — `syncingBySurvey` Maps at module scope, with a comment explaining they must survive React `useMemo` recreation of the queue instance; `packages/js-core/src/lib/common/command-queue.ts:31-35` — classic `getInstance()` singleton.
Junior: "modules run once, so variables persist."
Mid: instance state dies when React recreates the instance; module state persists per bundle load — here that's *the fix* for double-send bugs. Testability cost: `_syncLocks.clear()` (:42-52) exists purely for test isolation.
Senior: adds bundle-duplication risk (two copies of the module = two lock maps — UMD + ESM both ship, `AGENTS.md`), SSR/edge concerns (module state shared across requests on the server — fine here because this is browser-only), and when to prefer explicit DI.
Follow-ups: "Why is a Map keyed by surveyId, not a boolean?"
Drill: describe the bug that would come back if these Maps became instance properties.

## Q8: Result types vs exceptions in TypeScript — argue both sides.

What it's really testing: error-handling philosophy with type-system awareness.
Repo anchor: `packages/storage` Result consumed at `apps/web/app/api/v1/client/[environmentId]/storage/route.ts:95-106`; exceptions with typed classes at `.../responses/route.ts:175-191` (`InvalidInputError` → 400, `UniqueConstraintError` → 409).
Junior: picks one dogmatically.
Mid: Results make failure part of the signature (compiler-checked); exceptions keep happy paths clean and carry stacks; this repo uses Results at package boundaries and class-based throws inside services, mapped to HTTP at the route.
Senior: the mapping table *is* the API contract (error → status code); consistency beats purity — two idioms in one flow is a real maintenance tax worth raising in review; mentions `neverthrow`-style chaining as the next step if Results proliferate.
Follow-ups: "How do you keep error codes stable across API versions?"
Drill: write the route-level catch that maps 3 domain errors to statuses without `instanceof` leaking into the service layer.

## Q9: What does `"use server"` / `import "server-only"` actually protect?

What it's really testing: Next.js runtime boundaries, secret hygiene. (Round: frontend/fullstack.)
Repo anchor: `apps/web/modules/survey/editor/actions.ts:1` (`"use server"`); `apps/web/app/api/v1/client/[environmentId]/environment/lib/environmentState.ts:1` (`import "server-only"`).
Junior: "it makes it run on the server."
Mid: `"use server"` marks RPC entry points callable from the client — each exported function is a public, network-reachable endpoint and must authorize like one; `server-only` is the inverse guard: build-time error if a secrets-touching module leaks into a client bundle.
Senior: therefore server actions are *attack surface*: input schemas + `authenticatedActionClient` (`lib/utils/action-client/index.ts:51-65`) exist because "it's just a function call" is an illusion; also notes actions are POSTs — no CSRF token needed due to same-origin enforcement by Next, but worth verifying per deployment.
Follow-ups: "How would you find an unauthorized action in this repo?" (grep actions using raw `actionClient` and audit each.)
Drill: do that grep; classify 3 hits as safe/unsafe with reasons.

## Q10: How do you keep a hot endpoint from hammering the database?

What it's really testing: caching layers, TTL math, Node/HTTP specifics.
Repo anchor: Flow 3 — `environmentState.ts:22-70` (Redis `withCache`, 60s) + `environment/route.ts:60-75` (CDN `s-maxage=60, stale-while-revalidate=60`) + SDK `expiresAt` 1h.
Junior: "add caching."
Mid: names all three layers, their TTLs, and that invalidation is TTL-only — bounded staleness as an explicit choice.
Senior: cache stampede (many pods miss simultaneously at TTL expiry — is there locking? check `withCache`, `packages/cache/src/service.ts:241`; if not, that's a thundering-herd risk to raise), fail-open on Redis loss, and the write-path asymmetry: submission checks hit the DB (`getSurvey` in Flow 1), so stale caches can't create *incorrect* writes, only stale *reads*.
Follow-ups: "How would you invalidate on survey publish?" (delete `env:{id}:state` key in `updateSurvey` — and note why CDN can't be purged as easily.)
Drill: compute requests/sec the DB sees for 10k widget loads/min at each cache-hit ratio.

## Q11: Explain closure capture bugs in async loops.

What it's really testing: closures + async interleaving (conceptual — no direct repo bug).
Repo anchor: healthy example — `pipeline/route.ts:119-177` maps over `webhooks` creating one promise per webhook; each closure correctly captures its own `webhook` via the map callback parameter.
Junior: recites the `var` loop bug.
Mid: with `let`/callback params each iteration gets its own binding; the modern version of the bug is *shared mutable objects* captured across awaits, not loop variables.
Senior: points at `requestHeaders` (:140-144) being rebuilt per webhook — if it were hoisted above the map for "efficiency," a signature from one webhook could leak onto another. Naming that hypothetical shows you review async code for capture scope.
Follow-ups: "What about capturing `res`/`req` in a setTimeout?"
Drill: write the broken hoisted-headers version and the one-line reason it's wrong.

## Q12: What is an interactive Prisma transaction and what are its sharp edges?

What it's really testing: Node + DB connection behavior. (Round: API/data overlap.)
Repo anchor: `apps/web/app/api/v2/client/[environmentId]/responses/lib/response.ts:24-41` — `prisma.$transaction(async (tx) => …)` passing `tx` down into quota evaluation (`modules/ee/quotas/lib/evaluation-service.ts:32-50` accepts `tx?: Prisma.TransactionClient`).
Junior: "it makes queries atomic."
Mid: the callback holds a dedicated connection until it returns — long work inside (network calls!) starves the pool; note how the repo threads `tx` explicitly so helpers join the same transaction instead of silently using the global client.
Senior: the `tx ?? prisma` fallback pattern (:43 in evaluation-service) is the subtle risk — a helper that *forgets* to pass tx writes outside the transaction and breaks atomicity with no error; also names isolation level (default READ COMMITTED) and the quota-count race that follows (Pattern 12).
Follow-ups: "What belongs inside vs outside the transaction?" (Answer with Flow 1: quota links inside; pipeline/webhooks outside, post-commit.)
Drill: find one helper in the quota path and verify every query in it uses `tx`.

## Q13: How do you defend a login endpoint, at the code level?

What it's really testing: Node crypto habits, timing attacks. (Round: security overlap.)
Repo anchor: `apps/web/modules/auth/lib/authOptions.ts:190-260` — IP rate limit (:191), 128-char cap (:205), control-hash constant-time verify (:232-235), uniform errors (:238-260).
Junior: "hash passwords, rate limit."
Mid: explains the control hash — verify against a dummy hash when the user doesn't exist so response time doesn't reveal account existence; uniform error strings for the same reason.
Senior: adds the DoS angle (bcrypt cost × long passwords = CPU burn, hence the cap), notes the limiter fails open (`rate-limit.ts:129-134`) so Redis loss un-throttles brute force — and proposes fail-closed for `auth:*` namespaces specifically.
Follow-ups: "Why bcrypt/argon2 over sha256?" "Where do backup codes get consumed?" (:298-306 — used code nulled and re-encrypted.)
Drill: say the control-hash explanation in 30 seconds flat; it's a high-signal answer.

## Q14: Node streams vs buffering — when does a file upload melt your server?

What it's really testing: memory model, backpressure awareness.
Repo anchor: the design *dodge* — `apps/web/app/api/v1/client/[environmentId]/storage/route.ts:29-113` never touches file bytes; it mints S3 presigned URLs (`modules/storage/service.ts:16-30`) so the browser uploads directly.
Junior: "use multer."
Mid: proxying uploads buffers or streams through Node — CPU + memory + connection-time cost; presigned URLs move bytes to S3 and leave the app handling only JSON control messages.
Senior: enforcement analysis — size limits live in the presigned *policy* (S3 enforces), business validation must happen at minting time because S3 won't re-check; downloads have the mirror choice (`getFileStream` exists in `packages/storage/src/service.ts` for the streaming case).
Follow-ups: "When would you proxy anyway?" (virus scanning, transforms, private networks.)
Drill: whiteboard both sequence diagrams (proxy vs presigned) in under 2 minutes.

## Q15: What's the event-loop story of `setTimeout(fn, 0)` and why would production code use it?

What it's really testing: task-queue ordering intuition.
Repo anchor: `packages/js-core/src/index.ts:42-48` — after `setup()` resolves, `checkPageUrl` is scheduled with `setTimeout(..., 0)` so user commands queued synchronously after setup land in the CommandQueue *before* page-view processing.
Junior: "it delays by 0ms."
Mid: it defers to a macrotask — everything synchronous (including already-queued microtasks) runs first; here that ordering is a documented correctness requirement (the comment says exactly why).
Senior: frames it honestly — ordering-by-timing is fragile; the robust alternative is an explicit queue priority or an awaited hook; recognizing a *deliberate, commented* use versus a flaky-test hack is the judgment being tested.
Follow-ups: "queueMicrotask vs setTimeout 0?" "What changes in Node vs browser?"
Drill: predict console order for a snippet mixing await, queueMicrotask, and setTimeout — then explain how you knew.
