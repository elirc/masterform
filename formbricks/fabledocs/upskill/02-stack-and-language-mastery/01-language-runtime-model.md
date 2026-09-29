# Language & Runtime Model

## The event loop, as this repo uses it

Model: one JS thread per process; synchronous code runs to completion; `await` yields the thread; microtasks (promise callbacks) drain before macrotasks (timers, I/O callbacks).

Where the repo leans on this:
- **Deliberate macrotask deferral**: `packages/js-core/src/index.ts:42-48` — `setTimeout(0)` after setup so synchronously-queued user commands order before page-view processing. The comment documents the ordering dependency; that documentation is what makes it acceptable.
- **Unawaited work**: `apps/web/app/api/v2/client/[environmentId]/responses/route.ts:245-259` — `sendToPipeline` returns a promise nobody awaits. The handler finishes; the fetch continues in the background *if the runtime lets it live*. On long-running Node servers: fine. On serverless: roulette. Failure containment is the trailing `.catch` in `app/lib/pipelines.ts:22-24` — without it, an unhandled rejection.
- **Parallel vs serial**: `apps/web/app/api/v1/client/[environmentId]/storage/route.ts:59-62` runs independent reads with `Promise.all`; `apps/web/app/api/(internal)/pipeline/route.ts:304` uses `allSettled` for independent side effects. Rule: fail-fast for dependent reads, settle-all for independent effects, and name your concurrency cap when N is user-controlled (webhook count is per-customer — bounded how? Worth checking: it isn't, beyond the 5s timeout each).

Pitfall checklist (test yourself against the anchors):
- [ ] Can a promise rejection escape to crash the process here? (Find the `.catch` or `allSettled` that prevents it.)
- [ ] Does anything block the loop >10ms? (bcrypt in `authorize` is the honest answer — CPU work on the request path, capped by the 128-char rule, `authOptions.ts:203-216`.)
- [ ] Who cleans up on timeout? (`pipeline/route.ts:169-172` — `finally { dispatcher?.destroy() }`; the comment explains why `destroy` not `close`.)

## Node vs browser vs "browser on someone else's page"

- Node-only markers: `import "server-only"` (`environmentState.ts:1`), `next/headers`, Prisma, `process.env`.
- Dashboard browser: React 19, `"use client"`, TanStack Query.
- **js-core/surveys run on customer pages** — the harshest runtime: unknown CSP, other libraries' prototypes, ad blockers, private-browsing IndexedDB failures. That's why: module-level lock Maps instead of framework state (`response-queue.ts:36-53`), a hand-rolled CommandQueue instead of a framework (`command-queue.ts:25-50`), Preact instead of React (bundle size), and offline persistence (`offline-storage.ts`).

## Long-lived process concerns

- **Singletons across hot reload**: `packages/cache/src/client.ts:62-80` stashes the CacheService on `globalThis` — in dev, Next.js re-evaluates modules on every edit; module-level singletons would leak connections. `globalThis` survives. Same trick Prisma clients use everywhere; know it by name.
- **Connection lifecycle**: the Redis client self-destroys and resets the factory on error events (`client.ts:29-38`) — reconnection by recreation, not by nursing a broken socket.
- **Pool starvation**: interactive Prisma transactions (`responses/lib/response.ts:24-41`) hold a connection for their whole callback. Keep network calls out of transactions — this repo does (pipeline fires after commit).

## Interview angle

1. "Why not await the analytics/webhook call?" → Q1 in [08-interview-prep/01](../08-interview-prep/01-js-ts-node-deep-dive.md); answer with the pipeline example + serverless caveat.
2. "Promise.all vs allSettled" → Q2; answer with webhooks-vs-reads.
3. "How do singletons survive hot reload in Next dev?" → the `globalThis` pattern, `cache/src/client.ts:62-80`.
4. "Where's the CPU bottleneck in a typical Node app?" → password hashing; explain the 128-char cap as DoS defense.

Drill: pick any route file and label every `await` as (a) dependent read, (b) independent read that could parallelize, or (c) side effect that could defer. `pipeline/route.ts:179-344` has all three within 150 lines — do that one, then check whether your (b) findings are real wins or premature.
Self-grade — Basic: labels correct. Solid: found ≥1 real parallelization already done and why. Strong: found the one *deliberate* serialization (integrations before emails? read the ordering) and can defend or challenge it.
