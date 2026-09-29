# Key Flows

Seven end-to-end traces. Each is a real production path with anchors confirmed against the working tree (2026-07-11). Do them in order; later modules reuse these flows but never re-trace them from scratch.

---

## Flow 1: Submitting a survey response (public client API)

Why this flow matters: it's the product's revenue-critical write path — unauthenticated, internet-facing, and full of layered validation. If you understand this one flow you understand half the codebase's design values.

Open these files first:
- `apps/web/app/api/v2/client/[environmentId]/responses/route.ts:204-278` — the POST handler, top of the funnel
- `apps/web/app/api/v2/client/[environmentId]/responses/lib/utils.ts:13-73` — `checkSurveyValidity`
- `apps/web/app/api/v2/client/[environmentId]/responses/lib/response.ts:21-45` — the transaction
- `packages/database/schema.prisma:158-190` — the `Response` model

Trace:

| Step | Owner | File | What happens | Data shape | Risk |
| --- | --- | --- | --- | --- | --- |
| 1 | Widget (browser) | `packages/surveys/src/lib/response-queue.ts:55-80` | `ResponseQueue` posts the accumulated answers | `TResponseUpdate` | Offline/net failure → IndexedDB persistence + retry |
| 2 | Route | `.../responses/route.ts:204-212` | Parse params; Zod-validate body via `parseAndValidateJsonBody` with `ZResponseInputV2` | JSON → `TResponseInputV2` | Malformed JSON → 400, never a crash |
| 3 | Route | `.../responses/route.ts:216-222` | EE gate: `contactId` only allowed if org license enables contacts | — | License check does a DB+cache hop |
| 4 | Service | `.../responses/lib/utils.ts:18-26` | Survey must belong to this environment and be `inProgress` | `TSurvey` | **The tenancy boundary.** Skip this and any environmentId works |
| 5 | Service | `.../responses/lib/utils.ts:28-34` + `api/client/[environmentId]/responses/lib/single-use.ts:14-80` | Single-use link validation (decrypt `suId`/`suToken` from the response URL) | `{ singleUseId }` | Depends on `ENCRYPTION_KEY` being set |
| 6 | Service | `.../responses/lib/utils.ts:36-70` | reCAPTCHA verification if the survey enables it | token → score | Spam protection is EE-gated |
| 7 | Route | `.../responses/route.ts:108-137` | Answer-level validation against survey blocks (`validateResponseData`) | `ResponseData` | Wrong-shaped answers → 400 with per-question details |
| 8 | DB | `.../responses/lib/response.ts:24-41` | `prisma.$transaction`: create response **and** evaluate quotas atomically | `Response` row + `ResponseQuotaLink` rows | Duplicate `singleUseId` → unique violation → 409 |
| 9 | Route | `.../responses/route.ts:245-259` | `sendToPipeline({event: "responseCreated"})` (and `responseFinished` if done) — **not awaited** | HTTP POST to self | Fire-and-forget; failure only logged |
| 10 | Route | `.../responses/route.ts:261-268` | Return `{ id, ...quotaFull }` to the widget | minimal payload | Deliberately doesn't echo the response body |

Validation and authorization: input shape at step 2 (Zod), business rules at steps 4–7, and note there is *no user auth* — this endpoint is deliberately public; the "authorization" is possession of a valid environmentId + survey in `inProgress` state, plus optional single-use tokens and reCAPTCHA. Contrast with the management API (Flow 5).

Persistence and side effects: one transaction (step 8); side effects deferred to the pipeline (step 9), *after* commit. The response is durable even if every downstream consumer fails.

Tests that cover it: colocated `route.test.ts` and lib tests exist throughout `apps/web/app/api/v2/client/[environmentId]/responses/` (verified by directory listing; e.g. `lib/utils` and `lib/response` have `.test.ts` siblings).

What juniors usually miss: the order — cheap checks (shape, scoping) before expensive ones (reCAPTCHA network call, DB transaction); and that `singleUseId` idempotency is enforced by the **database** (`schema.prisma:186`), not by an `if` statement.

