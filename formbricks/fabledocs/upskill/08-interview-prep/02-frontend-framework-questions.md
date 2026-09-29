# Frontend Framework Questions

Twelve cards on React 19 / Next.js App Router, anchored to the dashboard and the widget. Round: frontend.

## Q1: Server Components vs Client Components — how do you split a real feature?
Testing: RSC mental model.
Anchor: the editor — server page shells under `app/(app)/environments/[environmentId]/surveys/[surveyId]/edit/` fetch and pass props; `modules/survey/editor/components/survey-editor.tsx:1` opts into `"use client"` and owns interaction.
Junior: "server components can't use hooks."
Mid: split by *interactivity need*: data assembly + authz on the server (no bundle cost, secrets safe); the stateful editor client-side; props crossing the boundary must serialize.
Senior: adds the contract view — the server→client prop is a snapshot; the editor clones it (`:88`) precisely because props aren't owned state; and the mutation returns through a typed server action, closing the loop without client fetch code.
Follow-ups: "What can't cross the boundary?" (functions, class instances, Dates become strings in some paths.)
Drill: sketch the editor's component tree marking S/C per node.

## Q2: Where does form/draft state live, and why not in the URL or a store?
Testing: state placement judgment.
Anchor: `survey-editor.tsx:86-113` — a dozen `useState`s incl. `localSurvey` (lazy-initialized `structuredClone`).
Junior: "useState for everything."
Mid: taxonomy — server snapshot (localSurvey), UI state (activeView), derived (invalidElements); URL for shareable state (which tab), local for drafts; a global store only when siblings can't share via props.
Senior: names the *conflict semantics* hiding in the choice: a local draft + explicit save = last-write-wins; multi-tab data loss; and the upgrade options (updatedAt check — Ticket M7, or full CRDT) with costs. State placement is concurrency design; saying that sentence wins the question.
Follow-ups: "When does this become useReducer?" (interdependent transitions — validation + caution dialogs are near the line.)

## Q3: Lazy useState initializer — why does `useState(() => structuredClone(survey))` matter?
Testing: render-model precision.
Anchor: `survey-editor.tsx:88`.
Junior: "it sets initial state."
Mid: the function form runs *once* on mount; passing `structuredClone(survey)` directly would clone on **every render** and discard it — wasted work proportional to survey size.
Senior: connects to render purity — initializers and render bodies run more often than juniors think (StrictMode double-invoke!); expensive work belongs in initializers, memos, or events; and clone-vs-reference is also a *correctness* choice (mutation of props would corrupt the server snapshot silently).
Follow-ups: "What breaks if we skip the clone?"

## Q4: How does TanStack Query decide when to refetch, and how do you keep keys sane?
Testing: server-state cache mechanics.
Anchor: `modules/survey/list/hooks/use-surveys.ts:19-29` — key factory (`surveyKeys.list({...})`), `placeholderData: keepPreviousData`.
Junior: "it caches API calls."
Mid: keys are identity — structural key from a factory prevents both typo-divergence and object-identity churn; staleTime vs gcTime; `keepPreviousData` for filter UX.
Senior: invalidation topology — mutations invalidate by key *prefix* (`use-delete-survey.ts` — read what it invalidates); the factory centralizes that contract exactly like `createCacheKey` does for Redis (same discipline, two tiers — say this cross-layer sentence).
Follow-ups: "Query cache vs Next's router cache vs Redis — who invalidates what?" (The three-cache answer from module 02.)

