# System Design From This Repo

The whiteboard exercise: **"Design a survey platform like Typeform — teams create surveys, embed them in apps or share links, collect responses at scale, and trigger webhooks/integrations."** You have studied a production implementation; answer with evidence.

How to use this file: walk the six steps aloud with a timer (35–40 min total). At each step, first give *your* answer, then compare with what Formbricks chose (anchors), then note the stronger/simpler alternative. The junior/mid/senior contrast at each step tells you what to level up.

---

## Step 1 — Requirements (5 min)

Functional: create/edit surveys with rich question types and logic; publish as link or in-app widget; collect responses (partial + complete); view results; webhooks + integrations; multi-tenant orgs with teams.
Non-functional: response submission must be highly available and low-latency (it runs on *customer* pages); survey config reads are extremely hot; response writes are bursty; tenancy isolation is non-negotiable; self-hostable (constrains infra choices!).

- Junior answer: lists features only.
- Mid answer: separates read path (config) from write path (responses), names the multi-tenancy requirement.
- Senior answer: notices **self-hostability** shapes everything — Formbricks avoids managed queues/lambdas because customers must run it with just Postgres + Redis + S3 (see `docker-compose.dev.yml`, `charts/`). Constraints like this are why "just use SQS" is sometimes wrong.

## Step 2 — API sketch (5 min)

What Formbricks actually has (`apps/web/app/api/`):
- Public **client API**, unauthenticated, environment-scoped: `GET /api/v1/client/{environmentId}/environment` (widget config, Flow 3), `POST /api/v2/client/{environmentId}/responses` (Flow 1), `POST .../storage` (Flow 7 signed uploads), displays, user/contact endpoints.
- **Management API**, `x-api-key` authenticated with per-environment permissions (`apps/web/app/api/v1/auth.ts:16-53`), v1/v2/v3 versions, OpenAPI spec at `openapi.yml`.
- **Dashboard mutations** as Next.js server actions (not REST) — `apps/web/modules/survey/editor/actions.ts:250-332`.
- **Internal pipeline** route authenticated by shared secret (`apps/web/app/api/(internal)/pipeline/route.ts:31-36`).

Alternative: one GraphQL API for everything — simpler surface, but loses per-audience auth models and cache-header control on the hot public GET.

- Junior: designs one CRUD API for everything.
- Mid: splits public client API from authenticated management API; mentions versioning.
- Senior: also splits *dashboard* traffic (server actions, session auth) from *machine* traffic (API keys), and calls out that the public API's "auth" is possession of environmentId + survey state checks — then immediately asks what that exposes (see Step 6).

## Step 3 — Data model (7 min)

Formbricks' choice (`packages/database/schema.prisma`): tenancy chain `Organization → Project → Environment (production/development) → Survey → Response/Display`, with `Membership`/`Team`/`ProjectTeam` for access, `Contact` + `ContactAttribute` for known users, `Webhook`, `Integration`, `SurveyQuota`.

The big decision: **survey structure (blocks/questions/endings/logic) and response answers are Json columns** (`schema.prisma:357-370`, `:167-175`), typed by Zod schemas (Pattern 11) — not normalized `Question`/`Answer` tables.

Tradeoffs to say aloud:
- Json wins: survey structure evolves weekly (new question types) with no migrations; read/write whole-document matches the editor UX; one row per response is fast to insert.
- Json costs: no DB-level FK from an answer to its question; aggregation ("average rating for question X") must unpack Json (`data->>'q1'`); schema drift over old rows is an *application* problem.
- The invariants that DID need the DB got real columns/constraints: `@@unique([surveyId, singleUseId])` (`schema.prisma:186`), `displayId @unique`, indexes for count queries (`:187-189` — note the comment "to determine monthly response count").

- Junior: normalizes everything into questions/answers tables.
- Mid: chooses Json for flexibility and can name one cost.
- Senior: articulates the *rule*: relational where the DB must enforce invariants or aggregate; Json where the app owns the contract — and points at the schema doing exactly this split.

## Step 4 — The write path & tenancy (8 min)

Walk Flow 1 (see [01-codebase-cartography/05-key-flows.md](../01-codebase-cartography/05-key-flows.md)): layered validation → tenancy check (`survey.environmentId === environmentId`, `apps/web/app/api/v2/client/[environmentId]/responses/lib/utils.ts:18-20`) → one transaction for response + quota links (`.../lib/response.ts:24-41`) → post-commit events.

Authorization story for the dashboard: session → Zod input → `checkAuthorizationUpdated` (org-role OR project-team permission, `apps/web/lib/utils/action-client/action-client-middleware.ts:94-121`) → EE license gates → audit log.