What seniors notice: `sendToPipeline` is not awaited (route.ts:245). In a serverless runtime the platform may freeze the lambda after the response returns, so events could occasionally never fire — and there's no retry or outbox. That's a real availability/consistency tradeoff to discuss, not a typo.

Interview angle: "Design a form/survey submission endpoint" — this flow *is* the model answer: layered validation, tenancy check, DB-level idempotency, transactional quota accounting, post-commit event fan-out. See [08-interview-prep/03-api-and-data-modeling-questions.md](../08-interview-prep/03-api-and-data-modeling-questions.md) Q4, Q7.

Drill: without looking, write the ordered list of everything that can reject a submission (there are at least 8 rejection points). Then check yourself against route.ts and utils.ts.
Self-grade — Basic: you got shape validation and "survey not found." Solid: you included environment scoping, status, single-use, reCAPTCHA, per-answer validation. Strong: you also got the EE contact gate, the unique-constraint 409, and can say *why* the order matters (cost + blast radius).

---

## Flow 2: The response pipeline (webhooks, emails, integrations)

Why this flow matters: it's the async backbone — and it's *not* a queue. Understanding what Formbricks chose instead of Kafka/BullMQ, and what that costs, is a senior-level story.

Open these files first:
- `apps/web/app/lib/pipelines.ts:5-25` — the producer: HTTP POST to self with `CRON_SECRET` header
- `apps/web/app/api/(internal)/pipeline/route.ts` — the consumer (348 lines, read it all once)

Trace:

| Step | Owner | File | What happens | Data shape | Risk |
| --- | --- | --- | --- | --- | --- |
| 1 | Producer | `pipelines.ts:10-24` | `fetch(WEBAPP_URL + "/api/pipeline")` with `x-api-key: CRON_SECRET`; `.catch(log)` | `TPipelineInput` JSON | No retry, no persistence — an at-most-once bus |
| 2 | Consumer | `pipeline/route.ts:31-36` | Header equals `CRON_SECRET` or 401 | — | Shared-secret auth; one secret for the whole instance |
| 3 | Consumer | `pipeline/route.ts:44-79` | Zod parse; survey fetched and **re-checked against environmentId** | `TPipelineInput` | Defense in depth even on an internal route |
| 4 | Consumer | `pipeline/route.ts:82-93` | Find webhooks matching env + trigger + (surveyIds contains survey OR empty) | `Webhook[]` | Prisma array filters |
| 5 | Consumer | `pipeline/route.ts:96-177` | Deliver each webhook: validate URL (SSRF), pin DNS-resolved IP into an undici dispatcher, `redirect: "manual"`, 5s `AbortSignal.timeout`, Standard-Webhooks signature if secret set | signed JSON | TOCTOU DNS-rebinding defense; failures logged, **never retried** |
| 6 | Consumer (`responseFinished` only) | `pipeline/route.ts:179-254` | Integrations (Sheets/Airtable/Slack/Notion via `handleIntegrations`), notification-email user query, follow-up emails | — | Giant nested Prisma query at :192-243 |
| 7 | Consumer | `pipeline/route.ts:273-301` | `autoComplete`: if responseCount ≥ target, set survey `completed` + audit event | — | Check-then-act on a count — race under concurrency (investigate) |
| 8 | Consumer | `pipeline/route.ts:303-318` | `Promise.allSettled` over webhook+email promises; rejected ones logged | — | One slow webhook ≤5s can't sink the batch |
| 9 | Consumer (`responseCreated`) | `pipeline/route.ts:319-344` | Stripe metering, PostHog product analytics, telemetry | — | All best-effort |

Validation and authorization: steps 2–3. Note the *internal* route still re-validates survey↔environment (`:73-79`) — never trust the caller, even yourself.

Persistence and side effects: this route is *all* side effects; the only writes are autoComplete status and audit events. Everything here is at-most-once: crash mid-route and webhooks that hadn't fired are lost silently.

Tests that cover it: `pipeline/lib/handleIntegrations.test.ts`, `posthog.test.ts`, `telemetry.test.ts` (verified present). No test found exercising the full route's webhook delivery path end-to-end — Playwright specs don't cover external webhook targets (inferred from spec listing).

