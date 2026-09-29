# Type System and Contracts

## The architecture of types here: Zod-first

The contract source of truth is **Zod schemas in `packages/types`** (plus `packages/database/zod/` for DB Json shapes). TS types are *derived* (`z.infer`), so runtime validation and compile-time types can't drift. This inverts the naive approach (write interfaces, hope the data matches).

The full chain for one field: survey blocks are Json in Postgres (`schema.prisma:361` with the `/// [SurveyBlocks]` json-types annotation) → typed via the Prisma json-types generator (`packages/database/json-types.ts`) → validated on write by `ZSurvey` (`editor/actions.ts:250`) → consumed as `TSurvey` throughout. The DB stores bytes; **Zod owns the contract**. That's Pattern 11; this file is about the TS mechanics that make it work.

## Discriminated unions and narrowing (the workhorse)

- Result types: `packages/cache/src/client.ts:11-58` — `Result<RedisClient, CacheError>`; `.ok` is the discriminant; consumers narrow with `if (!result.ok)`.
- Structural discrimination without a tag: `TValidatedResponseInputResult` (`responses/route.ts:30-35`) — arms distinguished by *property presence*, narrowed with `"response" in validatedInput` (:208). Know both forms: tagged (`.ok`) and structural (`in` operator); interviewers ask for the second one less often, which makes knowing it valuable.
- Error classes as discriminants: `error instanceof InvalidInputError` (:176-182) — runtime class checks doing the narrowing exceptions require. Note the asymmetry: unions are compiler-enforced (miss a case → type error with `never` checks), instanceof chains are vibes-enforced (miss a case → falls to the generic 500).

## Generics doing real work

- `TAccess<T extends z.ZodRawShape>` (`action-client-middleware.ts:21-37`) — an access-grant union generic over an optional validation schema; the schema's `_output` types the `data` field. Read it slowly once; it's a compact lesson in generic constraints + indexed access types.
- `withCache<T>(fn: () => Promise<T>, ...): Promise<T>` (`packages/cache/src/service.ts:241`) — the classic transparent-wrapper generic: whatever the fn returns, the cache returns. The un-typed hole: what's *in* Redis is `JSON.parse`d — the `T` is a promise (pun intended), not a proof. If the cached shape drifts across deploys, TS won't save you; that's a senior observation about the limits of compile-time contracts at serialization boundaries.
- Branded types: `CacheKey` (`packages/cache/src/cache-keys.ts` via `makeCacheKey`) — a string the compiler refuses to conflate with ordinary strings. Zero runtime cost, kills a whole bug class (raw keys). Also `ZEnvironmentId` (cuid-validated ids) — ids validated at boundaries rather than typed as bare `string` everywhere after.

## `unknown` vs `any` discipline

House style is imperfect and instructive: `handleErrorResponse = (error: any)` (`api/v1/auth.ts:55`) vs the `catch (error)` + instanceof style in newer code. Ticket 11 in [06-contribution-practice/01-good-first-tickets.md](../06-contribution-practice/01-good-first-tickets.md) is the hardening exercise. Rule to carry: `any` disables checking *transitively* (everything it touches degrades); `unknown` forces narrowing at first use. In catch clauses, `unknown` + type guards is strictly better.

Also note `@ts-expect-error` used honestly for legacy-API shims (`packages/js-core/src/index.ts:17-23`) — each has a reason comment, and it *fails the build if the error disappears* (unlike `@ts-ignore`). That distinction is a favorite interview nugget.

## satisfies, const-assertions, and config typing

`rateLimitConfigs = {...} as const` (`rate-limit-configs.ts:36`) — literal types preserved so `rateLimitConfigs.auth.login.namespace` is the literal `"auth:login"`, not `string`. Where you'd reach for `satisfies`: keeping the literal inference *and* checking the object against a `Record<string, TRateLimitConfig>` shape. Good refactor drill (on paper — don't PR it without cause).

## Pitfall checklist

- [ ] A Zod schema exists but is only used on one of two entry paths (v1 vs v2 API) — check both parse the same shape.
- [ ] `z.infer` type imported but data actually came from `JSON.parse` (cache, pipeline) — the serialization hole.
- [ ] Composite unique with nullable column (`schema.prisma:186`) — TS can't see that NULLs don't collide; the *DB* semantics matter.
- [ ] Weight-table objects (`action-client-middleware.ts:39-48`) keyed by types that could drift from the enum — would a new team role break this silently? Check what constrains them.

## Interview angle

1. "How do you keep API types in sync between client and server?" → Zod-first + z.infer + shared `packages/types`; contrast with codegen (OpenAPI — this repo *also* has `openapi.yml` for external consumers; know both live here).
2. "Discriminated unions vs exceptions" → [08-interview-prep/01](../08-interview-prep/01-js-ts-node-deep-dive.md) Q4/Q8.
3. "What can't the type system protect?" → serialization boundaries (cache/pipeline), DB Json drift — with the withCache example.
4. "Branded/nominal types in TS?" → `CacheKey`; explain the zero-cost trick in 30 seconds.

Drill: take `TValidatedResponseInputResult` and rewrite it (scratch file) as a tagged union with `kind: "ok" | "response"`. Which call sites get simpler? Which get noisier? That tradeoff — structural convenience vs tag explicitness — is the whole discriminated-union interview question in miniature.
Self-grade — Basic: rewrite compiles in your head. Solid: you can name one narrowing that gets better and one that gets worse. Strong: you can state when you'd enforce tags codebase-wide (answer: when arms start sharing properties).
