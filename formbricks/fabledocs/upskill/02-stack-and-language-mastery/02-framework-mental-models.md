# Framework Mental Models

## Next.js App Router: the two-world model

Mental model: every file is born a **Server Component** (runs at request time on the server, can touch the DB, ships no JS) until `"use client"` opts it into the browser world. Data flows server→client as serialized props; mutations flow client→server as **server actions** (`"use server"` RPCs).

This repo's usage:
- Server-side page shells under `apps/web/app/(app)/environments/[environmentId]/...` fetch data and hand it to client components (see how `survey-editor.tsx` receives `survey` as a prop and clones it, below).
- Route groups `(app)`, `(auth)`, `(internal)` organize without affecting URLs; dynamic segments like `[environmentId]` carry tenancy through the URL — which is why every client-API handler validates that param first.
- Mutations: server actions via next-safe-action (Flow 4) rather than API routes — the dashboard has almost no fetch('/api/...') for writes. Exception: the newer v3 surveys *list* API consumed by TanStack Query (below) — reads via REST, writes via actions is the emerging house split.
- `revalidatePath` (`editor/actions.ts:328`) invalidates Next's server-side render cache — distinct from Redis (Flow 3) and from TanStack Query's client cache. Three cache systems, three invalidation mechanisms; naming all three precisely is instant mid-level credibility.

Sharp edge: a server action is a public endpoint. The middleware chain (authn → schema → authz, Flow 4) exists because "it's just a function" is false. Grep exercise: find any action built on raw `actionClient` (not `authenticatedActionClient`) and justify each.

## React 19 as used in the dashboard

State placement example — the survey editor (`apps/web/modules/survey/editor/components/survey-editor.tsx:86-113`):
- `localSurvey = useState(() => structuredClone(survey))` — a *draft copy* of server data. Lazy initializer (function form) so the clone runs once, not per render. Edits mutate the draft; explicit save pushes it through `updateSurveyAction`. This is the classic "server state vs form state" split done manually.
- A dozen `useState` hooks colocated at the top — activeView, activeElementId, styling drafts. Fine at this scale; the senior question is when this should become a reducer (interdependent transitions) — the caution/validation states hint it's near the line.
- What wins on conflict? Nothing — last write wins (see reading-order file 28's note). Multi-tab editing loses data silently. Product decision embedded in a useState; know it exists.

Effects discipline: `AGENTS.md` mandates cleanup patterns that snapshot refs inside `useEffect` — the stale-closure family of bugs, acknowledged at the convention level.

## TanStack Query: server-state on the client

Anchor: `apps/web/modules/survey/list/hooks/use-surveys.ts:8-51`.
- `useInfiniteQuery` with cursor pagination (`getNextPageParam` reads `meta.nextCursor`).
- **Key factory** (`surveyKeys.list({...})` from `modules/survey/list/lib/query`) — keys as data, centrally defined, so invalidation can't typo. Same idea as the Redis `createCacheKey` registry (Pattern 5) — one discipline, two caches; say that sentence in an interview.
- `placeholderData: keepPreviousData` — old page stays visible while the next filter's data loads (no flash).
- Cost control: `includeTotalCount: pageParam === null` — the expensive COUNT runs only on the first page. A small line with a real query-plan story behind it.
- Provider scoped to the surveys layout (`app/(app)/environments/[environmentId]/surveys/query-client-provider.tsx`) rather than app-wide — cache lifetime bounded to the feature. Deliberate or accidental? Good maintainer question.

## The widget world: Preact + no framework assumptions

`packages/surveys` renders the survey itself with Preact components (`src/components/`), compiled to a self-contained bundle (Pattern 16). It cannot assume React exists on the host page, must style defensively, and manages its own state machine + `ResponseQueue`. When you see React idioms there, check imports — it's Preact's compat layer, and bundle size is the reason.

## Interview angle

1. "Server Components vs client components — what goes where?" → answer with the editor: server shell fetches, client component owns the draft; cross-link [08-interview-prep/02](../08-interview-prep/02-frontend-framework-questions.md) Q1–Q3.
2. "How do you paginate a large list?" → `use-surveys.ts` cursor + infinite query + count-once (Q6 there).
3. "Three caches in one app — name them and their invalidation" → Next render cache / TanStack Query / Redis. (Q5 there.)
4. "Why would you ever not use React?" → the widget's constraints; embeddable-JS tradeoffs.

Drill: open `survey-editor.tsx` and diagram which state is (a) server data snapshot, (b) UI-only state, (c) derived. Then answer: if `updateSurveyAction` fails, what un-saves? (Trace the error path back into component state.)
Self-grade — Basic: correct buckets. Solid: found the failure-path answer in code. Strong: you can propose where optimistic updates or a reducer would pay, and what they'd cost.