## Q5: Cursor vs offset pagination in an infinite list.
Testing: pagination + UX interplay.
Anchor: `use-surveys.ts:25-43` — `useInfiniteQuery`, `getNextPageParam` reading `meta.nextCursor`, `includeTotalCount` only on first page.
Junior: "offset uses page numbers."
Mid: offset breaks under concurrent inserts (rows shift → duplicates/skips) and costs OFFSET-scans; cursor (keyset) is stable and index-friendly; infinite scroll naturally wants cursors.
Senior: the COUNT economics (`includeTotalCount: pageParam === null` — count once, it's the expensive part); what invalidation does to loaded pages after a delete; and when offset is *right* (jump-to-page admin grids).
Follow-ups: "Design the cursor: what fields, why opaque?"

## Q6: A component re-renders too much — walk your diagnosis.
Testing: performance method, not memo trivia.
Anchor: the editor's block list (`elements-view.tsx`, `block-card.tsx`) as the plausible patient.
Junior: "wrap in React.memo."
Mid: profile first (React DevTools flame); find the *cause* — parent state churn, unstable props (inline objects/functions), context width; fix at the source; memo as last-mile with stable props.
Senior: state *architecture* as the real fix — keystroke-frequency state (a question's text) shouldn't live where it re-renders 40 siblings; colocate or split contexts; and knows memo's cost (comparison + staleness bugs with mutation — which matters here because the editor mutates a cloned object).
Follow-ups: "When is memoization harmful?"

## Q7: How do server actions change classic form handling?
Testing: mutations in the App Router era.
Anchor: `updateSurveyAction` (Flow 4) called from the menu bar; next-safe-action returning typed `{data}|{error}`.
Junior: "you don't need an API route."
Mid: RPC with progressive enhancement; input schema colocated (`inputSchema(ZSurvey)`); pending/error via hooks (`useAction` — grep its usage); but each action is a public endpoint needing authz (the middleware chain).
Senior: contrasts the repo's *split*: writes via actions (session-authed, dashboard-only), reads for lists via REST + TanStack (cacheable, cursor-friendly) — and why that split is principled, not accidental (mutations want authz+audit; list reads want cache semantics HTTP already has).
Follow-ups: "CSRF story for actions?" (origin checks; know it exists, verify per deployment.)

## Q8: Optimistic updates — when would you add them here, and what's the rollback story?
Testing: UX-consistency tradeoffs. Conceptual — no direct anchor (the editor is *not* optimistic; save is explicit).
Junior: "update the UI first, it feels fast."
Mid: optimistic = apply locally, reconcile on response, roll back on error (TanStack's `onMutate`/`onError` snapshot dance); good for low-conflict, reversible ops (delete survey with undo) — `use-delete-survey.ts` is where you'd add it; read what it does today first.
Senior: enumerates where optimism is *wrong*: non-idempotent effects, server-computed outcomes (quota results!), multi-writer surfaces (the editor's LWW problem would get worse); consistency perception is a product decision, not a hook.
Follow-ups: "Optimistic create — what do you use for the temporary id?"

## Q9: The widget runs on arbitrary customer pages. What breaks, and how does the code defend?
Testing: embeddable-frontend awareness (rare, differentiating).
Anchor: Preact renderer (`packages/surveys`), UMD bundle, module-level locks (`response-queue.ts:36-53`), IndexedDB offline store, `preloadSurveysScript` (`js-core/src/lib/survey/widget.ts:301`).
Junior: (usually has nothing.)
Mid: CSS collisions (scoped/reset styles), bundle size (Preact over React), no framework assumptions, resilient storage (IndexedDB may be blocked — degrade).
Senior: adds lifecycle chaos (SPA route changes → `checkPageUrl` listeners in js-core), double-load protection (singleton CommandQueue), and the *versioning* problem — deployed SDKs can't be force-upgraded, so the client API must stay backward-compatible ~forever (this constraint shapes Project 3).
Follow-ups: "How would you test this?" (Playwright against a hostile fixture page.)

## Q10: Accessibility — what would you audit first in a survey renderer?
Testing: a11y priorities under real constraints.
Anchor: `packages/surveys/src/components/` (inputs, buttons, progress) — respondent-facing = the highest a11y stakes in the product.
Junior: "add aria-labels."
Mid: keyboard-first walk of an entire survey (focus order across cards, Enter-to-advance vs textarea newlines), labels/roles on custom inputs (rating scales! `elements/` picker components), error announcement (`aria-live` on validation), contrast under *customer-themed* styling.
Senior: the theming trap — customers control colors (`styling` Json), so contrast is a *runtime* property; either constrain the theming API or surface a contrast warning in the editor; a11y as product architecture, not attribute sprinkling.
Follow-ups: "How do you regression-test it?" (axe in Playwright + manual SR pass on one flow.)

## Q11: i18n at scale — how does this app keep translations complete?
Testing: i18n process maturity.
Anchor: `t()` mandate (AGENTS.md), `apps/web/locales/`, `pnpm i18n` + `translation-check.yml` CI, lingo.dev auto-translation on commit; survey *content* translations are separate (`SurveyLanguage`, `schema.prisma:990`).
Junior: "JSON files per language."
Mid: two systems — UI chrome (message catalogs, CI-enforced) vs user content (per-survey languages, data-modeled); key naming conventions; machine translation as draft.
Senior: the split *is* the insight — UI strings are code (CI can gate), survey translations are data (product must handle missing-language fallbacks at render: find the fallback in the renderer). Plus locale ≠ timezone discipline (AGENTS.md's date rules).
Follow-ups: "What breaks with RTL?" 

## Q12: Explain hydration errors and where this app risks them.
Testing: SSR/CSR boundary depth.
Anchor: conceptual, with repo-plausible sites: date rendering (server locale vs client locale — exactly why AGENTS.md mandates shared formatters + explicit locale source), anything reading `window` during render.
Junior: "server HTML didn't match client."
Mid: causes — nondeterminism (dates, random ids), environment branching (window checks), locale differences; fixes — deterministic render inputs, useEffect for client-only, suppressHydrationWarning as last resort.
Senior: points at the house rule as *systemic prevention* (locale from `user.locale`/i18n, never browser-implicit) — conventions that make the bug class unrepresentable beat fixing instances; that's the pattern-vs-instance answer interviewers reward everywhere.
Follow-ups: "Why does React 19 make this better/worse?" (Better errors, selective hydration.)
