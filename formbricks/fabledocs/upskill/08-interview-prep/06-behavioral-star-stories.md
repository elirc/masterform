# Behavioral STAR Stories

Ten worksheets. Sources: work you actually do in module 06 (fill in Results as you complete tickets) plus two "studying the codebase" stories available immediately. Rehearsal check for all: under 2 minutes, concrete numbers/anchors, ends with impact + lesson.

## Story 1: Learning a complex codebase fast
Prompts: "Tell me about ramping up on unfamiliar code" / "How do you approach a large system?"
Source: this curriculum itself.
S: 300k+-line open-source monorepo (Next.js/Prisma/Redis, 36-model schema, embedded SDK), no team to ask. T: productive understanding in days. A: read maintainer docs first (AGENTS.md), mapped the tenancy chain from the schema outward, then traced seven end-to-end flows (submission → pipeline → webhooks) writing trace tables; verified every claim against code, kept a log separating confirmed facts from hypotheses. R: could locate and explain any major flow; found three architecture-level risks (delivery semantics, a count race, fail-open auth limiting) with evidence. Senior-signal: the confirmed-vs-hypothesis discipline. Resume bullet: *Mapped and documented a 36-model open-source SaaS monorepo, producing a verified risk register and contribution plan within one week.*

## Story 2: Finding a race condition by reading
Prompts: "A subtle bug you found" / "Concurrency experience?"
Source: Pattern 12 / Ticket M6 / Project 4 (upgrade the Result when you build it).
S: quota enforcement counted screened-in responses then inserted, inside a READ-COMMITTED transaction. T: assess whether limits could overshoot. A: wrote the two-transaction interleaving proving both pass the count; distinguished atomicity (present) from isolation (absent); designed the fix ladder (conditional UPDATE / FOR UPDATE / advisory lock) with contention costs; built a two-parallel-submits regression test. R: [fill after M6 — e.g. "confirmed overshoot locally, fix merged"]. Senior-signal: atomicity≠isolation stated precisely; acceptable-overshoot considered as an option. Resume bullet: *Identified and fixed a check-then-act race in quota enforcement, adding concurrency regression tests.*

## Story 3: Improving reliability without a rewrite
Prompts: "Biggest technical improvement you've driven."
Source: the M3→M4→M6/M1 arc, then Project 2's RFC.
S: event fan-out (webhooks/emails) was fire-and-forget with zero delivery metrics. T: improve reliability incrementally. A: instrumented outcomes first (counters by result), capped unbounded fan-out concurrency from measured data, then wrote an outbox RFC honoring the self-hosting constraint (no new infra — pg-boss on existing Postgres), with dual-write rollout and a kill-9 zero-loss acceptance test. R: [fill in]. Senior-signal: measure → bound → fix ordering; constraint-driven design. Resume bullet: *Designed measured, incremental reliability upgrades (metrics → backpressure → transactional outbox RFC) for an at-most-once event pipeline.*

## Story 4: Disagreeing with a design (fail-open rate limiting)
Prompts: "Disagreed with a technical decision" / "Pushed back and were wrong/right."
Source: R3 + Ticket M11 + the disagreement script in 07-03.
S: the rate limiter fails open everywhere, including login brute-force protection. T: challenge respectfully. A: acknowledged the availability rationale; quantified the exposure window (Redis outage = unthrottled auth); proposed the *narrowest* change (fail-closed for auth:* behind a flag) rather than inverting the philosophy; accepted that operators decide. R: [outcome or "proposal documented"]. Senior-signal: narrowing the disagreement to the smallest defensible change. Resume bullet: *Drove a threat-model-scoped change to fail-open rate limiting for authentication endpoints.*

## Story 5: A bug that was actually architecture
Prompts: "Hardest bug" / "A time you couldn't just fix it."
Source: debugging scenario 1 (lost webhooks).
S/T: intermittent missing webhook deliveries, pressure for a quick fix. A: bisected by log absence between producer and consumer; proved the loss window was the unawaited post-commit call; showed why in-request retries (the tempting patch) would worsen it; delivered detection immediately (alert on the failure log) and escalated the design fix with evidence. R: honest expectations set; roadmap item created. Lesson line: "some bugs are architecture; the fix is a document, not a diff." 

## Story 6: Making the invisible visible (observability)
Prompts: "Improved operations/monitoring."
Source: observability gap table + Tickets 17/M3.
S: three failure classes (pipeline sends, webhook outcomes, limiter fail-open) were logs-only. T: cheapest credible detection. A: added Redis to health checks, outcome counters at the single wrapper choke point, "how we'd know" sections to PR templates. R: [fill]. Senior-signal: choke-point placement (one file, all routes) over sprinkled instrumentation.

## Story 7: Scope discipline under review
Prompts: "Received hard feedback" / "PR that grew."
Source: any module-06 ticket where review suggested expansion (e.g. Ticket 4's "do all three wrappers").
A-skeleton: agreed on value, split into follow-up issue, kept the diff reviewable (CI size checks as ally), shipped the slice. Lesson: velocity through smallness.

## Story 8: Cross-layer feature end-to-end
Prompts: "Walk me through something you built."
Source: Ticket M10 (API-key last-used) or M7 (conflict warning) once done.
Structure the telling by layers: schema (additive migration + rollback line) → service (throttled write, why) → UI → tests per layer. The write-amplification tradeoff (hourly not per-request) is your depth moment.

## Story 9: Security thinking on a normal feature
Prompts: "How do you think about security day-to-day?"
Source: M8 (error beacon) design, or applying the pre-merge checklist to any ticket.
Key beat: designed an *unauthenticated* endpoint by assuming hostility — rate-limit namespace, size caps, PII scrubbing, sampling — and named the trust model (code on pages we don't control). Works even if unbuilt: "here's the design note."

## Story 10: Teaching/leveling others
Prompts: "Helped a teammate grow."
Source: the teach-backs — explaining Flow 1 or the caching stack to someone (do it for real: a peer, a meetup, a written post).
R: their restatement quality / the artifact. If you produce one public write-up from this curriculum (the three-cache story is the most shareable), this story writes itself and doubles as portfolio.

---

Coverage check vs common prompts: conflict (4), ambiguity (1,5), mistake (7 — or convert 2 if your first hypothesis was wrong: *say so*, wrongness-then-evidence is the best mistake story), tradeoff (3,8), leadership (10), failure-with-lesson (5). Rehearse the two strongest (1 and 5 are available *today*) before your first screen; upgrade Results fields as module-06 work completes.
