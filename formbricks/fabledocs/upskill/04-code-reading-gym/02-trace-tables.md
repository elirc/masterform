# Trace Tables

Fill each table yourself first (columns: step, file:line, value shape, owner, transformation, risk), then compare. Blank rows marked ⟵ are for you.

## Trace 1 (UI → API): dashboard survey list loads page 2

| Step | File | Value shape | Owner | Transformation | Risk |
| --- | --- | --- | --- | --- | --- |
| 1 | `modules/survey/list/components/survey-list.tsx` | filters object | UI state | user filter clicks → `TSurveyOverviewFilters` | filter state vs URL state divergence |
| 2 | `modules/survey/list/hooks/use-surveys.ts:19-23` | `surveyKeys.list({workspaceId, limit, filters})` | Query cache | inputs → stable cache key | unstable key (object identity) would refetch forever — key factory prevents |
| 3 | `use-surveys.ts:30-38` | `{pageParam: cursor}` | TanStack | cursor → HTTP query params; `includeTotalCount` only when cursor null | COUNT cost on every page if that flag regressed |
| 4 | `.../lib/v3-surveys-client.ts` → `GET /api/v3/surveys` | JSON `{data, meta:{nextCursor,totalCount}}` | API v3 | DB rows → DTO | authz: which environments may this session list? (find the check in the v3 route — do it) |
| 5 | `use-surveys.ts:39-43` | flattened `surveys[]` + totalCount | hook | pages → flat list | duplicate rows if cursor unstable across mutations |

Predict-then-verify: what happens to page 2's data when you delete a survey from page 1? (Read `use-delete-survey.ts` — what does it invalidate?)

## Trace 2 (persistence): one answer's journey into Postgres

| Step | File | Value shape | Transformation | Risk |
| --- | --- | --- | --- | --- |
| 1 | widget input | `{ [elementId]: "Berlin" }` | user keystrokes → ResponseData | — |
| 2 | `packages/surveys/src/lib/response-queue.ts` | `TResponseUpdate` | accumulate + finished flag | offline: parked in IndexedDB |
| 3 | `responses/route.ts:62-79` | `TResponseInputV2` | JSON body + environmentId merged, Zod-parsed | junk fields dropped/rejected here |
| 4 | `responses/route.ts:124-137` | validated data | per-element validation vs survey blocks | schema drift: old survey Json vs current rules |
| 5 | `responses/lib/response.ts:46-60` (`buildPrismaResponseData`) | `Prisma.ResponseCreateInput` | domain → ORM shape; ttc totals computed | field mapping bugs live here |
| 6 | Postgres `Response.data` Json | `{"q_x":"Berlin"}` | serialized | queryable only via Json operators |

## Trace 3 (auth): session established at login

| Step | File | Value shape | Risk |
| --- | --- | --- | --- |
| 1 | login form → NextAuth credentials callback | email+password+optional totp | — |
| 2 | `authOptions.ts:191` | rate-limit result | Redis down ⇒ fail-open |
| 3 | `authOptions.ts:221-235` | user row + bcrypt compare (control hash if absent) | timing uniformity is the invariant |
| 4 | `authOptions.ts:266-310` | backup-code branch: decrypt, match, burn | ENCRYPTION_KEY dependency |
| 5 | NextAuth callbacks (`authOptions.ts` later sections — read `jwt`/`session` callbacks yourself) | JWT → session object | ⟵ what claims end up in the cookie? fill in |
| 6 | subsequent request | `getServerSession` (`action-client/index.ts:52`) | session → `ctx.user` | stale user data vs DB (deactivated user with live session — check `isActive` handling in callbacks) |

## Trace 4 (error path): duplicate single-use submission

| Step | File | Value shape | Risk |
| --- | --- | --- | --- |
| 1 | second POST with same suId | `TResponseInputV2.singleUseId` | — |
| 2 | `responses/lib/response.ts` create | Prisma P2002 unique violation | must be *recognized*, not generic |
| 3 | `response-error.ts` (`isSingleUseIdUniqueConstraintError`) | typed guard → `UniqueConstraintError` | brittle if schema constraint name changes |
| 4 | `responses/route.ts:180-182` | 409 conflict response | client must render "already submitted", not retry forever |
| 5 | widget `response-queue.ts` error handling | ⟵ read `sendResponse`: does a 409 stop retries? fill in | infinite retry loop if not |

## Trace 5 (async): responseFinished → follow-up email

| Step | File | Value shape | Risk |
| --- | --- | --- | --- |
| 1 | `responses/route.ts:252-259` | pipeline event JSON | Dates → strings |
| 2 | `pipeline/route.ts:39-58` | re-hydrated `TPipelineInput` | skip-set correctness |
| 3 | `pipeline/route.ts:245-254` | `sendFollowUpsForResponse(response.id)` — refetches response fresh | id-based refetch avoids trusting the serialized copy: good |
| 4 | `follow-ups.ts:21-70` | `to` resolved: direct email or `response.data[to]` | dangling element id (debug scenario 5) |
| 5 | `follow-ups/lib/email.ts` | rendered email | rate limit 50/hr; whitelabel logo from org |

## Trace 6 (cache): environment state, cold → warm → stale

| Step | File | State | Risk |
| --- | --- | --- | --- |
| 1 | first GET | Redis miss → DB query → Redis set (60s) | stampede on popular envs? (single-flight — verify in `service.ts:241+`) |
| 2 | GETs for 60s | Redis hit; CDN also caching per headers | — |
| 3 | survey published at t=30s | DB new, caches old | dashboards show it (different path), widgets don't |
| 4 | t=60-120s | layers expire in order CDN/Redis | up to ~2min for fresh loads |
| 5 | already-loaded SDK | holds until `expiresAt` +1h | longest staleness window |

## Self-grading (all traces)

Basic: rows correct after checking. Solid: you filled the ⟵ rows from code, not guesses. Strong: each trace yielded at least one "I should verify…" note (e.g. the 409-retry question in Trace 4) — collect these; they're tickets and interview stories in larval form.
