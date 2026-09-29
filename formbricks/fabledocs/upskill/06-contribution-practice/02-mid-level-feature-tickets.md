# Mid-Level Feature Tickets

Twelve cross-layer tickets (schema/API/UI/tests). **Design note required before code** for each: half a page — approach, alternatives rejected, risk, rollback. That artifact is half the learning value and a direct interview exhibit.

## Ticket M1: Invalidate the env-state cache on survey pause/publish
Layers: service + cache. ~1 day.
Story: as a survey owner, pausing a survey should stop new widget displays within seconds, not minutes.
Anchors: `environmentState.ts:68-70` (the key), `modules/survey/editor/lib/survey.ts` (`updateSurvey`), cache deletion API in `packages/cache/src/service.ts`.
Design questions to answer first: only on status transitions, or every update? What about the CDN layer you *can't* purge (accept 60s)? Does the pipeline's `updateSurvey` for autoComplete (`pipeline/route.ts:277`) go through the same choke point (it must)?
Risk: cache-deletion failure handling (best-effort + log, or fail the save?). Rollback: remove the del call.
Interview story potential: "I converted a TTL-only cache to targeted invalidation and defended the remaining staleness budget."

## Ticket M2: `Retry-After` + standard 429 body across all three API wrappers
Layers: API framework. ~1-2 days. (Extends junior Ticket 4 to the full surface.)
Anchors: `with-api-logging.ts`, `modules/api/v2/auth/api-wrapper.ts`, v3's `api-wrapper.ts`.
Design note must include: a table of the three wrappers' current 429 shapes (discover by reading — they likely differ; that finding *is* the ticket's value).
Interview story: "I unified error contracts across API versions without breaking clients."

## Ticket M3: Webhook delivery counter metrics
Layers: pipeline + observability. ~2 days.
Anchors: `pipeline/route.ts:119-177`, `instrumentation-node.ts` (what metrics infra exists — investigate first).
Acceptance: `webhook_delivery_total{outcome=success|timeout|ssrf_rejected|http_error}` incremented per attempt; zero behavior change.
Interview story: "I made an invisible reliability gap measurable before proposing the fix" — the correct *order* (measure → propose) is the story.

## Ticket M4: Concurrency cap on webhook fan-out
Layers: pipeline. ~1 day. Fixes R6.
Anchors: `pipeline/route.ts:119-177`.
Design: p-limit(10) (or hand-rolled pool) around webhook promises; justify the number from measurement (M3 first!); ensure `allSettled` semantics preserved.
Risk: a slow-webhook tenant now delays their *own* later webhooks — document that as intended isolation.
Interview story: "bounded an unbounded fan-out; chose the limit from data."

## Ticket M5: Per-survey response export hardening
Layers: API + streaming. ~2-3 days.
Anchors: find the CSV/export path (grep `export` under `modules/analysis` / response service) — verify how it handles 100k responses (memory-bound?).
Design: pagination/streaming for large exports; content-disposition; authz check documented.
Risk: changing an endpoint users script against.
Interview story: "made an export scale from memory-bound to streaming."

## Ticket M6: Quota overshoot regression test + conditional-update fix
Layers: DB + service. ~2-3 days. Fixes R2 (get maintainer buy-in first — this changes EE behavior).
Anchors: `quotas/lib/utils.ts:135-176`, `responses/lib/response.ts:24-41`.
Design note must contain: the interleaving diagram; chosen fix (atomic `UPDATE ... WHERE` counter vs `FOR UPDATE` on quota row) with lock-contention analysis; the concurrent test (two parallel transactions, limit=1, exactly one screened in).
Interview story: the best one in this repo — "found, proved, and fixed a check-then-act race."

## Ticket M7: Survey editor conflict warning (multi-tab)
Layers: UI + action. ~3 days.
Anchors: `survey-editor.tsx:88` (localSurvey clone), `updateSurveyAction` (`editor/actions.ts:250-332`), `Survey.updatedAt`.
Design: optimistic-concurrency lite — send the loaded `updatedAt`; server rejects if DB is newer; UI offers reload-or-overwrite. NOT full CRDT/merge — write down why (cost/benefit).
Risk: false conflicts from autosaves (does the editor autosave? verify `auto-save-indicator.tsx` semantics first).
Interview story: "added last-write-wins protection with version checks — and argued why not real-time merge."

## Ticket M8: Client-error beacon for the widget
Layers: SDK + new public endpoint. ~3-4 days. (Observability table's hardest gap.)
Design: tiny POST endpoint (rate-limited! new namespace in `rate-limit-configs.ts`) receiving `{environmentId, error, context}` from js-core's error paths; sampling; PII scrubbing; dashboard nowhere yet (logs first).
Risk: you're building an abuse target — the design note's security section is the point of the exercise (apply the pre-merge checklist).
Interview story: "designed telemetry for code running on pages we don't control."

## Ticket M9: Follow-up referential integrity in the editor
Layers: editor validation. ~2 days. (Closes debugging scenario 5's root cause.)
Anchors: `follow-ups.ts:56-62` (runtime failure), editor's follow-up config components (`modules/survey/editor/components/`), `ZSurvey` refinements.
Design: on save, validate every followUp `to`/`replyTo` references an existing element id, hidden field, or literal email — a Zod `superRefine` on ZSurvey or a check in `updateSurvey`. Decide: block save vs warn.
Interview story: "closed a dangling-reference class across two Json documents with schema-level validation."

## Ticket M10: API-key last-used tracking
Layers: schema + API. ~2 days.
Anchors: `schema.prisma:773-805` (ApiKey), `getApiKeyWithPermissions`.
Design: `lastUsedAt` column; update *throttled* (once per hour, not per request — write the why: hot-path write amplification); surface in the keys settings UI.
Migration: additive nullable — the textbook safe change; write the rollback line anyway.
Interview story: "schema+API+UI feature with a write-amplification tradeoff."

## Ticket M11: Rate-limit budget dashboard doc + fail-closed auth flag
Layers: core + config. ~2 days. Part of R3.
Design: `RATE_LIMIT_FAIL_CLOSED_NAMESPACES=auth` env (validated in `env.ts`); limiter consults it in the catch path (`rate-limit.ts:112-134`); document the availability tradeoff (Redis outage now blocks logins — that's the *point*, and the flag keeps it opt-in).
Interview story: "made a fail-open/fail-closed decision configurable, per threat model."

## Ticket M12: Unify duplicated org/project id resolution
Layers: services. ~1-2 days.
Anchors: `editor/actions.ts:252,263,292` (repeated resolutions), `lib/utils/helper.ts` (the resolvers — check for React `cache()` wrapping first!).
Design: if not request-deduped, wrap in `cache()` per AGENTS.md's rule; if already deduped, this ticket becomes a measurement writeup proving it (also valuable).
Interview story: "verified before optimizing — and the verification was the deliverable."

---

Sequencing advice: M3 → M4 → (M6 or M1) is a coherent reliability arc that produces a portfolio narrative: *measure, bound, fix*. Do them in that order and you have a senior-flavored story spanning three PRs.
