# API and Data-Modeling Questions

Thirteen cards. Round: API/data unless noted.

## Q1: Design the URL structure for a multi-tenant API.
Anchor: `/api/v2/client/{environmentId}/responses` vs `/api/v1/management/...` (`apps/web/app/api/` tree).
Junior: nests everything RESTfully.
Mid: tenancy scope *in the path* for the public API (environmentId as the partition key of every request), separate trees per audience (client vs management), versioned roots.
Senior: why the tenant id in the URL matters — cacheable per-tenant, logs/rate-limits keyed trivially, and the handler has an unambiguous scope to validate resources against (`utils.ts:18-20`); contrasts with header/subdomain tenancy and their CDN implications.
Follow-ups: "Where would a `surveyId` in the path vs body matter?"

## Q2: What makes an endpoint idempotent, and which of your endpoints need it?
Anchor: `@@unique([surveyId, singleUseId])` (`schema.prisma:186`) → 409 mapping (`responses/route.ts:180-182`); client retries from `response-queue.ts`.
Junior: "GET is idempotent."
Mid: retried POSTs are the real problem; idempotency keys make retries safe; here the single-use id doubles as one, enforced by the DB.
Senior: notes the *gap* — non-single-use responses have no idempotency key, so a client retry after a lost ACK duplicates a response (probability low, product-impact low, but say it); and the webhook side (`webhook-id` header) as consumer-facing idempotency groundwork.
Follow-ups: "Design the idempotency-key table for the general case."

## Q3: Walk me through your validation stack for one write endpoint.
Anchor: the seven-layer table from [03-architecture/03](../03-architecture-and-patterns/03-validation-auth-and-permissions.md) applied to Flow 1.
Junior: "we validate with Zod."
Mid: format → existence → tenancy → state → field rules → DB constraints → rate limits, each with its failure status.
Senior: the ordering rationale (cost + information leak) and which layer *the DB must own* (uniqueness under concurrency) — atomicity vs isolation vocabulary deployed correctly.

## Q4: How do you evolve an API without breaking clients?
Anchor: v1/v2/v3 side-by-side; v2 reusing v1 internals (`responses/lib/response.ts:13`); `openapi.yml`; widget SDK as an unupgradeable client.
Junior: "version the API."
Mid: additive-within-version, new version for breaking; deprecation headers + sunset windows; generated OpenAPI as the contract artifact.
Senior: the *organizational* cost of parallel versions (shared-internals coupling — editing v1 libs changes v2, R5) and the policy gap (Project 6); "versioning is easy, *sunsetting* is the hard part" is the senior sentence.

## Q5: How do you prevent IDOR?
Anchor: resolve-up idiom — `getOrganizationIdFromSurveyId` then `checkAuthorizationUpdated` (`editor/actions.ts:252-267`).
Junior: "check the user is logged in."
Mid: per-resource ownership resolution before every use of a client-supplied id; the cross-tenant test (Recipe 4) as regression armor.
Senior: makes it systemic — the idiom lives in middleware-adjacent helpers so the safe path is the short path; audit strategy (count actions vs authz calls); existence-leak nuance (404 vs 403 policy).

## Q6: Sessions vs API keys vs OAuth — when each?
Anchor: three universes ([03-architecture/03](../03-architecture-and-patterns/03-validation-auth-and-permissions.md)); hashed keys with per-environment grants (`api/v1/auth.ts:16-43`).
Junior: defines each.
Mid: humans-with-browsers → cookie sessions (CSRF-aware); machines → keys (scoped, revocable, hashed at rest); third-party-acting-for-user → OAuth.
Senior: scoping design — keys grant per-environment permissions (an authz model, not just authn); org-only keys as a separate class; rotation/last-used (Ticket M10) as lifecycle table stakes.

