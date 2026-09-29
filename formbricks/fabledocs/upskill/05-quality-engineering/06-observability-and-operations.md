# Observability and Operations

The question this file trains: **"How would I know this broke?"** — asked per flow, answered with what exists, graded honestly.

## What exists

- **Structured logging**: `@formbricks/logger` (pino-flavored), context-object-first convention (`logger.error({error, url, surveyId}, "msg")`). Withcontext chaining (`action-client/index.ts:30`).
- **Error tracking**: Sentry per runtime (`sentry.server.config.ts`, `sentry.edge.config.ts`, `instrumentation*.ts`); deliberate classification — expected errors return messages, unexpected go to Sentry (`action-client/index.ts:18-31`; wrapper 5xx-vs-4xx split `storage/route.ts:98-105`); breadcrumbs for rate-limit violations (`rate-limit.ts:101-108`).
- **Metrics**: `apps/web/prometheus.yml` exists + OpenTelemetry-ish instrumentation files — open `instrumentation-node.ts` to see exactly what's exported before claiming metrics coverage (investigate).
- **Health**: `/health` route (`apps/web/app/health`, `api/v2/health`) — what does it actually check? (Ticket 17 found it doesn't cover Redis; verify current state.)
- **Product analytics as ops signal**: PostHog events (survey_published, app_connected) can double as "is the funnel alive" canaries.
- **Audit logs** (EE): who-did-what for mutations.

## Flow-by-flow: how would you know?

| Flow | Failure | Today's signal | Gap |
| --- | --- | --- | --- |
| Response submission | 5xx spike | `reportApiError` → Sentry (`responses/route.ts:184-190`) + request logs | Good |
| Response submission | elevated 4xx (validation regression after deploy) | logs only — 4xx deliberately not Sentry'd | No rate-based alert; a bad deploy rejecting 30% of responses looks "healthy" |
| Pipeline producer | self-call fails | `"Error sending event to pipeline"` log (`pipelines.ts:23`) | Log-only; no metric, no alert — top gap (matches R1) |
| Webhook delivery | customer endpoint failing | per-webhook error log (`pipeline/route.ts:174-176`) | Nothing customer-visible; no delivery dashboard; support finds out via ticket |
| Emails | send failures | per-recipient catch + log (`route.ts:264-269`) | Same |
| Rate limiter | fail-open activated | error log + Sentry capture (`rate-limit.ts:112-127`) | Signal exists — is anyone alerting on it? Operational, not code |
| Cache | Redis down | client event logs (`cache/src/client.ts:29-46`) + latency symptoms | /health doesn't say; latency alarm would fire first (debug scenario 4) |
| Widget (customer side) | bundle broken/blocked | *nothing server-side* — errors happen on pages you don't own | Genuinely hard; SDK debug mode (`getIsDebug`) helps humans, not monitoring. A client-error beacon endpoint is a legit senior project |
| Migrations/deploy | failed migration | CI/deploy logs; `DataMigration` table state | Standard |

## Operational surfaces

- Deploy: Docker images (multiple `.github/workflows/*docker*`), Helm chart (`charts/`), Formbricks Cloud pipeline (`deploy-formbricks-cloud.yml`). Rollback = redeploy previous image + migration reversibility discipline (additive-first, [03-architecture/02](../03-architecture-and-patterns/02-data-model-and-persistence.md)).
- Config as blast radius: `RATE_LIMITING_DISABLED`, `DANGEROUSLY_ALLOW_WEBHOOK_INTERNAL_URLS` — env flags that disable defenses. An ops-review question: who can set these in production, and would anyone notice? (The loud names are the *only* alarm.)

## The habit

For any PR you write here, add one sentence to the description: "If this breaks in production, we'll know because ___." If the blank is "a user tells us," either add the log/metric or say why it's acceptable. This single habit — installed now, on a training repo — is the most senior-sounding thing a mid-level candidate does in system-design follow-ups.

Interview angle: "How do you monitor a webhook system?" — answer with the table's webhook row: what exists (per-delivery logs), what's missing (delivery metrics, customer-visible status, alerting), and the incremental fix (counter metric + dead-letter visibility). Grounded gaps beat imaginary Grafana.

Drill: pick the "elevated 4xx" gap. Write (paper) the exact metric name, labels, and alert condition you'd add, and identify the single file where the counter increments (`with-api-logging.ts` — one place, every route: that's why the wrapper pattern pays).
