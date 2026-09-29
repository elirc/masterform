# Command Cheatsheet

All commands from root unless noted. **Every command below is __inferred__** from `package.json:12-53`, `AGENTS.md`, and workspace configs — none were executed during authoring (no Docker daemon on the authoring machine). Verify the first time you run each; correct this file where reality differs.

## Setup & daily

| Intent | Command | Notes |
| --- | --- | --- |
| Install | `pnpm install` | Node/pnpm versions: check root `package.json` engines |
| Env bootstrap | `pnpm dev:setup` | runs `scripts/setup-dev-env.sh` (bash — use Git Bash/WSL on Windows) |
| Services up/down | `pnpm db:up` / `pnpm db:down` | docker-compose.dev.yml: Postgres, Redis, mail, S3-compatible store — read the file for ports |
| Everything | `pnpm go` | db:up + all dev servers (turbo, concurrency 20) |
| App only | `pnpm dev` | web on :3000 (Turbopack) |
| Prisma codegen | `pnpm generate` | needed after schema changes / fresh install |

## Database

| Intent | Command |
| --- | --- |
| Apply migrations (dev) | `pnpm db:migrate:dev` |
| Create migration + regen client | `pnpm fb-migrate-dev` |
| Deploy migrations (prod-style) | `pnpm db:migrate:deploy` |
| Push schema, no migration (prototyping only) | `pnpm db:push` |
| Seed / clear+seed | `pnpm db:seed` / `pnpm db:seed:clear` |

## Test

| Intent | Command |
| --- | --- |
| All unit tests | `pnpm test` (turbo, no cache) |
| One package | `pnpm --filter @formbricks/cache test` |
| One file (web) | from `apps/web/`: `pnpm test <path-fragment>` — verify exact vitest args in `apps/web/package.json` |
| Coverage | `pnpm test:coverage` |
| E2E | `pnpm test:e2e` (Playwright; app+services must be running — check `playwright.config.ts` webServer setting) |
| One e2e spec | `pnpm test:e2e -- survey.spec.ts` |

## Quality

| Intent | Command |
| --- | --- |
| Lint | `pnpm lint` |
| Format | `pnpm format` |
| i18n generate+validate | `pnpm i18n` ; validate-only: `pnpm i18n:validate` |
| Storybook | `pnpm storybook` |

## Build & the surveys-bundle ritual

| Intent | Command |
| --- | --- |
| Build all | `pnpm build` |
| See a packages/surveys change live (from AGENTS.md, verbatim) | `rm -rf packages/surveys/dist apps/web/public/js/surveys.* node_modules/.cache/turbo && pnpm build --filter=@formbricks/surveys... --force` then hard-refresh the browser |
| Clean | `pnpm clean` ; nuclear: `pnpm clean:all` |

## Docker

Dev services: `docker compose -f docker-compose.dev.yml up -d` (what db:up does). Full-app compose files live in `docker/` — read before using.

No codegen beyond Prisma + i18n was found; OpenAPI (`openapi.yml`) generation is wired through the v2 API's `openapi-document.ts` — investigate `modules/api/v2` scripts if you need to regenerate it.
