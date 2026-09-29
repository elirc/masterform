# Side Effects, Async, and Reliability

## Inventory of side effects (where the outside world gets touched)

| Side effect | Trigger | Code | Delivery semantics |
| --- | --- | --- | --- |
| Customer webhooks | response created/finished | `pipeline/route.ts:119-177` | at-most-once, 5s timeout, no retry |
| Integrations (Sheets/Airtable/Notion/Slack) | responseFinished | `pipeline/lib/handleIntegrations.ts` | at-most-once |
| Notification emails | responseFinished | `pipeline/route.ts:256-270` | at-most-once, per-recipient catch |
| Follow-up emails | responseFinished | `modules/survey/follow-ups/lib/follow-ups.ts` | at-most-once, rate-limited 50/hr |
| Survey autoComplete close | responseFinished | `pipeline/route.ts:273-301` | check-then-act on count |
| Stripe metering | responseCreated | `pipeline/route.ts:320-326` | best-effort, caught |
| PostHog/telemetry | many | `capturePostHogEvent`, `sendTelemetryEvents` | best-effort |
| Email (auth flows) | signup/reset/invite | `modules/email`, `packages/email` | direct send |
| S3 writes | uploads | presigned, browser-direct | S3's semantics |
| Cache writes | reads (fill), TTL (expiry) | `withCache` | eventually consistent |

Everything async funnels through one door: `sendToPipeline` → `/api/pipeline`. There is **no queue, no cron scheduler in-process, no worker fleet** — reliability is exactly as good as one HTTP request's lifetime. (Check `docker/` and `charts/` for a cron container calling cron endpoints before assuming *nothing* scheduled exists — investigate.)

## The concepts, graded against this repo

- **Idempotency**: producing the same result when applied twice. Present: single-use responses (DB unique). Present at the *protocol* level for webhooks: `webhook-id` uuidv7 + signature headers (`pipeline/route.ts:136-154`, Standard Webhooks) let consumers dedupe — but since delivery never retries, the dedupe key currently protects nobody. If retries are ever added, this groundwork makes them safe. That's forward-compatible design worth naming.
- **Retries**: absent server-side; present client-side (`ResponseQueue` retryAttempts + offline persistence, `packages/surveys/src/lib/response-queue.ts`). So the system is at-least-once from browser→server (with DB dedupe for single-use), at-most-once server→world. Asymmetry driven by who owns the data loss: a lost response is unacceptable; a lost webhook is (deemed) tolerable.
- **Outbox**: absent. The upgrade path is detailed in [08-interview-prep/04](../08-interview-prep/04-system-design-from-this-repo.md) step 5 and senior project 2.
- **Compensation**: n/a — no multi-service sagas; the transaction (Flow 1) keeps the only multi-write atomic.
- **Backpressure**: none on webhook fan-out beyond per-request 5s timeouts; a customer with 50 webhooks makes 50 parallel fetches. Timeout-bounded, concurrency-unbounded — a review comment waiting to happen.
- **Timeouts**: exemplary on webhooks (`AbortSignal.timeout(5000)` with the comment explaining why not Promise.race — `:102-115`); Redis connect 3s (`cache/src/client.ts:24-26`). Audit question for everything else: what's the timeout on integration API calls? (Read `handleIntegrations` and its per-service clients — if none, that's a finding.)
- **Failure visibility**: every swallow site logs with context (`pipelines.ts:22-24`, `pipeline/route.ts:174-176,305-317`). Logs-only means detection depends on someone reading logs — see [05-quality-engineering/06-observability-and-operations.md](../05-quality-engineering/06-observability-and-operations.md).

## Side effects in risky places (flag list)

1. Unawaited `sendToPipeline` at `responses/route.ts:245-259` — the crash window between commit and fan-out.
2. The write inside `withCache` (`environmentState.ts:29-56`) — cache-conditional execution.
3. Backup-code consumption during login (`authOptions.ts:298-306`) — a write on the auth read path; correct (one-time codes must burn) but must never move above the constant-time verification.
4. `checkSurveyValidity` mutating its input (`utils.ts:33`) — not a side *effect* on the world, but effect-shaped for the caller.
5. autoComplete survey-close (`pipeline/route.ts:273-301`) — a *product-visible* state change decided by a racy count, executed in the fan-out path where duplicate `responseFinished` events (client retries!) could double-fire it. Mostly harmless here (idempotent target state), which is exactly the analysis to practice: race + idempotent effect = tolerable.

## The reliability story to tell in interviews

"The system is at-least-once from the browser (durable client queue), atomic at the write (one transaction), and at-most-once beyond it (fire-and-forget pipeline). The DB constraint dedupes client retries; nothing dedupes or retries outbound webhooks — but the Standard-Webhooks headers already carry a message id, so adding retries later won't break consumers. If I owned this, the first reliability investment is an outbox table written inside the existing transaction, drained by a worker with backoff — the fan-out code (`pipeline/route.ts`) barely moves."

Four sentences, five anchors, one upgrade path — that's a senior answer. Rehearse it.

Drill: for each row of the inventory table, write what "we'd know it broke" looks like today (log line? Sentry? nothing?). Three rows will come up empty or logs-only — those are your observability tickets.
Self-grade — Basic: table filled. Solid: the empty rows found. Strong: you proposed the *cheapest* detection (a metric/alert, not a rewrite) per empty row.
