# Performance Thinking

Rule zero: **measure first**. Every claim below is a *hypothesis with an anchor* — the drill is turning three of them into numbers on your machine (seeded DB, `EXPLAIN ANALYZE`, browser profiler), not nodding along.

## The performance domains of this system

| Domain | Hot spots here | How to measure |
| --- | --- | --- |
| Widget bundle | `packages/surveys` UMD served to every customer page | `ls -l apps/web/public/js/`; bundle analyzer on the vite build |
| Client API latency | response POST, environment GET | local k6/autocannon; the request logs from `with-api-logging` |
| Server/DB | pipeline queries, response lists, quota counts | Prisma query logging + `EXPLAIN ANALYZE` |
| Caching | hit ratios, stampedes | Redis MONITOR locally; log sampling |
| Dashboard render | survey editor with large surveys | React DevTools profiler |
| Email/integrations | third-party latency inside pipeline | timing logs around `handleIntegrations` |

## Likely hotspots, ranked by (impact × confidence)

1. **The notification-recipient query** (`pipeline/route.ts:192-243`): five nested relation filters + a Json-path condition, per finished response. The TODO at :191 admits the caching gap. Measure: EXPLAIN with realistic membership fan-out. Fix ladder: cache membership→env mapping; precompute recipients on survey save; or move to a materialized join table.
2. **EE license checks on the hot public path** (`responses/route.ts:82-96` + `getIsSpamProtectionEnabled` etc.): each adds cache/DB hops to every response POST. Measure: time the POST with/without contactId. Mitigation exists (license caching, `createCacheKey.license.*`) — verify its TTL actually covers the calls made here.
3. **Quota `groupBy` count on every response in a quota'd survey** (`quotas/lib/utils.ts:135-151`): fine at 1k responses; check the plan at 1M. The `ResponseQuotaLink` PK/indexes (`schema.prisma:453-471`) — read them and predict whether the groupBy is index-only.
4. **Environment-state payload size** (`environmentState.ts:59-64`): every in-progress survey + all actionClasses ships to every widget. A tenant with 200 surveys pays it on each cold load. Measure: `curl | wc -c` on a seeded env. Fix: field trimming (check what `toJsEnvironmentStateSurvey`-style transforms already drop) before pagination.
5. **Interactive transaction duration under load** (Flow 1): the quota evaluation extends transaction lifetime → connection-pool pressure. Measure: pgbouncer/pool stats during a k6 burst. This is the *systemic* cost of R2's eventual fix too — locks lengthen the same critical section; say that tradeoff out loud in interviews.
6. **Webhook fan-out concurrency** (R6): N parallel fetches with pinned dispatchers per event. Measure: memory/socket counts with a 50-webhook fixture.

## Finding the classics, concretely

- **N+1**: grep for `await` inside `for` loops over query results (`grep -rn "for (const" apps/web/modules --include="*.ts" -A3 | grep await` is a crude but effective first pass). Prisma's `include` mostly protects reads; loops over *service calls* (each hiding a query) are where it hides — audit `handleIntegrations` per-integration handling.
- **Serial-that-could-be-parallel**: adjacent independent `await`s. Counter-example done right: `storage/route.ts:59-62`. Audit `updateSurveyAction`'s three sequential `getOrganizationIdFrom*`/`getProjectIdFrom*` resolutions (`editor/actions.ts:252,263,292`) — same lookups, sequential, possibly duplicated. Measure before submitting the "fix": they may be React-`cache()`-deduped (check `apps/web/lib/utils/helper.ts` for a `cache()` wrapper — the difference decides whether there's a PR here at all).
- **Expensive renders**: the editor re-rendering the whole block list per keystroke — profile before believing; `structuredClone` of a large survey per state init (`survey-editor.tsx:88`) is once-per-mount, fine.
- **Oversized bundles**: the *dashboard* can afford weight; the *widget* cannot. Any PR adding a dependency to `packages/surveys`/`js-core` needs a size diff in its description (house norm worth importing even if not enforced).
- **Missing indexes / unbounded queries**: schema indexes are commented (`schema.prisma:187-189`) — for any new list query, either it rides an existing index or the PR adds one with EXPLAIN evidence (Ticket 8's discipline).

## Interview angle

"How would you find and fix a slow endpoint?" — narrate: reproduce with a seeded dataset → measure (logs/EXPLAIN, not vibes) → rank by user impact → fix the query/cache, not the symptom → guard with a perf assertion or at least a dashboard. Then give hotspot 1 above as your worked example, *including* the TODO comment as evidence that production code accretes known debt — that detail lands because it's true of every codebase the interviewer has owned.

Drill: seed a local DB (find/write a seed script — `pnpm db:seed` exists at root, __inferred__ `package.json:20-21`), run `EXPLAIN ANALYZE` on hotspot 3's groupBy at 10k links, and write the three-sentence verdict: plan used, cost, fix-or-fine.
Self-grade — Basic: got a plan output. Solid: interpreted it correctly (seq scan vs index-only). Strong: your verdict includes the volume threshold where the answer changes.