What juniors usually miss: `sendToPipeline` and this route run in the *same web app*. There is no worker process. Scale-out means these HTTP self-calls land on any pod behind the load balancer.

What seniors notice: (a) the SSRF triple-defense at :96-115 and :156-172 — URL validation + IP pinning + manual redirects — is textbook and worth quoting in interviews; (b) at-most-once delivery with no outbox means webhook consumers can't rely on Formbricks; (c) the `usersWithNotifications` query (:192-243) walks five relations — a latency hotspot candidate.

Interview angle: "How would you deliver webhooks reliably?" and "What is SSRF and how do you defend against it?" — see [08-interview-prep/04-system-design-from-this-repo.md](../08-interview-prep/04-system-design-from-this-repo.md) and 03-architecture/04-side-effects. The *gap* here (no retries/outbox) is senior project #2 in [06-contribution-practice/03-senior-build-projects.md](../06-contribution-practice/03-senior-build-projects.md).

Drill: draw the sequence diagram from widget → responses route → pipeline route → customer webhook, marking each hop as at-most-once / at-least-once / exactly-once. Then annotate where a crash loses data.
Self-grade — Basic: correct boxes and arrows. Solid: correct delivery semantics per hop. Strong: you can name the two fixes (await + outbox table + worker, or a real queue) and their operational cost.

---

## Flow 3: SDK environment sync (the read-side cache stack)

Why this flow matters: this endpoint is hit by every embedded widget on every customer page load. It's the repo's cleanest example of layered caching and of the consistency cost that buys.

Open these files first:
- `apps/web/app/api/v1/client/[environmentId]/environment/route.ts:20-80` — GET handler + cache headers
- `apps/web/app/api/v1/client/[environmentId]/environment/lib/environmentState.ts:19-71` — `withCache` usage
- `packages/cache/src/cache-keys.ts:17-50` — key discipline

Trace:

| Step | Owner | File | What happens | Data shape | Risk |
| --- | --- | --- | --- | --- | --- |
| 1 | SDK | `packages/js-core/src/lib/common/setup.ts` (via `index.ts:14-48`) | Widget boots, needs surveys/actionClasses/project config | — | Runs on customer pages — payload size matters |
| 2 | Route | `environment/route.ts:26-53` | environmentId trimmed + CUID-validated with Zod before touching the DB | string | Rejects `<environmentId>` placeholder junk cheaply |
| 3 | Cache | `environmentState.ts:22-70` | `cache.withCache(fn, key, 60_000)` — Redis-backed, 60s TTL, graceful fallback to direct fn | `TJsEnvironmentState["data"]` | Redis down → still serves (fail-open read) |
| 4 | DB | `environment/lib/data.ts` | Single optimized query for env + surveys + actionClasses | — | — |
| 5 | Side effect | `environmentState.ts:29-56` | First-ever call flips `appSetupCompleted=true` + PostHog "app_connected" — *inside the cached fn* | — | Write hidden inside a read path (senior eyebrow) |
| 6 | Route | `environment/route.ts:60-75` | Respond with `expiresAt: now+1h` and `Cache-Control: s-maxage=60, stale-while-revalidate=60` | JSON | Three cache layers now hold this data |

