# Formbricks Upskill Curriculum

A training lab built on the real Formbricks codebase for one learner: a junior fullstack JS engineer (React/Node/TS CRUD experience) who wants to reach mid-level fast, build senior judgment, and pass interviews for mid-level fullstack roles.

Every page teaches two things at once:

1. **This codebase** — where things live, how its real flows work, with exact file/line anchors.
2. **Transferable skill** — why the pattern exists, when it fails, and how to talk about it in an interview.

## What this repo is

Formbricks is an open-source Experience Management platform: teams create surveys (link surveys, in-app "app" surveys, website surveys), collect responses, and act on them via integrations, webhooks, and email follow-ups. It runs as a pnpm + Turborepo monorepo. `apps/web` is a Next.js App Router application containing the dashboard UI, three generations of REST APIs (`/api/v1`, `/api/v2`, `/api/v3`), server actions, and an internal event pipeline. Persistence is Postgres via Prisma (`packages/database/schema.prisma`, ~36 models), with Redis for caching and rate limiting (`packages/cache`) and S3-compatible storage for file uploads (`packages/storage`). The survey renderer (`packages/surveys`, Preact, compiled to a UMD bundle served from `apps/web/public/js/`) and the browser SDK (`packages/js-core`) ship to end-user websites — a genuinely distributed system with offline response queueing. Enterprise features (SSO, audit logs, quotas, teams) live behind a license check in `apps/web/modules/ee/`.

## How to use this curriculum

| Time budget | Path |
| --- | --- |
| One weekend | [00-fast-track.md](00-fast-track.md) — run it, trace two flows, make one safe change |
| Two weeks (interview soon) | [08-interview-prep/07-two-week-cram-plan.md](08-interview-prep/07-two-week-cram-plan.md) |
| Eight weeks | Modules 01 → 05 in order, one module ~per week, doing every drill; modules 06–07 in weeks 6–8 |
| Ongoing contribution | Module 06 tickets, using 05 (quality) and 07 (collaboration) as your working handbook |

### Recommended paths by profile

- **Brand-new junior**: 00 → 01 (all) → 02 → 04 (drills) → 05-01/02 → 06-01 tickets. Skip 03-06 (critique) until later.
- **Junior with React/Node/TS familiarity**: 00 → 01-05 (key flows) → 03 (all) → 04 → 06. Dip into 02 only where a drill exposes a gap.
- **Mid-level engineer new to this repo**: 01-01, 01-05, 03-05 (pattern catalog), 03-06 (critique), 05-05 (security), then 06-02/03.
- **Senior doing architecture review**: 01-01, 03-06, 09-reference/risk-register.md, 05-06.
- **Candidate with an interview in two weeks**: go straight to [08-interview-prep/07-two-week-cram-plan.md](08-interview-prep/07-two-week-cram-plan.md); it pulls in everything else on a schedule.

## Learning tracks (the module map)

| Module | What it gives you |
| --- | --- |
| [01-codebase-cartography](01-codebase-cartography/README.md) | The map: system shape, reading order, glossary, tooling, 7 traced key flows |
| [02-stack-and-language-mastery](02-stack-and-language-mastery/README.md) | TS/Node/React/Next mental models, anchored to real files |
| [03-architecture-and-patterns](03-architecture-and-patterns/README.md) | Boundaries, data model, authz, async reliability, 16 pattern cards, critique |
| [04-code-reading-gym](04-code-reading-gym/README.md) | Annotation drills, trace tables, fake-code contrasts, review katas |
| [05-quality-engineering](05-quality-engineering/README.md) | Testing, debugging, performance, security, observability — as practiced here |
| [06-contribution-practice](06-contribution-practice/README.md) | Junior tickets → mid-level features → senior projects → design katas |
| [07-career-and-collaboration](07-career-and-collaboration/README.md) | Review mindset, PRs/RFCs, maintainer communication |
| [08-interview-prep](08-interview-prep/README.md) | 50+ question cards (mostly repo-anchored), system design, STAR stories, cram plan |
| [09-reference](09-reference/command-cheatsheet.md) | Commands, risk register, rubrics, verification log |

## Conventions used throughout

- **File anchors**: `apps/web/app/lib/pipelines.ts:5-25` means open that file at those lines. Line numbers were confirmed against the working tree on 2026-07-11; if the repo has moved since, search for the named symbol.
- **Fake code**: every snippet not from this repo starts with `// Illustrative fake code: not from this repo`. Everything else is real.
- **Verification labels**: commands are marked __verified__ (actually run during authoring) or __inferred__ (read from scripts/docs but not executed). Behavioral claims that were not fully confirmed are labeled "investigate" or "possible risk" — treat these as hypotheses to test, not known bugs.
- **Drills and self-grading**: most sections end with a drill and a Basic/Solid/Strong rubric. Grade yourself honestly; "Strong" descriptions are what a mid-to-senior candidate sounds like.
- **Interview angle**: sections flag when a concept is common interview material and cross-link into 08-interview-prep.

## Senior vocabulary (also interview vocabulary)

You will meet these words with definitions in context: *invariant* (a condition that must always hold, e.g. "one response per single-use ID"), *boundary* (where responsibility changes hands), *contract* (the shape both sides agreed on), *ownership* (which layer is allowed to change a piece of state), *idempotency* (safe to do twice), *isolation* (tenants/transactions not seeing each other's partial state), *authorization* (may this actor do this to this resource), *consistency* (readers agree with writers — eventually or immediately), *latency*, *observability* (can you tell it broke), *migration*, *rollback*, *blast radius* (how much breaks if this goes wrong). Interviewers listen for these used precisely, not decoratively.

## The mindset ladder

- **Junior asks**: "How do I make it work?"
- **Mid-level asks**: "Is this the right pattern? What breaks it?"
- **Senior asks**: "What does this commit us to, who pays the cost, and how do we reduce risk?"

Interviews for mid-level roles test exactly the second and third questions. When a question card in module 08 shows a "mid-level answer" and a "senior answer," that's the ladder made audible. Climb it by doing the drills, not by reading faster.
