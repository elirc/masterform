# Data Model and Persistence

## The entity map (36 models, one chain that matters)

```
Organization ─┬─ Membership ─ User          Team ─ TeamUser ─ User
              ├─ OrganizationBilling        Team ─ ProjectTeam ─ Project
              └─ Project ─┬─ Environment(prod|dev) ─┬─ Survey ─┬─ Response ─ TagsOnResponses
                          │                          │          ├─ Display
                          │                          │          ├─ SurveyQuota ─ ResponseQuotaLink
                          │                          │          ├─ SurveyFollowUp / SurveyTrigger / SurveyLanguage
                          │                          ├─ ActionClass / Webhook / Integration / Tag
                          │                          └─ Contact ─ ContactAttribute ─ ContactAttributeKey
                          └─ Language
ApiKey ─ ApiKeyEnvironment ─ Environment      Segment ─ Survey
```

Anchors: Organization `schema.prisma:667`, Project `:628`, Environment `:582`, Survey `:344`, Response `:158`, Contact `:134`, ApiKey `:773`, Team `:1010`. Tenancy flows down this chain; every access check ultimately resolves an id *up* the chain (`getOrganizationIdFromEnvironmentId` and friends in `apps/web/lib/utils/helper.ts`).

## The big bet: documents in Json columns

Survey structure (`blocks`, `endings`, `welcomeCard`, `variables`, `styling`, `singleUse`, `recaptcha` — `schema.prisma:357-407`) and response payloads (`data`, `meta`, `ttc`, `variables` — `:167-175`) are Json, typed by the `/// [TypeName]` json-types bridge and validated by Zod (Pattern 11).

What got *real* columns instead — and why:
- Anything filtered/joined: `status`, `type`, `environmentId`, `segmentId` — the query planner needs them.
- Anything constrained: `singleUseId` + `@@unique([surveyId, singleUseId])` (`:186`), `displayId @unique`, `slug @unique` (`:414`).
- Anything counted at scale: indexes with *reasons in comments* — `@@index([surveyId, createdAt]) // to determine monthly response count` (`:188`), `@@index([contactId, createdAt])` (`:189`). Comment-your-indexes is a habit to steal.

Consistency expectations: parent-child deletes ride `onDelete: Cascade` (Response→Survey `:163`, Contact `:165`). Cross-document references (a follow-up's `to` pointing at a question id inside a Json blob) have **no referential integrity** — the app owns them, and debugging Scenario 5 shows what happens when it doesn't.

## Transactions: where atomicity actually lives

The only multi-write atomicity on the hot path: `prisma.$transaction(async (tx) => ...)` wrapping response-create + quota evaluation (`responses/lib/response.ts:24-41`), with `tx` explicitly threaded into every helper (`evaluation-service.ts:32-50` takes `tx?: Prisma.TransactionClient`; falls back to global client — the fallback is the risk, Q12 in interview-prep/01).

What's deliberately *outside* transactions: the pipeline and all side effects (post-commit); cache writes; PostHog. What's *missing* from transactions (investigate-level): quota counting is check-then-act (Pattern 12) — atomic with the response, but not isolated from concurrent counters.

## How to safely change the schema here

1. Additive first: new nullable column or new table; deploy code that writes-both/reads-old; backfill; flip reads; drop later. The migration history (`packages/database/migration/`, 100+ folders) shows this style — e.g. early ones renaming flags in two steps.
2. `pnpm fb-migrate-dev` creates the migration + regenerates the client (__inferred__, root `package.json:38`). The migration SQL is a code-review artifact — read the generated SQL, don't trust the diff summary.
3. Json-shape changes are the sneaky ones: **no migration file appears** when `ZSurvey` gains a required field, but every old row is now invalid. Options: make it optional with a default in Zod, write a data migration (see `DataMigration` model `:563` — the repo tracks these!), or version the document. If an interviewer asks "what's the hardest schema change you can imagine," "evolving a validated Json document with millions of rows" is a great answer with this repo as the example.
4. Index changes: create concurrently in Postgres for big tables (raw SQL in the migration); verify with EXPLAIN on seeded data (Ticket 8).

## Reading queries like a reviewer

Two instructive real queries:
- The notification-recipient query (`pipeline/route.ts:192-243`): five-relation nested filter with a Json-path condition (`notificationSettings: { path: ["alert", surveyId], equals: true }`). Correct, and expensive-looking; the TODO comment above it (:191) admits a caching gap. Senior read: this is where you'd check pg_stat_statements first.
- The quota count (`quotas/lib/utils.ts:135-151`): `groupBy` with `_count`, excluding the current response, conditional on partial-submission policy. Elegant SQL-shaped Prisma; its concurrency semantics are the catch (Pattern 12).

## Interview angle

1. "Normalize or not?" → the real rule this schema demonstrates: relational where the DB must *enforce or query*, document where the app owns the contract. Anchor: Survey model.
2. "How do you change a schema without downtime?" → the additive playbook + the Json-shape caveat above.
3. "Where do transactions belong?" → smallest atomic business fact (response+quota), side effects out.
4. "Design the data model for a survey tool" → draw the entity map from memory; you now can.

Drill: write the Prisma (or SQL) for "responses this month per survey for org X" and decide which existing index serves it (`:187-189`). Then check how the app actually counts (`getResponseCountBySurveyId`, `apps/web/lib/response/service.ts`) and whether your query matches its shape.
Self-grade — Basic: query returns the right thing. Solid: index choice justified. Strong: you noticed count caching implications (`createCacheKey.response.countBySurveyId` exists in the key registry — who uses it and when is it stale?).
