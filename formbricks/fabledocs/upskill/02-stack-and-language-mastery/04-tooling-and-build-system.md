# Tooling and Build System

## The dependency graph you must respect

```
packages/types ──▶ everything
packages/database (prisma generate!) ──▶ apps/web, packages using DB types
packages/survey-ui ──▶ packages/surveys ──build──▶ dist/ ──copied──▶ apps/web/public/js/
packages/js-core ──▶ published to npm as @formbricks/js
packages/cache, logger, storage ──▶ apps/web
```

Two codegen steps make "it doesn't compile" confusing for newcomers:
1. **Prisma client** — `pnpm generate` (turbo task) must run after schema changes; types like `Prisma.TransactionClient` come from generated code, not source.
2. **Survey bundle** — `packages/surveys` builds with Vite to UMD+ESM and the artifact is **copied into `apps/web/public/js/`**; the web app imports from `dist/`, not source (`AGENTS.md`, "Survey Packages Build & Cache").

## The triple-cache trap (memorize this incident-shaped lesson)

Change `packages/surveys/src/...` → nothing happens in the browser. Three caches must all be busted (`AGENTS.md` gives the exact commands):
1. Turbo's build cache (`--force`, or `rm -rf node_modules/.cache/turbo`)
2. The stale copied bundle (`rm -rf packages/surveys/dist apps/web/public/js/surveys.*`)
3. The browser's cached UMD file (hard refresh / disable cache)

Generalizable lesson: **every copy step creates a cache with no invalidation protocol**. When a build pipeline copies artifacts, "my change does nothing" bugs are guaranteed; the fix is either don't copy (import source) or version the artifact (hashed filenames). This repo chose copying for deployment simplicity and documents the workaround. Great interview story about build-system debugging — tell it as "three stacked caches, bisected top-down."

## Turborepo mental model

`turbo.json` declares task → task dependencies (`build` depends on `^build` = dependencies' builds; `generate` before builds needing Prisma). Outputs are content-hashed and cached. Payoff: CI and local runs skip unchanged packages. Cost: stale-cache classes of bugs, hence `--force` in the AGENTS.md recipes. `pnpm test --filter=@formbricks/web` style filters scope any task to one package plus its deps (`--filter=@formbricks/surveys...` = "and everything it depends on").

## Vite in three roles

- Library bundler for `packages/surveys` and `packages/js-core` (UMD/ESM library mode — check each `vite.config.ts`).
- Test runner substrate: **Vitest** configs per package, stitched by root `vitest.workspace.ts` (globs every package's vite config). One command, N isolated projects.
- `packages/vite-plugins` — house plugins; skim to see what's customized.

Next.js itself uses its own toolchain (`next dev --turbopack` per `apps/web/package.json:4`) — so the repo runs Turbopack for the app AND Vite for libraries. Don't conflate Turbo*repo* (task runner) with Turbo*pack* (bundler); interviewers enjoy that confusion.

## Prisma workflow

- Schema: `packages/database/schema.prisma`; migrations in `packages/database/migration/` (note: singular directory name), 100+ timestamped folders since 2023.
- Change flow (__inferred__ from root `package.json:38`): edit schema → `pnpm fb-migrate-dev` (creates migration + regenerates client) → commit both.
- Json-typed columns via the `/// [TypeName]` annotations + `packages/database/json-types.ts` — the bridge between schema.prisma and Zod-land.
- `db:push` exists for prototype-only schema sync (no migration file) — never for shared branches.

## Lint/format/hooks

ESLint via shared `packages/config-eslint`, Prettier preset (110-char, double quotes, import sorting) via `config-prettier`, husky + lint-staged pre-commit (root `package.json:31,54-56`). i18n has its own toolchain: `pnpm i18n` scans/generates translations; CI's `translation-check.yml` enforces it — user-facing strings must go through `t()` (`AGENTS.md`).

## Interview angle

1. "Monorepo tooling — what does Turborepo actually buy you?" → task graph + remote caching; cost = cache-staleness bugs; give the surveys-bundle story.
2. "How do you ship an embeddable widget from a monorepo?" → library-mode Vite build + copy + the cache tax (Pattern 16).
3. "What's your migration workflow?" → the fb-migrate-dev flow; emphasize migration files are code-reviewed artifacts.
4. "Turbopack vs Turborepo?" → two-sentence disambiguation; then stop talking. Knowing when to stop is also being tested.

Drill: from a clean clone (mentally), list the exact commands to get a working dev environment including generated Prisma types, then to see a `packages/surveys` change live. Compare with `AGENTS.md`; anything you missed is a gap in your model, not trivia.
