# Annotation Drills

Ten drills. For each: open the anchor, and annotate — **inputs, outputs, dependencies, invariants, side effects, failure modes** — before reading the "what to find" notes. Grade with the rubric at the bottom.

## Drill 1: `sendToPipeline`
Anchor: `apps/web/app/lib/pipelines.ts:5-25`.
What to find: input `TPipelineInput`; output a promise nobody meaningfully uses; deps `CRON_SECRET`, `WEBAPP_URL`, global fetch; invariant *attempted* — CRON_SECRET must exist (throws!) — note the asymmetry: missing config throws synchronously, network failure only logs; side effect: HTTP POST; failure modes: DNS/self-unreachable, consumer 4xx (silently logged), process death pre-flush.

## Drill 2: `checkSurveyValidity`
Anchor: `apps/web/app/api/v2/client/[environmentId]/responses/lib/utils.ts:13-73`.
What to find: the tenancy invariant (:18-20) is the *first* check; mutation of the input parameter (:33 — `responseInput.singleUseId = ...` — a function named "check" that writes; flag it); EE license lookups mid-validation (network/DB dependency inside a "pure-sounding" function); return convention `Response | null` where null = valid (inverted-feeling contract worth annotating).

## Drill 3: The rate-limit Lua script
Anchor: `apps/web/modules/core/rate-limit/rate-limit.ts:35-66`.
What to find: window arithmetic (fixed windows keyed by `windowStart`); the invariant "EXPIRE set exactly once per window" enforced by `current == 1`; failure mode: fail-open at three distinct points (disabled flag :18, no Redis :28-33, exception :112-134) — enumerate all three; dependency: clock (`Date.now()`) — what happens with clock skew across pods? (Nothing terrible: windows are per-key computed from each pod's clock — worth one sentence.)

## Drill 4: `createResponseWithQuotaEvaluation`
Anchor: `apps/web/app/api/v2/client/[environmentId]/responses/lib/response.ts:21-45`.
What to find: transaction boundary = atomicity unit (response + quota links commit together); `tx` threading as an explicit dependency; the shape merge at :37-40 (`quotaFull` added conditionally — output type `TResponseWithQuotaFull`); invariant: no pipeline events until after this returns (check the caller!); failure mode: any quota error rolls back the *response itself* — is that intended? (Annotate as a question — that's a legitimate review finding either way.)

## Drill 5: Webhook delivery closure
Anchor: `apps/web/app/api/(internal)/pipeline/route.ts:119-177`.
What to find: per-webhook closure capturing `webhook`, `body`, fresh headers; the signature only when `webhook.secret` exists; SSRF layers in order (validate → pin → manual redirect → timeout); `finally { dispatcher?.destroy() }` — resource cleanup as an invariant; failure mode: `.catch` per webhook means one failure is isolated — connect to `allSettled` downstream.

## Drill 6: `authorize` (login)
Anchor: `apps/web/modules/auth/lib/authOptions.ts:190-260`.
What to find: ordering as a security invariant (rate limit before any work; constant-time verify before any branching); the CONTROL_HASH trick; outputs: user object or thrown uniform errors; side effects: audit log lines and (later, :298-306) backup-code consumption — a *read* path that writes; failure modes: enumerate what each throw reveals (should be: nothing distinguishable).

## Drill 7: `getEnvironmentState`
Anchor: `apps/web/app/api/v1/client/[environmentId]/environment/lib/environmentState.ts:19-71`.
What to find: the cached-function contract (pure-ish, but :29-56 hides a write + PostHog call that only fires on cache miss); TTL 60s; dependency chain Redis → single optimized query; invariant: payload must be safe-for-public; failure mode: Redis down → every request hits DB (fail-open = load risk under traffic spike).

## Drill 8: `checkAuthorizationUpdated`
Anchor: `apps/web/lib/utils/action-client/action-client-middleware.ts:94-121`.
What to find: OR-semantics over the access array (first success returns); role fetched once (:103) but team checks query per-item (N+1-ish if many access items — usually 2, so fine; annotate the scaling assumption); the odd middle case: `checkOrganizationAccess` can return a *validation-errors object* (truthy, non-true) — trace what happens then (:107-109); failure mode: throws `AuthorizationError` only after all grants fail.

## Drill 9: `ResponseQueue.add`/sync locks
Anchor: `packages/surveys/src/lib/response-queue.ts:36-110` (read past the shown range in your editor).
What to find: two locks (`syncing` vs `requestInProgress`) — annotate why one isn't enough; IndexedDB id mapping for cleanup (`pendingDbIds`); invariant: at most one in-flight send per survey across *all* instances; failure modes: storage quota full, retry exhaustion (what happens to the queue item? read `sendResponse` to answer).

## Drill 10: Storage upload handler
Anchor: `apps/web/app/api/v1/client/[environmentId]/storage/route.ts:29-113`.
What to find: parallel fetches (:59-62) — annotate why parallelism is safe here (independent reads); the tenancy check *after* both fetches (order fine? cheap enough); license-based limit as input to the presigned policy; error split at :98-105 (5xx carries `error` up to the wrapper for reporting, 4xx doesn't) — that's the observability contract.

---

## Self-grading rubric (per drill)

- **Basic**: you correctly listed inputs/outputs and at least one failure mode.
- **Solid**: you identified every side effect (including logs/analytics), stated the key invariant in one sentence, and traced one dependency you initially missed.
- **Strong**: you found at least one thing worth a review comment (a naming lie, a hidden write, an ordering assumption, a scaling assumption) and phrased it as a question a maintainer would welcome — e.g. for Drill 2: "`checkSurveyValidity` assigns `responseInput.singleUseId` as a side effect — would returning the decrypted id keep this function read-only?"

Do at most two drills per sitting. Write annotations in the margin (or a scratch file), don't just think them — the writing is the drill.
