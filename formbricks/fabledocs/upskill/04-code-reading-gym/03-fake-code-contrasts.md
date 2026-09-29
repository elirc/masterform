# Fake-Code Contrasts

Ten bad-vs-better pairs. All snippets are fake (labeled); each maps to a real pattern in this repo. Read the bad one first and articulate the smell *before* reading the fix.

## Contrast 1: Check-then-insert vs constraint

```ts
// Illustrative fake code: not from this repo
const existing = await prisma.response.findFirst({ where: { surveyId, singleUseId } });
if (existing) return conflict();
await prisma.response.create({ data: {...} }); // two requests race past the check
```
Better: let the DB own it — `@@unique([surveyId, singleUseId])` (`schema.prisma:186`) and catch the violation (`responses/route.ts:180-182`). Smell name for review comments: TOCTOU / check-then-act.

## Contrast 2: Missing tenancy filter

```ts
// Illustrative fake code: not from this repo
export async function getSurveyForClient(surveyId: string) {
  return prisma.survey.findUnique({ where: { id: surveyId } }); // any tenant's survey!
}
```
Better: scope to the caller's environment and verify — `checkSurveyValidity`'s first check (`utils.ts:18-20`). The fetch may be by id, but a comparison against the authenticated/route scope must follow before use.

## Contrast 3: Awaiting side effects inside the request (or transaction)

```ts
// Illustrative fake code: not from this repo
await prisma.$transaction(async (tx) => {
  const r = await tx.response.create({...});
  await fetch(webhook.url, {...}); // network I/O holding a DB connection + rollback loses nothing? wrong both ways
});
```
Better: commit first, side effects after (`responses/route.ts:234-259`); network calls never inside transactions. And know the repo's *next* flaw: those post-commit effects aren't durable either (Pattern 3) — the fix ladder is: out of tx → outbox.

## Contrast 4: Swallowed error, no context

```ts
// Illustrative fake code: not from this repo
} catch (e) { console.log("webhook failed"); }
```
Better: structured context + classification — `logger.error({ error, url }, \`Webhook call to ${webhook.url} failed\`)` (`pipeline/route.ts:174-176`); and 5xx-vs-4xx routing to Sentry via the wrapper contract (`storage/route.ts:98-105`). A log you can't query by surveyId is barely a log.

## Contrast 5: Coupling UI shape to DB shape

```ts
// Illustrative fake code: not from this repo
const survey = await prisma.survey.findUnique({ where: { id }, include: { responses: true } });
return <Editor survey={survey} />; // raw ORM row (with every response!) into a client component
```
Better: explicit selection shaped for the consumer — `responseSelection` (`api/v1/.../responses/lib/response.ts`), the environment-state payload building exactly what the SDK needs (`environmentState.ts:59-64`). DTOs are a boundary; ORM rows are not.

## Contrast 6: N+1 permission checks

```ts
// Illustrative fake code: not from this repo
for (const survey of surveys) {
  if (await canUserSeeSurvey(user, survey.id)) results.push(survey); // one DB hop per row
}
```
Better: resolve the grant once, filter in the query — `getMembershipRole` once per action (`action-client-middleware.ts:103`), and list endpoints filtering by environment scope in the WHERE clause. Authorization belongs in the query plan for list operations.

## Contrast 7: Ad-hoc cache keys and stringly TTLs

```ts
// Illustrative fake code: not from this repo
await redis.set("env_" + id, JSON.stringify(data)); // no TTL, no namespace, one typo from a collision
```
Better: typed registry + explicit TTL — `createCacheKey.environment.state(id)` with branded `CacheKey` (`cache-keys.ts:17-23`) and `withCache(fn, key, 60_000)` (`environmentState.ts:68-70`).

## Contrast 8: Stale-closure interval in React

```ts
// Illustrative fake code: not from this repo
useEffect(() => {
  const t = setInterval(() => save(localSurvey), 5000); // captures the FIRST localSurvey forever
  return () => clearInterval(t);
}, []); // lying deps
```
Better: depend on the value, or read through a ref updated each render. The repo's convention (AGENTS.md) mandates snapshot-refs in cleanup for the sibling bug. Also note the editor avoids the whole class by saving explicitly through user action (Flow 4) instead of ambient timers.

## Contrast 9: `any` at the error boundary

```ts
// Illustrative fake code: not from this repo
catch (error: any) { return res.status(500).json({ msg: error.message }); } // leaks internals, types nothing
```
Better: `unknown` + typed classes/guards mapped deliberately — `error instanceof UniqueConstraintError → 409` (`responses/route.ts:175-191`), generic public message for the rest (`getUnexpectedPublicErrorResponse`, `:43-44`). Error *messages* are API contract; don't let exceptions author them.

## Contrast 10: Casual public-contract change

```ts
// Illustrative fake code: not from this repo
// "cleanup": rename a response field the API returns
return { data: { responseId: response.id } }; // was `id` — every SDK breaks
```
Better: the repo's own discipline — versioned APIs (v1/v2/v3 side by side), additive changes within a version, `openapi.yml` as the published contract, and the widget consuming its own public API so breaks surface in-house first. Before renaming anything a client sees, find every consumer — including ones you don't control (that's what "public" means).

---

Drill: for each contrast, write the one-sentence review comment you'd leave on the *bad* version — specific, kind, and pointing at the repo's own precedent. Example for #1: "This check races with concurrent submits; we already enforce this invariant with a unique constraint elsewhere (schema.prisma:186) — can we catch P2002 instead?" Collect all ten; they're your review-round vocabulary.
