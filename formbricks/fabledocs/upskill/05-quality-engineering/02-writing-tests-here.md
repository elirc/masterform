# Writing Tests Here

Seven recipes using this repo's actual conventions: Vitest, colocated files, `vi.mock` at module boundaries, `__mocks__` for shared fixtures. Commands are __inferred__ from package scripts — verify the exact test invocation in `apps/web/package.json` once, then trust it.

Run targeted: `pnpm --filter @formbricks/web test -- <path-fragment>` (or from `apps/web/`: `pnpm test <path>`). Package suites: `pnpm --filter @formbricks/cache test`.

## Recipe 1: Happy-path service test (response creation)

Target: `createResponseWithQuotaEvaluation` (`responses/lib/response.ts:21-45`).
Shape: mock `@formbricks/database` so `prisma.$transaction` invokes your callback with a `tx` stub; stub `evaluateResponseQuotas` to return `{shouldEndSurvey:false}`; assert the created response is returned and quota evaluation received the same `tx`.
The assertion that matters: **the `tx` object identity** — that's the atomicity contract (Q12 territory). A test asserting only the return value tests nothing this function is *for*.

```ts
// Illustrative fake code: not from this repo (test sketch)
vi.mock("@formbricks/database", () => ({ prisma: { $transaction: vi.fn((cb) => cb(txStub)) } }));
expect(evaluateResponseQuotas).toHaveBeenCalledWith(expect.objectContaining({ tx: txStub }));
```

## Recipe 2: Validation failure matrix

Target: `checkSurveyValidity` (`responses/lib/utils.ts:13-73`).
Shape: table-driven — `[wrong environmentId → "does not belong"], [status draft → 403], [singleUse without ENCRYPTION_KEY → 500], [recaptcha enabled, no token → 400 code recaptcha_verification_failed]`. Mock the EE license helpers per-case.
House detail: assert on the parsed body of the returned `Response` (they're real Response objects — Leak 1 makes tests slightly clunky; feel that cost, it strengthens the refactor argument).

## Recipe 3: Permission failure (server action)

Target: any action using `checkAuthorizationUpdated`, e.g. `updateSurveyAction`.
Shape: mock `getMembershipRole` → `"member"`, `getProjectPermissionByUserId` → `"read"`; expect `AuthorizationError`. Then the green twin: `manage` permission passes.
Reference for mocking the session chain: `apps/web/lib/utils/action-client/index.test.ts` and `action-client-middleware.test.ts` — copy their setup rather than inventing one.

## Recipe 4: Cross-tenant rejection (the IDOR test)

Target: v2 responses route.
Shape: build survey fixture with `environmentId: "env_A"`, POST against `env_B`'s URL; assert 400 "Survey does not belong to this environment" and — the part juniors skip — assert `prisma.response.create` was **never called**.
Generalize: every list/get endpoint deserves one "other tenant's id" test. If you add one such test to any untested route, that's a real contribution (Ticket family).

## Recipe 5: Async side effect (pipeline events)

Target: the POST handler's pipeline calls (`responses/route.ts:245-259`).
Shape: mock `sendToPipeline`; submit unfinished response → exactly one call (`responseCreated`); finished → two calls, and assert the *payloads* (event names, surveyId).
Trap this recipe teaches: the call is unawaited — if your handler-under-test resolves before the mock registers, you've reproduced the production race in miniature. `await vi.waitFor(() => expect(sendToPipeline).toHaveBeenCalled())` — and now you understand the serverless caveat experientially.

## Recipe 6: Cache behavior (hit, miss, fallback)

Target: `getEnvironmentState` (`environmentState.ts:19-71`).
Shape: mock `cache.withCache` two ways — (a) pass-through executing the fn (miss): assert DB called and `appSetupCompleted` updated when false; (b) returning canned data without executing (hit): assert DB **not** called. Case (c): fn throws Redis-ish error — what's the contract? Read `service.ts:241+` first, then encode it.
Reference: `packages/cache/src/service.test.ts` and `cache-integration.test.ts` show the package's own idioms.

## Recipe 7: E2E journey (Playwright)

Target: close-survey message (Ticket 15).
Shape: copy the skeleton of `apps/web/playwright/survey.spec.ts` — its fixtures create account/org/survey; add: publish → close → visit link → `expect(page.getByText(closedMessage)).toBeVisible()`.
Run: `pnpm test:e2e -- survey` (filter syntax — verify against the playwright config). Budget minutes, not seconds; that's the layer's price.

## Migration-behavior note

No dedicated migration test harness found. The pragmatic equivalent here: `pnpm db:migrate:dev` against a seeded local DB + the `DataMigration` model's tracking (`schema.prisma:563`). For a schema-changing PR, your "test" is the reversibility note in the PR body + running the migration both directions locally. Honest limits are better than imaginary coverage.

Drill: actually write Recipes 2 and 4 (they're the highest interview-value per hour) and get them green. Bring one to interviews as your "how I test" exhibit — walking through a real cross-tenant test you wrote is worth three abstract testing-philosophy answers.
