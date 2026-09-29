# Refactor and Design Katas

Eight judgment exercises. Output is a written artifact (diff sketch, RFC, or review), not merged code. Self-grade with each kata's criteria; "Strong" always includes knowing when NOT to do the refactor.

## Kata 1: Fix the boundary leak in `checkSurveyValidity`
Task: redesign `responses/lib/utils.ts:13-73` to return a typed result (`{valid: true, singleUseId?} | {valid: false, reason: TRejectionReason}`) instead of HTTP Responses, and remove the input mutation (:33). Sketch the route-side mapping and count call sites (v1 uses a sibling — check before claiming scope).
Grade — Basic: new signature typed correctly. Solid: route mapping preserves every status/detail exactly (list them). Strong: your writeup states the trigger condition for actually doing it ("when a non-HTTP caller appears") and estimates the diff at N files — restraint documented.

## Kata 2: Design the outbox (RFC exercise)
Task: write the full RFC for Project 2 using the template in [07-career-and-collaboration/02](../07-career-and-collaboration/02-writing-prs-and-rfcs.md). One page + schema + rollout.
Grade — Basic: mechanism correct. Solid: dual-write rollout with parity verification; dead-letter story. Strong: ordering semantics addressed (what do integrations assume today? evidence, not guesses) and the pg-boss-vs-BullMQ choice argued from the self-hosting constraint, with the losing option's advantages stated fairly.

## Kata 3: Split the pipeline monolith
Task: propose the module decomposition of `pipeline/route.ts` (348 lines): webhook delivery, notifications, integrations, follow-ups, lifecycle (autoComplete), analytics — as testable units the route orchestrates.
Grade — Basic: sensible file split. Solid: each unit's contract typed (input event, Result out), orchestration order preserved and *documented* (emails after integrations — required or accidental? find out). Strong: your plan is three PRs, each shippable, tests moving with code — not one big-bang refactor.

## Kata 4: Remove duplication across API versions — or defend it
Task: v1 and v2 responses routes share lib code unevenly (`responses/lib/response.ts:11-13` imports v1 internals). Map the sharing, then argue: consolidate into `modules/api/shared`, or keep duplication so versions can diverge?
Grade — Basic: accurate map. Solid: a decision with costs on both sides. Strong: you noticed the *versioned-contract* principle — shared code must never change v1 behavior when v2 evolves — and your structure makes that mechanically true (frozen v1 snapshots vs parameterized shared core).

## Kata 5: Type-safety hardening — kill the `any`s in auth
Task: `api/v1/auth.ts:55` (`handleErrorResponse(error: any)`) and the `accessItem: any` params in `action-client-middleware.ts:66,80`. Sketch the typed versions (the middleware ones want the `TAccess` union + narrowing by `type`).
Grade — Basic: compiles conceptually. Solid: no casts; narrowing via discriminant. Strong: you found what the `any` was *hiding* (does `checkProjectTeamAccess` handle a `team`-type item passed accidentally? typed version makes the question impossible) — that's the argument for the PR.

## Kata 6: Design a safe migration — `questions` → `blocks` completion
Task: the Survey model carries both `questions` (`schema.prisma:359`) and `blocks` (`:361`) — a migration mid-flight. Design the completion: how do you retire `questions`? (Investigate first: who still reads it? `getElementsFromBlocks(survey.blocks)` suggests blocks won; grep `survey.questions` consumers.)
Grade — Basic: correct read/write inventory. Solid: staged plan (backfill verify → read cutover → column drop N releases later) with the validation query between stages. Strong: you addressed the *API contract* — management API consumers may receive/send `questions`; retiring a column ≠ retiring a field; version implications enumerated.

## Kata 7: Reduce the notification query (performance kata)
Task: hotspot 1 (`pipeline/route.ts:192-243`). Propose the fix ladder with estimated wins: (a) cache membership→env recipients, (b) precompute on membership/settings change, (c) denormalized recipients table.
Grade — Basic: ladder ordered by effort. Solid: invalidation triggers enumerated for (a) and (b) — settings change, membership change, team change; that enumeration is the real cost. Strong: you defined the measurement gate before each rung ("don't build (b) until (a)'s hit rate is proven insufficient") — measure-first as a design discipline, not a slogan.

## Kata 8: Review a flawed PR end-to-end (capstone)
Task: Kata 4 from the review gym (in-request webhook retries) — but now write the *full* review: summary comment (intent-acknowledging, direction-setting), inline comments (severity-ranked), and the alternative you'd co-design (link your Kata 2 RFC).
Grade — Basic: findings correct. Solid: tone passes the "would I be glad to receive this?" test; blocking items have reasons + paths, not just verdicts. Strong: your summary comment gets the author to *want* the outbox design — persuasion through evidence, which is the actual senior skill being trained by this entire curriculum.

---

Cadence: one kata per week alongside module 08 prep. Katas 2, 6, 7 produce artifacts you can literally hand to interviewers ("want to see an RFC I wrote?") — polish those three.