## Q7: Model a survey with heterogeneous question types. Tables or documents?
Anchor: the Json bet (`schema.prisma:357-370`) + what stayed relational.
Junior: picks one absolutely.
Mid: both, by rule — enforce/aggregate → columns+constraints; app-owned flexible structure → Json+Zod; gives the singleUseId-constraint vs blocks-Json split as the worked example.
Senior: the *costs* ledger — no FK into documents (dangling follow-up refs, scenario 5), schema-drift over old rows, Json-path query ergonomics — and the mitigation for each (app-level validation M9, backward-compatible Zod, generated columns if aggregation arrives).

## Q8: Design response ingestion for burst traffic.
Anchor: Flow 1's shape — thin validation, one transaction, deferred side effects.
Junior: "add a load balancer."
Mid: keep the critical path short (the repo's post-commit pipeline is this); DB writes append-only + indexed; rate limits per environment.
Senior: what actually falls over in order — connection pool (interactive tx duration), quota contention (Pattern 12 lock ladder), the EE license lookups per request (cacheable); then the queue conversation with the self-hosting constraint. Bursts are a *sequencing-of-bottlenecks* answer, not an infra shopping list.

## Q9: Transactions — what goes inside, what stays out, and why?
Anchor: `responses/lib/response.ts:24-41` (in: response+quota links) vs pipeline (out).
Junior: "wrap related writes."
Mid: smallest atomic business fact; no network I/O inside; error → rollback → the 409/400 mapping.
Senior: isolation-level honesty (READ COMMITTED default; the count race), `tx ?? prisma` fallback as a silent-atomicity-break risk, and pool-time economics of interactive transactions.

## Q10: Cache an API response and tell me the whole invalidation story.
Anchor: Flow 3's three layers; TTL-only; no active invalidation found (verification log).
Junior: "Redis with a TTL."
Mid: layer-by-layer TTLs, worst-case staleness math, fail-open on Redis loss.
Senior: the decision framework — per entity, choose bounded-staleness (TTL) vs active invalidation by *write-visibility requirements*; pause-survey arguably deserves the `del` (Ticket M1); CDN layers can't be purged per-key on most plans, so TTLs bound what invalidation can't reach. Plus the write-path-stays-strict principle that makes read staleness safe.

## Q11: Design webhooks as a provider.
Anchor: the whole webhook subsystem — matching (`pipeline/route.ts:82-93`), signing (:136-154, Standard Webhooks), SSRF defense (:96-177), and the missing retries (R1).
Junior: "POST the event to their URL."
Mid: signatures (why: consumer authenticates *you*), timeouts, per-event types, secrets per endpoint.
Senior: delivery semantics up front (at-most-once today; at-least-once + consumer dedupe is the industry answer — the message-id header already anticipates it); SSRF as the provider-side threat; delivery visibility as the support-cost fix (Project 1). This card + Q2 chain into the system-design variation 5.

## Q12: File uploads: presigned or proxied?
Anchor: `storage/route.ts:29-113`; `getSignedUrlForUpload` (`modules/storage/service.ts:16-30`).
Junior: "multer to disk."
Mid: presigned = bytes bypass the app; validate at mint-time; policy enforces size; rate-limit the minting.
Senior: when to proxy anyway (AV scanning, transforms, private buckets with VPC endpoints); the *download* side (signed GETs vs streaming through — `resolveStorageUrlsInObject` in the pipeline shows URLs being rewritten for webhook payloads: files referenced in events need resolvable URLs — a subtle contract worth mentioning).

## Q13: How would you do a zero-downtime schema change on a hot table?
Anchor: `Response` (the hot table); migration history discipline; `DataMigration` tracking model (`schema.prisma:563`).
Junior: "run the migration at night."
Mid: additive playbook — nullable column → dual-write → backfill → read-switch → drop later; index creation concurrently.
Senior: the Json-document twist this repo adds (Zod-schema changes are invisible migrations — old rows must stay parseable); lock analysis per DDL type; and the rollback answer ("every phase independently revertible; the *data* migration is the point of no return, so it goes last and is verified first").
