# Maintainer Communication

Open-source maintainers triage dozens of threads daily. Every template below optimizes for *their* scan-read, which is also exactly how you should write to senior engineers at work.

## Asking for help without outsourcing thinking

The rule: show the search before asking for the destination.

```markdown
**Goal**: make the widget show a quota-full message when POST /responses returns quotaFull.
**What I've found**: onQuotaFull callback exists in ResponseQueue config (packages/surveys/src/lib/response-queue.ts:27); I can't find where js-core surfaces it to the host page.
**What I tried**: grepped js-core exports; read index.ts:14-60 — commands funnel to CommandQueue but no quota event.
**Question**: is host-page exposure intentionally unsupported, or just not built? If the latter, would a PR adding an event be welcome, and where should it hook?
```

Four lines of evidence turn "help me" into "confirm my map" — answerable in one minute, and it *advertises competence* while asking.

## Bug reports

Minimal repro or it didn't happen. Structure: **versions/env → exact steps → expected → actual → evidence (logs with context, not screenshots of logs) → suspicion (labeled as such)**. Suspicion example done right: "possibly the 60s env-state TTL (environmentState.ts:69) — timing fits, but I haven't confirmed the Redis layer specifically." Wrong: "your cache is broken."

## Proposing a feature

Lead with the problem, not the solution; size the change; offer the labor:
```markdown
**Problem**: self-hosters can't tell whether webhooks are being delivered (only per-attempt error logs).
**Proposal sketch**: delivery-outcome counter metrics; optionally a WebhookDelivery table (details negotiable).
**Scope**: metrics-only version is ~1 day, no schema change; table version needs a retention decision.
**I'm offering**: to build either — which fits the roadmap?
```

## Respectful disagreement

Acknowledge the constraint you might be missing → evidence → concrete alternative → genuine exit: *"Makes sense that fail-open protects availability. My worry is the auth namespace specifically — Redis loss un-throttles login brute force (rate-limit.ts:129-134). Would a fail-closed allowlist for auth:* behind a flag be acceptable? If there's an operational reason this was rejected before, happy to drop it."* If they say no twice, you drop it or write the RFC — never re-litigate in review threads.

## Responding to review on your PR

- Every comment gets a response; batch-push fixes, then reply "done" with commit refs.
- Wrong feedback: assume missing context first — yours or theirs. *"I think the tx-threading covers that case (evaluation-service.ts:43 falls back only when tx is undefined, and route always passes it) — am I reading it right?"* You're correct → you taught kindly; you're wrong → you learn where your model broke. Either way you win.
- Feedback that grows scope: *"Agreed it's worth doing — can I take it as a follow-up so this stays reviewable? Filed as [issue]."*

## Cadence and channels

Issues for anything you'd want findable in a year; PR comments for the diff at hand; discussions/Discord for "is this direction sane" *before* investing weeks (the Project 6 policy proposal, the SDK-surface tickets). Silence for a week ≠ rejection — ping once, politely, with new information if you have it ("bumping with a repro script").

Interview angle: behavioral questions about conflict and ambiguity ("disagreed with a decision", "pushed back on review") are answered *verbatim* by the disagreement and wrong-feedback scripts above — you're rehearsing the stories as you use them. Tag each real exchange you have while working this repo; three of them become STAR worksheets in [08-interview-prep/06](../08-interview-prep/06-behavioral-star-stories.md).
