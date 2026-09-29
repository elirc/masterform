# Boundaries and Layers

A *boundary* is where responsibility changes hands and a contract applies. A *layer* owns a concern and must not reach around its neighbors. This repo has clear intended layers with a few instructive leaks.

## The intended layering (write path)

```
Route handler / Server action        ← HTTP-shape concerns: parse, status codes, headers
  └─ modules/<feature>/lib services  ← business rules, orchestration
      └─ lib/<domain>/service.ts     ← older shared services (survey, response, organization)
          └─ Prisma (packages/database) ← persistence only
cross-cutting: packages/{cache,logger,storage,types}, modules/core/rate-limit, modules/ee/license-check
```

What each layer owns / must not own:
- **Routes/actions** own HTTP: Zod parsing, status mapping, cache headers, rate-limit config. Must not own business rules. Good example: `apps/web/app/api/v2/client/[environmentId]/responses/route.ts` keeps rules in `lib/` helpers and maps errors to statuses (:175-191).
- **Feature services** own rules and orchestration: `responses/lib/response.ts` decides the transaction scope; `follow-ups/lib/follow-ups.ts` owns follow-up eligibility. Must not build HTTP responses… ⚠ see Leak 1.
- **Persistence** owns storage shape. Must not own authorization (it doesn't — authz never appears in `packages/database`; correct).
- **packages/** own reusable mechanics with no product knowledge. `packages/cache` knows nothing about surveys — the key *registry* names domain concepts but that's deliberate centralization (Pattern 5).

## Two generations of code, one repo

`apps/web/lib/<domain>/service.ts` (older, flat services) coexists with `apps/web/modules/<feature>/lib/` (newer, feature-scoped). Both are called from new code — e.g. the v2 responses route imports `getSurvey` from `@/lib/survey/service` (route.ts:11) *and* feature-local helpers. This is normal codebase geology; the skill is telling strata apart before copying a pattern. Rule from `AGENTS.md`: new work goes in `modules/`.

## Boundary leaks worth studying (each is a review-comment template)

**Leak 1 — services returning HTTP responses.** `checkSurveyValidity` (`responses/lib/utils.ts:13-73`) returns `Response | null` — a *business validation* function that constructs HTTP `Response` objects, so it can only be reused by HTTP callers. Symptom: v1 and v2 share it awkwardly. Better shape: return a typed result; let routes map to HTTP. Judgment note: for a route-local helper this is pragmatic; it becomes debt the moment a non-HTTP caller (e.g. a future queue worker) needs the same rules. Both sentences belong in the review comment.

**Leak 2 — a "check" that mutates.** Same function, `:33`: `responseInput.singleUseId = singleUseValidationResult.singleUseId` — validation writing into its input. The caller's later logic silently depends on this. Contract-honesty issue more than a bug.

**Leak 3 — a write hidden in a cached read.** `environmentState.ts:29-56` flips `appSetupCompleted` inside the `withCache` callback. Documented and tolerated (one-time flag), but the *shape* — side effect whose execution depends on cache hits — is a leak between the caching layer's contract ("this is a pure read") and reality. If someone adds a second write there, it inherits cache-conditional execution invisibly.

**Leak 4 — cross-version imports.** v2 responses route imports from v1's tree (`responseSelection` from `api/v1/.../response`, `responses/lib/response.ts:13`; single-use from the unversioned `api/client/`). API versions share internals — fine for DRY, but it means "v1 is frozen" is false: editing v1 lib code changes v2 behavior. Blast-radius awareness, not a bug.

**Strong boundary worth imitating —** the EE fence: enterprise logic lives under `modules/ee/` with runtime license gates at entry points (`getIsContactsEnabled`, `responses/route.ts:82-96`). Grep-auditable, legally meaningful (separate LICENSE file), and gate-at-the-boundary rather than sprinkled ifs.

**Strong boundary #2 —** the SDK/API contract: `packages/js-core` talks to the server *only* through the public client API (no shared in-process types beyond `packages/types`). The widget is a true external client of its own backend — which keeps the public API honest (dogfooding).

## Interview angle

"Describe a layering violation you've found and how you'd fix it" — use Leak 1: name the smell (transport types below the transport layer), the cost (reuse blocked), the fix (typed results + route-level mapping), and the pragmatism (fine until a second transport exists). That last clause is what makes the answer senior rather than dogmatic.

Drill: pick `modules/survey/follow-ups/lib/follow-ups.ts` and grade its boundary hygiene against Leaks 1–3: does it return HTTP? mutate inputs? hide writes? Write the three-line verdict.
Self-grade — Basic: verdict written. Solid: verdict correct (it returns `Result`, not `Response` — cleaner than Leak 1). Strong: you noticed it *does* rate-limit inside the service (`applyRateLimit` import) and can argue whether throttling is a business rule or a transport concern — there's a defensible case each way.
