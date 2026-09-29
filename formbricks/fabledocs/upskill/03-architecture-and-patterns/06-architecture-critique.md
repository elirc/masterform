# Architecture Critique

An honest assessment: strongest choices, real risks, and a prioritized "if I owned this for 3 months" plan. Confirmed observations are anchored; hypotheses are labeled. This file doubles as system-design interview material — cross-linked from [08-interview-prep/04](../08-interview-prep/04-system-design-from-this-repo.md).

## Strongest design choices (steal these)

1. **Security engineering on the hot boundaries.** Constant-time login with control hash (`authOptions.ts:232-235`), SSRF triple defense with the TOCTOU explicitly named in comments (`pipeline/route.ts:96-172`), bcrypt DoS cap, presigned uploads keeping bytes out of the app, pnpm postinstall allow-listing. This is top-decile for open source.
2. **Invariants pushed into the database.** `@@unique([surveyId, singleUseId])`, unique displayId, commented indexes (`schema.prisma:184-189`). The app treats constraint violations as domain outcomes (409s), not crashes.
3. **The typed-registry discipline, twice.** Cache keys (`packages/cache/src/cache-keys.ts`) and TanStack Query keys (`modules/survey/list/lib/query`) — the same idea at two layers; nobody hand-builds keys.
4. **One transaction around the one multi-write business fact** (`responses/lib/response.ts:24-41`), side effects strictly post-commit. Textbook.
5. **The infrastructure-minimal deployment story.** Postgres + Redis + S3, no queue, no workers — a self-hoster can run this. Architecture serving the business model (open-core, self-hostable) rather than resume-driven complexity.
6. **Client-side durability where it matters** — the widget's IndexedDB response queue with cross-instance locks (`response-queue.ts:36-53`): user data survives network failure even though the server-side pipeline is best-effort.

## Risks and tradeoffs (prioritized)

**R1 — At-most-once webhook/event delivery.** Confirmed by code: unawaited producer (`responses/route.ts:245`), catch-and-log (`pipelines.ts:22-24`), no retry/outbox/replay anywhere in the fan-out. Impact: customer-facing integrations silently miss events on any crash/restart/timeout. Likelihood: low per-event, certain at scale. This is the #1 architectural gap.

**R2 — Check-then-act counters.** Quota screening (`quotas/lib/utils.ts:135-176`) and autoComplete (`pipeline/route.ts:273-301`) count then decide without locks. Hypothesis (unproven by test): overshoot under concurrent submissions at READ COMMITTED. Impact: quota limits exceeded by small margins — for research-panel use cases, that's billable wrongness. Fix cost: low (conditional UPDATE or `FOR UPDATE`).

**R3 — Fail-open rate limiting incl. auth.** Confirmed: `rate-limit.ts:112-134` returns allowed on any Redis failure, including `auth:login`. Combined with 3s Redis connect timeout, a flapping Redis both un-throttles brute force *and* adds seconds to login. Fix: fail-closed for auth namespaces + a circuit breaker to skip known-down Redis quickly.

**R4 — TTL-only env-state cache with three stacked layers.** Confirmed absence of invalidation (verification log). Worst-case ~2min for new loads, 1h for live SDKs, to see a pause/publish. Probably a deliberate tradeoff; the risk is *nobody deciding it on purpose* per feature (e.g. "pause survey" arguably deserves active invalidation of the Redis layer — one `del` call).

**R5 — Version sprawl: v1/v2/v3 APIs sharing internals.** v2 imports v1 libs (`responses/lib/response.ts:13`); three wrapper stacks exist. Cost is ongoing comprehension + accidental-behavior-change risk, not a bug. Needs a written deprecation policy more than code.

**R6 — Unbounded fan-out concurrency.** 50 webhooks = 50 parallel pinned-dispatcher fetches inside one request (`pipeline/route.ts:119-177`). Bounded per-request by timeout, unbounded per-instance under response bursts. Hypothesis: socket/memory pressure at high volume.

**R7 — The pipeline route is a 348-line monolith** doing webhooks + integrations + emails + follow-ups + billing + analytics + survey state. Every new side effect lands here; testability degrades (its lib/ pieces are tested, the orchestration isn't). Refactor seam, not incident.

## "Owning this for 3 months" — the plan

Month 1 (make the invisible visible, zero behavior change): metrics/alerts on pipeline send failures, webhook delivery outcomes, rate-limiter fail-open activations, Redis health in /health (Ticket 17); write the API deprecation policy; concurrency cap (p-limit) on webhook fan-out (R6, tiny diff).
Month 2 (correctness): R2 conditional-update fix for quotas + a concurrency regression test (two parallel submits against limit=1); R3 fail-closed auth rate limiting behind a flag; R4 decision memo per cached entity — add the one `del` on survey pause/publish if product agrees.
Month 3 (the big one, as an RFC first): transactional outbox for pipeline events (R1) — event row inside the existing Flow-1 transaction, drainer worker (pg-boss on existing Postgres or BullMQ on existing Redis — self-hosting constraint favors pg-boss), keep `/api/pipeline` as the execution target, retries with backoff + dead-letter table + admin UI. Migration path: dual-write (send + outbox) behind a flag → verify parity in metrics → cut over → remove direct send. Test strategy: kill -9 during load and assert zero lost deliveries after drain; that test is the acceptance criterion.

Explicitly *not* doing: microservices, GraphQL rewrite, replacing Json survey documents (the bet is sound), swapping Prisma. Saying what you won't do is half of senior judgment.

## Confirmed vs hypothesis ledger

Confirmed by reading: R1, R3, R4 (absence), R5, R7, all strengths. Hypotheses needing a test before acting: R2 overshoot magnitude, R6 pressure threshold, cache-stampede behavior of `withCache` (does it single-flight? read `service.ts:241+` before claiming).

Interview use: pick R1 or R2, present as *finding → evidence → impact → fix → migration → test*. That six-beat structure is what "senior" sounds like in a system-design or code-review round.
