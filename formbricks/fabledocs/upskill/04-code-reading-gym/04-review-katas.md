# Review Katas

Nine fake PRs. For each: read the intent and diff summary, list your findings graded **Blocking / Important / Optional**, then compare. All diffs are invented; the "resembles" anchors are real code the diff pretends to touch. Practice kind, specific language — every expected finding below is phrased as you could post it.

## Kata 1: "Add Discord webhook support"
Intent: new webhook type delivering to Discord.
Diff: adds a `discord` branch in the pipeline route that `fetch(webhook.url)`es directly, skipping `validateAndResolveWebhookUrl` "because Discord URLs are trusted."
Resembles: `pipeline/route.ts:156-172`.
Blocking: "Skipping URL validation reopens the SSRF hole the pinned dispatcher exists to close — an attacker can save any URL with type=discord. Can we keep the validate+pin path and just add Discord payload formatting?"
Important: no timeout on the new fetch; missing signature headers for consistency.
Optional: dedupe the payload-building with the generic branch.

## Kata 2: "Speed up response list"
Intent: dashboard responses page is slow.
Diff: adds `take: 10000` removal (fetch all), moves filtering to JS `Array.filter`, adds `include: { contact: true, tags: true }` everywhere.
Resembles: `apps/web/lib/response/service.ts`.
Blocking: unbounded query — memory + latency blow up with volume; filtering must stay in SQL (WHERE + the `@@index([surveyId, createdAt])` at `schema.prisma:188`).
Important: `include`-everything widens payloads; use explicit `select` like `responseSelection`.
Optional: add the missing pagination cursor instead (see `use-surveys.ts` pattern).

## Kata 3: "Quick admin action to rename any survey"
Intent: support tooling.
Diff: new server action using `actionClient` (not `authenticatedActionClient`), takes `surveyId, name`, updates directly.
Resembles: `editor/actions.ts:250-267`.
Blocking (two): unauthenticated action = public rename endpoint; no `checkAuthorizationUpdated` = cross-tenant IDOR even if authenticated.
Important: no audit logging for a mutating admin tool (house rule: `withAuditLogging`).
Optional: Zod schema missing `.trim()`/length limit on name.
This kata is the *count-the-layers* exercise: session, schema, authz, audit — the diff has zero of four.

## Kata 4: "Retry failed webhooks"
Intent: address delivery gaps.
Diff: wraps webhook fetch in a `for (let i=0; i<5; i++)` loop with `await delay(2**i * 1000)`, inside the existing pipeline request.
Resembles: `pipeline/route.ts:119-177`.
Blocking: up to ~31s of in-request retries × N webhooks holds the pipeline HTTP call (and its caller's unawaited fetch) far past sensible request lifetimes; retries must move out of the request path (outbox/queue — link the critique R1).
Important: retrying non-idempotent consumer endpoints without honoring the existing `webhook-id` dedupe story; retry on 4xx is wrong (only 5xx/network).
Optional: `delay` already exists in `response-queue.ts` — but that's a *client* package; don't import across that boundary.
Meta-lesson: a fix in the wrong layer can be worse than the gap.

## Kata 5: "Cache survey in the response endpoint"
Intent: cut a DB read from the hot path.
Diff: wraps `getSurvey(surveyId)` in `withCache(..., 10min)` inside `responses/route.ts`.
Resembles: `responses/route.ts:224` + `environmentState.ts:22-70`.
Blocking: submission validation (status! singleUse config!) now trusts 10-minute-old data — a paused survey accepts responses for up to 10 minutes. Read-path staleness was safe *because* the write path stayed strict (Flow 3 notes); this diff breaks that split.
Important: no invalidation on survey update compounds it.
Optional: if latency data justifies caching, cache only immutable-ish fields, short TTL, or invalidate on update.

## Kata 6: "Tidy up checkSurveyValidity"
Intent: refactor for readability.
Diff: reorders checks so reCAPTCHA verification runs first ("fail fast on bots"), merges the env check into the end, returns `boolean` instead of `Response | null`.
Resembles: `responses/lib/utils.ts:13-73`.
Blocking: tenancy check moved *after* a network call to Google — cross-environment probes now trigger outbound requests and burn quota; cheap/security checks must precede expensive ones.
Important: `boolean` loses *which* failure (API consumers need distinct statuses/details); the Response-building critique is fair (Leak 1) but the fix is typed results, not booleans.
Optional: the input mutation at :33 could be removed in the same PR — scope creep? Discuss, don't demand.

## Kata 7: "Add company field to signup"
Intent: marketing wants company names.
Diff: adds field to the signup form component and to `User` model + migration; no changes to invite flow, SSO signup, or the Zod signup schema (form "validates" via HTML required).
Resembles: `modules/auth/signup/`, `schema.prisma:908-946`.
Blocking: server-side schema doesn't validate the new field — client-only validation is no validation.
Important: two other account-creation paths (invite accept, SSO JIT) now create users without the field — nullable? required? decide once; migration adds a NOT NULL column without default → fails on existing rows.
Optional: PII implications — where does it show, who can edit.

## Kata 8: "Fix flaky e2e"
Intent: `survey.spec.ts` flakes.
Diff: adds `await page.waitForTimeout(3000)` in four places and a retry loop around the whole test.
Resembles: `apps/web/playwright/survey.spec.ts`.
Blocking: none (tests only) — but Important: fixed sleeps trade flake-rate for guaranteed slowness and hide the actual race; use event-based waits (`expect(locator).toBeVisible()`, response waits) and find what's actually async (often the surveys-bundle load or the 60s cache — a *product* insight hiding in a test flake).
Optional: quarantine tag instead of blanket retries.

## Kata 9: "Single-use links for app surveys"
Intent: extend single-use beyond link surveys.
Diff: removes the `survey.type !== "link"` early-return in `validateSingleUseResponseInput` and calls it for app surveys too.
Resembles: `single-use.ts:19-21`.
Blocking: app surveys don't flow through URLs with suId/suToken params — the validator reads `responseInput.meta.url` (:43+); app-survey metas won't carry tokens, so this breaks all app-survey submissions for surveys with singleUse config lingering enabled.
Important: the *feature* needs a design (where does a token live in the SDK flow?) — request an RFC, kindly: "I think this needs a short design note on token transport for in-app surveys before code."
Optional: —.

---

## Grading yourself

Per kata — Basic: caught the Blocking issue. Solid: caught Blocking + Important with anchors to precedent. Strong: your comments propose a path (not just a verdict) and correctly *rank* severity — ranking is what interviewers watch in review rounds. If you flagged Optional nits as Blocking anywhere, that's the thing to fix about your reviewing before interviews; it reads as junior faster than missing a bug does.