Variation an interviewer will press: **"Two people submit the last quota slot simultaneously — what happens?"** Honest answer from the code: quota counting is count-then-act inside the transaction (`apps/web/modules/ee/quotas/lib/utils.ts:135-176`); under READ COMMITTED both can pass — possible overshoot by a few. Then give the fix menu: `FOR UPDATE` lock on the quota row, atomic conditional `UPDATE ... WHERE count < limit`, or serializable + retry — and note overshoot-by-one may be an acceptable business cost. *This exact exchange — knowing the race, the fixes, and that "accept it" is an option — is the mid→senior boundary.*

- Junior: "validate and insert."
- Mid: transaction, unique-constraint idempotency, tenancy check placement.
- Senior: the race analysis above, plus "the response must never be lost because a webhook failed" → side effects after commit.

## Step 5 — Async & integrations (8 min)

Formbricks' choice: no queue. `sendToPipeline` fire-and-forget HTTP self-call (`apps/web/app/lib/pipelines.ts:5-25`) → `/api/pipeline` fans out to webhooks (SSRF-defended, DNS-pinned, 5s timeout, `pipeline/route.ts:96-177`), integrations, notification + follow-up emails, autoComplete — all `Promise.allSettled`, all at-most-once.

Present both sides:
- Why it's defensible: self-hosters get zero extra infrastructure; the write path stays fast; webhook targets are bounded by timeout; failures are logged.
- What it can't do: guaranteed delivery, retries with backoff, replay, ordering, backpressure. A crash between commit and fan-out silently drops customer webhooks.
- The upgrade path (say it as a migration, not a rewrite): (1) write an `Event` outbox row inside the Flow-1 transaction; (2) a poller/worker (or pg-boss/BullMQ on the existing Redis) delivers with retries + dead-letter; (3) keep the HTTP route as the worker's execution target so the fan-out code doesn't move; (4) idempotency keys on delivery (the `webhook-id` header already exists! `pipeline/route.ts:137-143` — Standard Webhooks compliance means consumers can dedupe today).

Variation prompts: "now 10× traffic" (the pipeline route becomes the bottleneck — separate worker pool); "now guarantee webhook delivery" (outbox above); "now add real-time dashboards" (Postgres LISTEN/NOTIFY or Redis pub/sub → SSE/WebSocket — note nothing real-time exists today).

- Junior: "send the webhook after saving."
- Mid: names at-most-once vs at-least-once and proposes a queue.
- Senior: proposes the *incremental* outbox migration honoring the self-hosting constraint, and spots that signatures/message-ids already exist for consumer-side dedupe.

## Step 6 — Scale, caching, security (7 min)

Read path: three-layer cache (CDN headers + Redis 60s + SDK 1h `expiresAt`) on environment state (`environmentState.ts:22-70`, `environment/route.ts:60-75`), TTL-only. Rate limiting: atomic Redis Lua, fail-open, per-namespace budgets (`modules/core/rate-limit/`). Security highlights worth volunteering: constant-time login with control hash (`authOptions.ts:232-235`), SSRF triple defense (Pattern 9), 128-char password DoS cap, presigned S3 uploads so bytes never transit the app (`storage/route.ts:29-113`).

- Junior: "add Redis."
- Mid: names the layers with TTLs and the staleness consequence.
- Senior: distinguishes fail-open (rate limiter, cache) from fail-closed (tenancy checks) and can say *why each direction* was chosen per component; raises "what does the public env-state payload leak?" as an open review question.

---

## Variation prompts (practice each as a 10-min delta)

1. **Add multi-region.** Where does this design hurt? (Single Postgres; Redis cache keys are region-local; webhook egress IPs per region.)
2. **Add real-time response streaming to the dashboard.** (Pipeline route gains a pub/sub publish; SSE endpoint; note ordering and auth of the stream.)
3. **10× response traffic on one viral survey.** (Hot survey row reads → cache survey by id; response inserts are append-only and fine; quota counting becomes the contention point — now the Pattern-12 fix is mandatory, not optional.)
4. **Add GDPR deletion.** (Cascades exist — `onDelete: Cascade` on Response→Survey; but S3 files, webhook payloads already delivered, and integration copies are the hard part. Deletion is a *distributed* problem.)
5. **Make webhook delivery exactly-once.** (Trick prompt: you can't; best is at-least-once + consumer idempotency via the existing `webhook-id`. Saying "exactly-once delivery doesn't exist, only exactly-once *processing*" earns senior points.)

## Grading yourself

- Basic: covered all six steps without anchors; design is generic but coherent.
- Solid: every step included at least one "here's what a real implementation chose and why" with a correct anchor from memory.
- Strong: you volunteered a race condition, a delivery-semantics analysis, and a constraint-driven justification (self-hosting) unprompted — and your variation answers were migrations, not rewrites.

Cross-links: architecture critique in [03-architecture-and-patterns/06-architecture-critique.md](../03-architecture-and-patterns/06-architecture-critique.md) is the long-form version of steps 4–6. The debugging/review simulations in [05-debugging-and-code-review-rounds.md](05-debugging-and-code-review-rounds.md) reuse these flows.
