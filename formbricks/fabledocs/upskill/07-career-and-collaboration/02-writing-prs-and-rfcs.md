# Writing PRs and RFCs

## PR mechanics in this repo

CI will judge you before humans do: `semantic-pull-requests.yml` wants conventional titles (`feat:`, `fix:`, `docs:`, `chore:`); `pr-size-check.yml` penalizes big diffs — split before submitting; `translation-check.yml` fails hardcoded user-facing strings; lint/test/e2e must pass. Pre-commit hooks (husky + lint-staged) format on commit.

## The PR description template

```markdown
## What
One paragraph. The change, not the diff. "Adds Retry-After headers to 429 responses from the v1 API wrapper."

## Why
The user/operator problem, with evidence (issue link, log excerpt, anchor).

## How
Only the non-obvious decisions: "Threaded retryAfter from checkRateLimit (already computed, rate-limit.ts:85) through the wrapper rather than recomputing."

## How tested
Exact commands + what they prove. "pnpm --filter @formbricks/web test -- with-api-logging — new cases: 429 includes header; non-429 unaffected."

## Risks
Blast radius + rollback. "Additive header; no client can break. Rollback: revert."

## Follow-ups
What you deliberately didn't do. "v2/v3 wrappers tracked in #xxx."
```

The Risks and Follow-ups sections are what distinguish mid-level PRs — they prove you saw the edges of your change. Never omit them; "Risks: none I can see, because X" is itself information.

Commit messages: imperative subject ≤72 chars matching the semantic type; body = why. Squash-friendly: each PR tells one story.

## When to RFC instead

Trigger conditions: touching a public contract (client API, SDK, webhook payload), adding infrastructure (queue, table with retention), changing security posture (fail-open→closed), or any change whose *rollback is hard*. If reviewers would need to imagine the design from a diff, the design needs its own document first.

## RFC template (tuned to this repo)

```markdown
# RFC: [Title]
Status: Draft | Reviewed | Accepted
## Problem
Observable today: [evidence — logs, anchors, support tickets]. Who pays: [users/operators/maintainers].
## Constraints
Self-hostable (Postgres+Redis+S3 only). Three API versions live. Widget SDKs deployed on customer pages can't be force-upgraded.
## Proposal
Mechanism + the smallest schema/API sketch that makes it concrete.
## Alternatives considered
2-3, each with the reason it lost STATED FAIRLY (a strawman alternative discredits the whole doc).
## Migration & rollout
Flags, dual-write/parity phases, cutover criteria, rollback per phase.
## Test & observability plan
The acceptance test (e.g. "kill -9 under load, zero lost deliveries"). What metric proves it works in prod.
## Open questions
Genuinely open ones — an RFC with no open questions reads as a decree.
```

Worked example to study: write Project 2's outbox RFC (Kata 2) with this template; the architecture-critique file's month-3 plan gives you the content, the template gives the form.

## Interview angle

"Tell me about a technical document you wrote" — an RFC beats any verbal answer; bring the outbox RFC. Also: PR-description discipline *is* the answer to "how do you communicate risk?" — quote your own Risks sections.

Drill: write the full PR description for junior Ticket 1 (the log-context change) — yes, all six sections for a 5-line diff. Feeling how fast it goes when the change is small is the point: the template's cost is minutes; skipping it costs review round-trips.
