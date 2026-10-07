# Runtime and Tooling Map

## Package manager & task runner

- **pnpm** workspaces (`pnpm-workspace.yaml`); note `onlyBuiltDependencies` allow-lists which packages may run postinstall builds (sharp, esbuild, prisma) — pnpm v10 blocks the rest for supply-chain safety. That's a security control worth naming in interviews.
- **Turborepo** (`turbo.json`) orchestrates `build`/`dev`/`test` across the graph with caching. The cache is why `AGENTS.md` tells you to use `--force` when iterating on `packages/surveys` — stale cache + copied bundle = invisible changes.

## Commands that matter (all __inferred__ from `package.json:12-47` + `AGENTS.md`; see cheatsheet for the full table)

| Intent | Command |
| --- | --- |
| Everything up, one shot | `pnpm go` (db:up + all dev servers) |
| Just the app | `pnpm dev` |
| DB services (Postgres, Redis, …) | `pnpm db:up` / `pnpm db:down` (docker-compose.dev.yml) |
| Migrate dev DB | `pnpm db:migrate:dev`; create a migration: `pnpm fb-migrate-dev` |
| Unit tests | `pnpm test` (turbo → vitest per package); `pnpm test:coverage` |
| E2E | `pnpm test:e2e` (Playwright, `playwright.config.ts`) |
| Lint / format | `pnpm lint` / `pnpm format` |
| i18n | `pnpm i18n` (generate + validate translations) |

## Runtime boundaries (the four JavaScripts)

1. **Node server** — Next.js route handlers, server actions, the pipeline. Has secrets, Prisma, Redis. Files often start with `import "server-only"` or `"use server"`.
2. **Dashboard browser** — React 19 client components (`"use client"`), TanStack Query for data fetching (`modules/survey/list/hooks/use-surveys.ts`), react-hook-form. No secrets ever.
3. **Customer-page browser** — `packages/js-core` (SDK) + `packages/surveys` (Preact widget, bundled UMD/ESM into `apps/web/public/js/`). Constraints: tiny bundle, no framework assumptions, survives hostile CSS/JS environments, IndexedDB for offline.
4. **CDN/edge** — no edge *code*, but cache headers (`environment/route.ts:60-75`) make the CDN an active component with its own state.

Sentry configs per runtime: `sentry.server.config.ts`, `sentry.edge.config.ts`, plus `instrumentation.ts`/`instrumentation-node.ts` — observability is runtime-aware.

## Test tooling

- **Vitest** everywhere, colocated `*.test.ts(x)` (378 files in apps/web alone), workspace-wired via `vitest.workspace.ts` reading each package's `vite.config`.
- **Playwright** e2e in `apps/web/playwright/` (signup, surveys, follow-ups, storage smoke...), CI workflow `.github/workflows/e2e.yml`; an Azure-scaled variant exists (`playwright.service.config.ts`, root script `test-e2e:azure`).
- **Storybook + Chromatic** for visual review (`apps/storybook`, `.github/workflows/chromatic.yml`).
- **SonarQube** for smells/hotspots (`sonar-project.properties` — explains the `// NOSONAR` comments you'll meet, e.g. `authOptions.ts:194`).

## CI (`.github/workflows/`, 20 workflows)

The ones that gate PRs: `pr.yml`, `test.yml`, `lint.yml`, `e2e.yml`, `translation-check.yml`, `pr-size-check.yml` (yes — PR *size* is checked; keep diffs small), `semantic-pull-requests.yml` (conventional PR titles). Release/deploy: docker builds, helm chart, `deploy-formbricks-cloud.yml`. If you contribute, read `pr.yml` first to know what will run against you. (Contents inferred from filenames; open them before relying on details.)

## Environment variables (high level, no secrets)

Central registry: `apps/web/lib/constants.ts` + validated env in `apps/web/lib/env.ts` (t3-env style — look for the schema). Load-bearing ones you'll meet in this curriculum: `DATABASE_URL`, `REDIS_URL` (cache + rate limiting; absent = fail-open), `CRON_SECRET` (pipeline auth), `ENCRYPTION_KEY` (single-use links, 2FA backup codes), `WEBAPP_URL`, `RATE_LIMITING_DISABLED`, `DANGEROUSLY_ALLOW_WEBHOOK_INTERNAL_URLS` (SSRF escape hatch), S3 credentials for storage, `SENTRY_DSN`, `POSTHOG_KEY`.

Interview angle: "walk me through your project's toolchain" is a real screen question. A mid-level answer names the tools; a senior answer names the *couplings*: turbo cache vs copied bundles, pnpm script-blocking as supply-chain defense, fail-open behaviors tied to `REDIS_URL`, and PR-size checks as a review-culture signal.

Drill: run `pnpm test --filter=@formbricks/cache` and `pnpm lint` locally (verify script names first); then find in `turbo.json` which tasks depend on `generate` (Prisma client codegen) and explain why build order matters.
