# Mission Learning Path Journal

The missions are ordered from system pulse to ownership judgment. Tier 1 builds navigation muscles with setup, folders, types, components, routes, props, data entry, and routing. Tier 2 connects these pieces into state, hooks, side effects, API contracts, middleware, full-stack traces, diff reading, composition, and TypeScript inference. Tier 3 asks you to reverse-engineer decisions, spot bugs, design tests, and narrate system evolution.

The survey list path was chosen because it crosses the product surface without becoming too broad: route re-export (`apps/web/app/(app)/environments/[environmentId]/surveys/page.tsx:1-4`), server page (`apps/web/modules/survey/list/page.tsx:23-59`), client component (`apps/web/modules/survey/list/components/survey-list.tsx:40-244`), React Query hook (`apps/web/modules/survey/list/hooks/use-surveys.ts:8-51`), API client (`apps/web/modules/survey/list/lib/v3-surveys-client.ts:38-121`), v3 route (`apps/web/app/api/v3/surveys/route.ts:20-82`), shared wrapper (`apps/web/app/api/v3/lib/api-wrapper.ts:345-423`), auth (`apps/web/app/api/v3/lib/auth.ts:79-122`), Prisma pagination (`apps/web/modules/survey/list/lib/survey-page.ts:204-267`), and schema (`packages/database/schema.prisma:344-417`).

A senior engineer would use checkpoints differently at each tier. At junior level, they verify orientation. At mid-level, they ask whether the boundaries are correct. At senior level, they ask what future change would break the system, what would be expensive at scale, and what evidence tests provide.

