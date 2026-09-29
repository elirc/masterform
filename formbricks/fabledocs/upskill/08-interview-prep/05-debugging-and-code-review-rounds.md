# Debugging and Code-Review Round Simulations

Seven timed simulations converted from modules 04/05. Rules: timer on, talk aloud continuously (interviewers grade the search narration, not silent brilliance), write your answer before checking the source material.

## Debugging round A (25 min): "Webhooks missing for some responses"
Source: debugging scenario 1.
Setup you're given: "A customer reports ~2% of responses never hit their webhook endpoint. Responses appear in the dashboard. Logs are available. Go."
Interviewer follow-ups to rehearse: "You can't reproduce it — now what?" (log-absence bisection between `pipelines.ts:23` and `pipeline/route.ts:175`); "Fix it." (the honest answer: no small fix — at-most-once design; propose measurement M3 then outbox, and *say the cost of each*); "What if I told you it correlates with deploys?" (process-death window at the unawaited call — `responses/route.ts:245`).
Rubric — hire-signal behaviors: states delivery-semantics vocabulary unprompted; bisects by evidence not vibes; distinguishes patch vs design fix. Anti-signals: proposes retries inside the request (Kata 4's trap); blames the customer's endpoint without evidence.

## Debugging round B (20 min): "Survey edits not appearing"
Source: scenario 2. Setup: "Support escalation: customers say published changes take forever. Sometimes a minute, sometimes an hour."
Follow-ups: "Which layer explains 'sometimes an hour'?" (SDK `expiresAt` — the bimodal distribution *is* the diagnostic: two populations, two layers); "Design the fix and its blast radius" (M1 invalidation + accept CDN 60s).
Rubric: curl-based layer bisection; using the *distribution shape* as evidence = strong; jumping to "add cache busting everywhere" = weak.

## Debugging round C (20 min): "Login takes 4 seconds"
Source: scenario 4. Setup: "Started this morning. No deploy. All users."
Follow-ups: "Redis is 'up' per the dashboard" (up ≠ healthy — connect-timeout adds 3s before fail-open, `cache/src/client.ts:24-26`); "Why didn't your monitoring catch it?" (fail-open logs exist, nobody alerts — observability gap, name the metric).
Rubric: CPU-vs-I/O split in the first minute; the "fallback paths need latency budgets" conclusion.

## Debugging round D (15 min, rapid): "Duplicate responses on a single-use survey"
Source: scenario 3. The DB-constraint-as-proof reasoning, compressed. Rubric: reaches "true duplicates are impossible; therefore the ids differ or the sighting is wrong" within 5 minutes.

## Review round A (30 min): the unauthorized admin action
Source: review kata 3. You're handed the diff description; produce a written review in 20 min, then defend it in 10.
Defense questions: "The author says it's internal-only tooling, ship it" — hold the line kindly: internal is a network claim, actions are public endpoints (`"use server"` reality); offer the 3-line fix (swap client + add authz) so blocking costs the author minutes, not days.
Rubric: found both blockers (authn AND authz — many candidates find one); severity ranking correct; comments quote house precedent (`editor/actions.ts:250-267`).

## Review round B (30 min): the hot-path cache
Source: review kata 5. Defense question: "p95 improved 40ms — why are you blocking a win?" (correctness of *paused-survey* enforcement beats 40ms; offer the safe alternative: cache after status checks, or invalidate-on-update).
Rubric: identifies stale-*enforcement* (not just stale data) as the issue; proposes the preserving-the-win alternative — blocking with a path reads senior, blocking with a "no" reads junior.

## Review round C (20 min): the flaky-test patch
Source: review kata 8. Softest diff, subtlest judgment: nothing is "broken," everything is worse. Rubric: names the hidden product insight (what's actually racing — bundle load or cache TTL — is worth a ticket); requests event-based waits with a concrete locator example, not a lecture.

---

## Scoring yourself across all seven

After each: (1) Did I state a plan before acting? (2) Did I name evidence for every claim? (3) Did I give severity/priority rankings? (4) Did I offer a path, not just a verdict? (5) Did I say "I'd verify X before concluding" at least once? Four of five = interview-ready on that scenario; repeat the misses next week. The two-week plan (file 07) schedules exactly this rotation.
