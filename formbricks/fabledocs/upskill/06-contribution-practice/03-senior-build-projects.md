# Senior Build Projects

Six projects, 2 days to 4 weeks. Each is realistic for this codebase — something a maintainer could plausibly accept given an RFC. Do the full design artifact even if you never build it; the artifact is interview gold.

## Project 1: Webhook delivery visibility (1 week)
Problem: customers can't see whether their webhooks fired (observability table, R1-adjacent).
Product value: cuts "my integration is broken" support tickets; prerequisite for retries.
Design checklist: `WebhookDelivery` table (webhookId, event, responseId, status, httpStatus, durationMs, error, createdAt) — write path in `pipeline/route.ts:119-177`; retention policy (30d? size math!); list UI under environment settings following an existing settings page's structure; authz identical to webhook management.
Likely files: pipeline route, new `modules/webhooks/` module, schema + migration.
Migration plan: additive table; feature-flag the writes.
Test plan: unit on the recorder (all outcome branches incl. SSRF-rejected), e2e asserting a delivery row after response.
Security: delivery payloads may contain response PII — store status + metadata, NOT bodies (write this decision down; it's the review battleground).
Performance: one insert per delivery — batch or fire-and-forget? (Irony noted: don't make the audit log less reliable than the thing it audits… or accept that it's telemetry. Decide explicitly.)
Rollout/rollback: flag on staging cloud first; rollback = flag off, table stays.
Open questions: expose redelivery button now or with Project 2?
Stretch: per-webhook health summary (success rate, p95).
Interview story potential: full-stack feature with data-retention and PII judgment.

## Project 2: Transactional outbox for pipeline events (2-4 weeks, RFC first)
Problem: R1 — at-most-once event delivery.
Design checklist: the month-3 plan in [03-architecture/06](../03-architecture-and-patterns/06-architecture-critique.md) is your skeleton — outbox row inside the Flow-1 transaction (`responses/lib/response.ts:24-41`); drainer choice (pg-boss on existing Postgres vs BullMQ on existing Redis — argue from the self-hosting constraint); keep `/api/pipeline` as execution target; backoff + dead-letter; idempotent consumers via existing `webhook-id`.
Test plan: the kill-9-under-load zero-loss test is the acceptance criterion; parity metrics during dual-write.
Rollout: dual-write flag → parity dashboard (needs M3) → cutover → cleanup.
Open questions: event ordering guarantees (per-survey FIFO or none — check what integrations assume); multi-pod drainer coordination (pg-boss handles; hand-rolled needs `FOR UPDATE SKIP LOCKED` — name it in the RFC).
Interview story: THE system-design story — take it to every loop.

## Project 3: Environment-state payload diet + delta sync (1-2 weeks)
Problem: hotspot 4 — every widget cold-load ships every in-progress survey.
Design checklist: measure first (payload size distribution across seeded tenants); trim fields the widget never reads (diff `TJsEnvironmentState` against actual widget usage in `packages/js-core`/`surveys`); then ETag/If-None-Match so unchanged state costs 304 (the route already has cache headers to build on, `environment/route.ts:60-75`).
Blast radius: the SDK contract — version carefully; old SDKs must keep working (additive only, or content-negotiated).
Test plan: SDK integration tests against both payload versions; bundle-size and payload-size assertions in CI.
Interview story: "API payload optimization with a deployed-clients constraint" — pairs beautifully with a frontend-perf interview.

## Project 4: Quota correctness under concurrency (3-5 days)
Problem: R2/Pattern 12, done properly (M6 is the ticket-sized version; this is the full treatment).
Design checklist: reproduce with a load test (two-phase: prove overshoot exists and its magnitude); comparison doc of four fixes (conditional UPDATE counter, FOR UPDATE, advisory lock, serializable+retry) with a benchmark of each under the Flow-1 transaction; pick; implement; regression suite.
The deliverable is the *comparison doc with numbers* as much as the fix.
Interview story: concurrency depth few mid-level candidates can evidence.

## Project 5: SDK error beacon + client health dashboard (2 weeks)
Problem: zero server-side visibility into widget failures (M8 is phase 1).
Design checklist: M8's endpoint; then aggregation (counts by error signature per environment), a settings-page panel, and sampling/quota logic so one broken customer page can't flood (rate-limit namespace + per-env daily cap — the cap needs state: where? Redis counter with TTL, reusing the limiter's own pattern).
Security: the endpoint is unauthenticated by nature — treat as hostile input end-to-end; scrub PII; size-cap bodies.
Interview story: "designed abuse-resistant telemetry from hostile runtimes."

## Project 6: Management API deprecation policy + v1→v2 migration guide (2-3 days, docs+code)
Problem: R5 — three API versions, no visible sunset story.
Design checklist: inventory v1/v2/v3 endpoint overlap (start from `openapi.yml` + route trees); write the policy doc (support windows, deprecation headers); implement `Deprecation`/`Sunset` headers in the v1 wrapper (`with-api-logging.ts` — one place); usage metrics per version (label the existing request logging) to make sunset data-driven.
Rejection risk: policy is a maintainer decision — frame the PR as a proposal with the mechanical parts (headers, metrics) ready.
Interview story: "API lifecycle management" — a topic senior interviews love and juniors never have evidence for.

---

Choosing: if you have two weeks and one shot, do Project 1 (shippable, visible, self-contained). If you're optimizing purely for interview ammunition, write Project 2's RFC + build Project 4 — reliability + concurrency covers the two hardest question families.
