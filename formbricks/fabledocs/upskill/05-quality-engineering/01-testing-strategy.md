# Testing Strategy

## The layers this repo actually has

| Layer | Tooling | Where | What it covers |
| --- | --- | --- | --- |
| Unit/service | Vitest, colocated `*.test.ts(x)` | 378 files in apps/web; every package has its own | pure logic, services with mocked Prisma/Redis, route handlers with mocked deps |
| Component | Vitest + testing-library (check imports in any `.test.tsx`) | alongside components | render/interaction of dashboard components |
| Visual | Storybook + Chromatic CI | `apps/storybook`, `chromatic.yml` | shared UI regressions |
| E2E | Playwright | `apps/web/playwright/*.spec.ts` | signup→survey→response journeys, storage smoke, follow-ups |
| Load (targeted) | `rate-limit-load.test.ts` (`modules/core/rate-limit/`) | one-off | limiter under concurrency |
| Static | ESLint, TS, SonarQube | CI | smells, hotspots, types |

What CI runs: `test.yml` (vitest), `e2e.yml` (Playwright), `lint.yml`, `translation-check.yml`, `sonarqube.yml`, plus PR meta-checks (size, semantic title). Contents inferred from filenames — open `.github/workflows/test.yml` once to see exact commands and coverage gates before relying on them.

## What belongs at each layer (the judgment table)

- **Unit**: validation branches (`checkSurveyValidity`'s six outcomes), pure transforms (`calculateTtcTotal`), error mapping (P2002 → 409). If it has interesting *branches*, it's unit-testable — extract until it is.
- **Service-with-mocks**: orchestration order (authz before write in actions), cache-hit vs miss behavior of `withCache` consumers, quota evaluation cases. The repo's `__mocks__` directories (AGENTS.md mentions the convention) hold shared Prisma/logger mocks — reuse them, don't hand-roll.
- **E2E**: only *journeys with integration risk*: publish survey → widget shows it → response lands → appears in dashboard. One good e2e beats twenty asserting button colors.
- **Do not test**: Prisma itself, Zod itself, Next routing, or private helpers already covered through their public caller. Deleting a redundant test is a contribution.

## Isolation techniques in use (find one example of each)

- Mocked module boundaries: `vi.mock("@formbricks/database")` style — check any service test's top matter, e.g. `apps/web/modules/ee/quotas/lib/quotas.test.ts`.
- Test-only escape hatches, honestly labeled: `_syncLocks` in `response-queue.ts:42-52` (`/** @internal Exposed for tests only. */`) — module-level state needs explicit reset hooks; this is the acceptable shape of that compromise.
- Time: look for `vi.useFakeTimers` in queue/retry tests (`response.queue.test.ts`) — retries and TTL windows must not sleep for real.
- Fixtures: Playwright fixtures under `apps/web/playwright/fixtures/` and `utils/` — the e2e suite's account/survey builders.

## Flake prevention rules (derived from what's here)

1. No `waitForTimeout` — wait on conditions (Kata 8).
2. Every test owns its data (e2e creates its own survey; unit tests build inputs via factories/mocks).
3. Module state must have a reset hook if tests touch it (`_syncLocks.clear()` pattern).
4. Clock and randomness injected or faked — uuid/Date.now in assertions is a flake seed.
5. Order-independence: if a test needs another to run first, it's one test written as two.

## Coverage philosophy

`pnpm test:coverage` exists (root `package.json:26`); SonarQube tracks it. Chase *branch* coverage on money paths (response validation, authz middleware) and ignore vanity line-coverage on glue. A useful self-audit: for the seven key flows, name the test file covering each step — the blanks you find are exactly the tickets in [06-contribution-practice](../06-contribution-practice/01-good-first-tickets.md).

Interview angle: "How do you decide what to test?" — answer with the judgment table + one concrete blank you found ("pipeline orchestration is tested at lib level but the route's ordering isn't — here's the test I'd add"). Specific beats philosophy.

Drill: run one suite per layer locally (a package unit suite, one apps/web test file, one Playwright spec — commands in [09-reference/command-cheatsheet.md](../09-reference/command-cheatsheet.md), all __inferred__, verify script names). Note wall-clock time of each; the 100×-cost gradient you just measured is *why* the testing pyramid is shaped like that — now you've felt it, you can say it.