Validation and authorization: format validation only — the environment state is deliberately public (it's config for a public widget). Ask yourself what's in the payload and whether any of it is sensitive; that's a real review question.

Persistence and side effects: the sneaky `appSetupCompleted` write (step 5). It's guarded as "one-time, tolerates TTL," and the comment says so — an example of a *documented* rule-break.

Tests that cover it: `environmentState.test.ts` and `data.test.ts` colocated (verified present).

What juniors usually miss: there are **three** stacked caches with different TTLs — CDN (60s), Redis (60s), SDK client (re-check after 1h). "Why doesn't my published survey show up?" is answered here.

What seniors notice: no active invalidation. I searched write paths for cache deletion of `env:{id}:state` and found none (see verification log) — publishing a survey becomes visible when TTLs lapse, worst-case ~2 minutes for new page loads, up to 1h for already-loaded SDKs. That's an explicit consistency/latency tradeoff: **investigate** before claiming it's a bug; it may be intended.

Interview angle: "How do you cache an API response, and how do you invalidate it?" — the honest senior answer includes "sometimes you don't invalidate; you choose TTL-only and accept staleness." See [08-interview-prep/03-api-and-data-modeling-questions.md](../08-interview-prep/03-api-and-data-modeling-questions.md) Q10.

Drill: a customer says "I closed the survey but people kept answering for a minute." Write the timeline of which cache served what, and where the response POST got rejected (hint: Flow 1 step 4 checks status against the *DB*, not the cache).
Self-grade — Basic: you name the Redis TTL. Solid: all three layers with numbers. Strong: you separate read-path staleness (allowed) from write-path enforcement (still strict) and can defend that split.

---

## Flow 4: Saving and publishing a survey (server action write path)

Why this flow matters: it's the canonical *authenticated dashboard* write — server actions, layered authorization, audit logging, EE feature gates, and cache revalidation in one place.

Open these files first:
- `apps/web/modules/survey/editor/actions.ts:250-332` — `updateSurveyAction`
- `apps/web/lib/utils/action-client/index.ts:14-65` — the action client chain
- `apps/web/lib/utils/action-client/action-client-middleware.ts:94-121` — `checkAuthorizationUpdated`

Trace:

| Step | Owner | File | What happens | Data shape | Risk |
| --- | --- | --- | --- | --- | --- |
| 1 | UI | `modules/survey/editor/components/survey-menu-bar.tsx` | Save button calls `updateSurveyAction` | `TSurvey` | Whole survey object crosses the wire |
| 2 | Framework | `action-client/index.ts:14-49` | `next-safe-action` base client: error shaping, Sentry, audit event id | — | Expected errors → message; unexpected → generic |
| 3 | AuthN | `action-client/index.ts:51-65` | `getServerSession` → load user or throw | session | NextAuth cookie session |
| 4 | Input | `editor/actions.ts:250` | `.inputSchema(ZSurvey)` — full Zod parse of the survey | `TSurvey` | ZSurvey is one of the largest schemas in the repo |
| 5 | AuthZ | `editor/actions.ts:253-267` | `checkAuthorizationUpdated`: org role `owner`/`manager` **or** projectTeam permission ≥ `readWrite` | — | Two parallel permission systems joined by OR |
| 6 | EE gates | `editor/actions.ts:269-287` | Spam-protection, follow-ups, external-URL permissions (with grandfathering) | — | License checks before write |
| 7 | DB | `modules/survey/editor/lib/survey.ts` (`updateSurvey`) | Persist | `Survey` row | Json columns mean no schema migration for question changes |
| 8 | Audit | `editor/actions.ts:251,273-290` | `withAuditLogging("updated","survey",…)` wraps the action; old/new object captured | audit event | EE audit-logs module |
| 9 | Analytics | `editor/actions.ts:294-326` | PostHog diff events; `survey_published` when draft→inProgress | — | Telemetry only |
| 10 | Cache | `editor/actions.ts:328` | `revalidatePath(/environments/{envId}/surveys/{id})` | — | Revalidates *Next.js* cache — note: not the Redis env-state cache (Flow 3) |

Validation and authorization: steps 3–6 — the full stack: session → input schema → role/permission → license. Memorize this order; it's the repo's write-path contract.

Persistence and side effects: single update; audit + analytics after. Compare how much *less* transactional ceremony there is here vs Flow 1 (no quota logic, no unique constraints in play).

Tests that cover it: `editor/actions` has sibling tests in the module (verify with `ls apps/web/modules/survey/editor/` — inferred); authorization middleware has `action-client-middleware.test.ts` (verified present).

What juniors usually miss: `checkAuthorizationUpdated`'s access array is an **OR** of grants (`action-client-middleware.ts:105-119`) — you pass if *any* item passes. Reading it as AND inverts the security model.

What seniors notice: step 10 revalidates the dashboard's Next cache but nothing touches the Redis `env:{id}:state` key — which connects directly to Flow 3's staleness. Also `getOrganizationIdFromSurveyId(parsedInput.id)` runs *before* authorization: resource-existence is discovered pre-authz, a common and mostly-fine pattern, but worth knowing when it leaks existence info.

Interview angle: "How do you secure a mutation in a multi-tenant app?" — answer with this exact chain. See [08-interview-prep/03-api-and-data-modeling-questions.md](../08-interview-prep/03-api-and-data-modeling-questions.md) Q5 and [04-system-design-from-this-repo.md](../08-interview-prep/04-system-design-from-this-repo.md) step 4.

Drill: list what each of the five layers (session, schema, role, license, audit) would catch that the others wouldn't.
Self-grade — Basic: three layers. Solid: all five with a concrete miss each. Strong: you also flag the OR-semantics and the cache split.

---

## Flow 5: Management API request (API-key auth)

Why this flow matters: second authentication universe. Dashboard users get sessions (Flow 4); machines get hashed API keys with per-environment permissions.

Open these files first:
- `apps/web/app/api/v1/auth.ts:16-53` — `authenticateApiKey` / `authenticateRequest`
- `apps/web/modules/organization/settings/api-keys/lib/api-key.ts` — `getApiKeyWithPermissions` (hash lookup)
- `packages/database/schema.prisma:773-835` — `ApiKey` + `ApiKeyEnvironment`

Trace:

| Step | Owner | File | What happens | Risk |
| --- | --- | --- | --- | --- |
| 1 | Client | any HTTP client | Sends `x-api-key` header | Keys are bearer credentials — full power of their grants |
| 2 | Auth | `api/v1/auth.ts:45-53` | Extract header, delegate to `authenticateApiKey` | Missing header → null → 401 by caller |
| 3 | Auth | `api/v1/auth.ts:16-43` | Look up key **with permissions**; build `TAuthenticationApiKey` with `environmentPermissions[]` | Org-only keys rejected unless route opts in |
| 4 | Route | e.g. `apps/web/app/api/v1/management/` handlers | Route checks the specific environment/permission | Authz is per-route — a route that forgets is an IDOR |
| 5 | Rate limit | `modules/core/rate-limit/rate-limit-configs.ts:11-16` | 100/min per API namespace | Fail-open if Redis down |

What juniors usually miss: authentication (step 3) only says *who you are*; each route must still check `environmentPermissions` against the resource it's touching (step 4). The comment at `auth.ts:27` says exactly this.

What seniors notice: v1/v2/v3 coexist (`apps/web/app/api/`), and v2 has its own wrapper stack (`apps/web/modules/api/v2/auth/authenticated-api-client.ts`). Version sprawl is a maintenance tax — ask "what's the deprecation story?"

Interview angle: sessions vs API keys vs OAuth — when each fits. [08-interview-prep/03-api-and-data-modeling-questions.md](../08-interview-prep/03-api-and-data-modeling-questions.md) Q6.

Drill: find one v2 management route and write down where it enforces the environment permission. (Start at `apps/web/modules/api/v2/management/`.)
Self-grade — Basic: found the route. Solid: found the exact permission check. Strong: you can say what happens with an org-only key and why `allowOrganizationOnlyApiKey` exists.

---

## Flow 6: Login with credentials (+2FA)

Why this flow matters: the most security-dense 150 lines in the repo — rate limiting, timing-attack defense, DoS caps, and one-time backup codes, all readable in a single `authorize` function.

Open these files first:
- `apps/web/modules/auth/lib/authOptions.ts:190-310` — `authorize`
- `apps/web/modules/core/rate-limit/rate-limit.ts:13-135` — the limiter it calls

Trace:

| Step | File | What happens | Why it's there |
| --- | --- | --- | --- |
| 1 | `authOptions.ts:191` | `applyIPRateLimit(rateLimitConfigs.auth.login)` — 10 per 15min per IP | Brute-force cost |
| 2 | `authOptions.ts:205-216` | Reject passwords >128 chars | bcrypt CPU-DoS cap |
| 3 | `authOptions.ts:221-235` | Find user; verify against `user.password || CONTROL_HASH` | **Constant-time**: unknown emails still pay the bcrypt cost, defeating user-enumeration timing |
| 4 | `authOptions.ts:238-260` | Only now branch on user-missing / inactive / wrong password — all return the same "Invalid credentials" | No information leak in error text |
| 5 | `authOptions.ts:266-306` | 2FA backup code: decrypt stored codes, match, **null out the used code and re-encrypt** | One-time use enforced by mutation |
| 6 | `rate-limit.ts:46-61` | The limiter itself: Lua INCR+EXPIRE, atomic across pods | Race-free counting |

What juniors usually miss: step 3's control hash. Without it, "email not found" returns in 1ms and "wrong password" in 100ms — a timing oracle.

What seniors notice: the limiter fails *open* (`rate-limit.ts:129-134`) — if Redis is down, login brute-forcing is unthrottled. Availability was chosen over strictness; know that this is a choice, and know the alternative (fail-closed for auth namespaces only).

Interview angle: "How do you prevent user enumeration?" / "Where would you rate limit?" — [08-interview-prep/01-js-ts-node-deep-dive.md](../08-interview-prep/01-js-ts-node-deep-dive.md) Q14 and security checklist. This flow is also a ready-made STAR story about reading unfamiliar security code.

Drill: annotate every `throw` in `authorize` with what an attacker could learn from it (message + timing). Confirm they're uniform.
Self-grade — Basic: listed the throws. Solid: explained the control hash. Strong: proposed fail-closed rate limiting for auth and can argue both sides.

---

## Flow 7: File upload from a survey (signed URLs)

Why this flow matters: the repo's cleanest "don't proxy bytes through your server" example, plus an unauthenticated endpoint that still enforces tenancy and quotas-by-license.

Open these files first:
- `apps/web/app/api/v1/client/[environmentId]/storage/route.ts:29-113` — the POST handler
- `apps/web/modules/storage/service.ts:16-30` — `getSignedUrlForUpload`
- `packages/storage/src/service.ts` — the S3 presigned-post machinery

Trace:

| Step | Owner | File | What happens | Risk |
| --- | --- | --- | --- | --- |
| 1 | Widget | file-upload question | Requests an upload slot: fileName, fileType, surveyId | Public endpoint |
| 2 | Route | `storage/route.ts:33-55` | Zod parse (`ZUploadPrivateFileRequest`) | Malformed → 400 |
| 3 | Route | `storage/route.ts:59-80` | Survey + org fetched in parallel; **survey.environmentId must equal the URL param** | Same tenancy invariant as Flow 1 |
| 4 | License | `storage/route.ts:82-85` | Max upload size 10MB standard / bigger for licensed orgs | EE gate on a limit, not a feature |
| 5 | Storage | `modules/storage/service.ts:16+` | Build presigned S3 POST (random UUID prefix, sanitized filename) | Server never touches file bytes |
| 6 | Rate limit | `storage/route.ts:112` | `customRateLimitConfig: rateLimitConfigs.storage.upload` (5/min) | Abuse cap on slot-minting |
| 7 | Widget → S3 | direct upload | Browser PUTs/POSTs to S3 with the signed fields | Size limit enforced by the presigned policy |

What juniors usually miss: the server hands out a *capability* (the signed URL), not the upload itself. Validation must happen at slot-minting time because S3 won't re-check business rules.

What seniors notice: filename sanitization (`modules/storage/utils.ts`) and the random key prefix prevent path collisions and overwrite attacks; the 5/min rate limit is per-identifier — check what the identifier is (IP? environment?) in `with-api-logging.ts` before assuming it stops a distributed abuser.

Interview angle: "How do you handle file uploads at scale?" — presigned URLs vs proxying: latency, cost, validation tradeoffs. [08-interview-prep/03-api-and-data-modeling-questions.md](../08-interview-prep/03-api-and-data-modeling-questions.md) Q12.

Drill: write the failure matrix — what happens if (a) the signed URL expires, (b) the file exceeds the size, (c) the same fileName is uploaded twice, (d) S3 is down. Find the code that determines each.
Self-grade — Basic: 2 of 4 grounded in code. Solid: all 4. Strong: you also identified where a *malicious* fileType would be caught (or not) and framed it as a review comment.
