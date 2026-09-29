# Tier 1 Junior Missions

### Mission 1: The App's Heartbeat
**Tier:** Junior  
**Time Estimate:** 25 minutes  
**Goal:** Explain how the app boots locally and what services it needs.  
**The Concept:** A survey platform is like a research booth: the web app is the booth, Postgres stores answers, MailHog catches outgoing mail, Valkey supports fast state, and RustFS stores uploaded files.  
**Design Intent Before You Read the Code:** Read `package.json`, `.env.example`, `docker-compose.dev.yml`, and `scripts/setup-dev-env.sh`; each owns a different part of local runtime. Bad setup docs create false debugging trails.  
**Find It In The Code:** Open `package.json:18-31`, `.env.example:9-61`, `scripts/setup-dev-env.sh:126-161`, `docker-compose.dev.yml:1-76`.

```bash
# package.json:18-31
pnpm db:up     # starts docker-compose.dev.yml services
pnpm dev       # runs Turbo dev tasks in parallel
pnpm test      # runs workspace Vitest tasks without cache
```

```bash
# scripts/setup-dev-env.sh:139-160
for key in "${REQUIRED_GENERATED_KEYS[@]}"; do
  # Generates ENCRYPTION_KEY, NEXTAUTH_SECRET, and CRON_SECRET when missing.
  upsert_env_value "${key}" "$(openssl rand -hex 32)"
done
```

**The Aha Moment:** Local bugs often start as missing infrastructure, not broken React.  
**Socratic Checkpoint:** What command creates `.env`? Which services run in Docker? Which env var points Prisma to Postgres? Why are generated secrets required? Which script starts Next?  
**How to self-grade:** Strong answers cite `pnpm dev:setup`, Postgres/MailHog/Valkey/RustFS, `DATABASE_URL`, auth/encryption secrets, and `apps/web/package.json:6-9`.  
**Connects To:** Mission 2, because setup only matters once you can navigate the folders.

### Mission 2: The Folder Mental Map
**Tier:** Junior  
**Time Estimate:** 30 minutes  
**Goal:** Describe where routes, features, shared utilities, and database code live.  
**The Concept:** Think of Formbricks like a survey operations center: `app` is the hallway signage, `modules` are teams doing work, `lib` is shared equipment, and `packages/database` is the records vault.  
**Design Intent Before You Read the Code:** Route files should stay small, feature modules should hold product logic, and packages should share cross-app contracts.  
**Find It In The Code:** Open `apps/web/app/(app)/environments/[environmentId]/surveys/page.tsx:1-4`, `apps/web/modules/survey/list/page.tsx:23-59`, `packages/database/src/client.ts:1-20`.

```tsx
// apps/web/app/(app)/environments/[environmentId]/surveys/page.tsx:1-4
import { SurveysPage, metadata } from "@/modules/survey/list/page"; // Route points to module.
export { metadata }; // URL metadata is re-exported.
export default SurveysPage; // Feature implementation lives elsewhere.
```

**The Aha Moment:** In this repo, URL files are often pointers to module-owned features.  
**Socratic Checkpoint:** What owns URL structure? What owns survey list logic? What owns Prisma? Why separate them? What file would you open first for a survey dashboard bug?  
**How to self-grade:** Strong answers mention App Router files, `apps/web/modules/survey/list`, `@formbricks/database`, and route delegation.  
**Connects To:** Mission 8, because routing is the skeleton you will trace later.

### Mission 3: TypeScript Is a Contract
**Tier:** Junior  
**Time Estimate:** 35 minutes  
**Goal:** Explain one runtime schema and the TypeScript type inferred from it.  
**The Concept:** Survey filters are the form on the survey operator's clipboard: if the boxes are defined, both UI and API know what can be checked.  
**Design Intent Before You Read the Code:** Zod validates data at runtime and produces TypeScript types so code and validation stay aligned.  
**Find It In The Code:** Open `apps/web/modules/survey/list/types/survey-overview.ts:1-39`.

```ts
// apps/web/modules/survey/list/types/survey-overview.ts:4-39
export const ZSurveyOverviewSort = z.enum(["createdAt", "updatedAt", "name", "relevance"]); // Allowed sort values.
export const ZSurveyOverviewFilters = z.object({
  name: z.string(),              // Search text is always a string.
  status: z.array(ZSurveyStatus), // Status values reuse shared survey status schema.
  type: z.array(ZSurveyOverviewType),
  sortBy: ZSurveyOverviewSort,
});
export type TSurveyOverviewFilters = z.infer<typeof ZSurveyOverviewFilters>; // TS type comes from runtime schema.
```

**The Aha Moment:** The best TypeScript types describe runtime truth, not wishful thinking.  
**Socratic Checkpoint:** What values can `sortBy` hold? Why infer instead of rewriting a type? What happens if API supports a sort UI does not? Where is status imported from? What field is a list item required to have?  
**How to self-grade:** Strong answers name Zod, `z.infer`, shared status, filter/list item fields, and mismatch risks.  
**Connects To:** Mission 12, because API contracts depend on these shapes.

### Mission 4: Your First React Component
**Tier:** Junior  
**Time Estimate:** 30 minutes  
**Goal:** Understand a small reusable component.  
**The Concept:** A button is a standard survey-control knob: it should look consistent whether it submits, links, or loads.  
**Design Intent Before You Read the Code:** Shared UI components centralize style variants and avoid hand-styling every feature.  
**Find It In The Code:** Open `apps/web/modules/ui/components/button/index.tsx:7-67`.

```tsx
// apps/web/modules/ui/components/button/index.tsx:7-67
const buttonVariants = cva("inline-flex items-center ...", {
  variants: {
    variant: { default: "...", destructive: "...", secondary: "..." }, // Visual intent.
    size: { default: "h-9 px-4 py-2", sm: "h-8 ...", icon: "h-9 w-9" }, // Layout scale.
  },
});

const Comp = asChild ? Slot : "button"; // Lets links receive button styling.
return <Comp disabled={loading || disabled}>{loading ? <Loader2 className="animate-spin" /> : children}</Comp>;
```

**The Aha Moment:** Reusable UI is a contract between product consistency and developer speed.  
**Socratic Checkpoint:** What library builds variants? What does `asChild` do? How is loading rendered? What can go wrong with disabled links? Why forward refs?  
**How to self-grade:** Strong answers identify `cva`, Radix `Slot`, loading state, ref forwarding, and `asChild` caveat.  
**Connects To:** Mission 6, because props define how components are safely reused.

### Mission 5: Your First Node.js Route
**Tier:** Junior  
**Time Estimate:** 20 minutes  
**Goal:** Explain a minimal Next route handler.  
**The Concept:** A health route is the front desk saying "we are open."  
**Design Intent Before You Read the Code:** A route handler maps an HTTP method to a response.  
**Find It In The Code:** Open `apps/web/app/health/route.ts:1-3`.

```ts
// apps/web/app/health/route.ts:1-3
export async function GET() {
  return Response.json({ status: "ok" }); // HTTP GET returns JSON.
}
```

**The Aha Moment:** Next route handlers are just exported HTTP method functions.  
**Socratic Checkpoint:** What URL likely maps to this file? What HTTP method is supported? What status body is returned? Does it need auth? Where would a POST go?  
**How to self-grade:** Strong answers mention `/health`, `GET`, JSON response, no auth, and method exports.  
**Connects To:** Mission 12, because real API routes add contracts around this same idea.

### Mission 6: Props Are a Typed Contract
**Tier:** Junior  
**Time Estimate:** 35 minutes  
**Goal:** Explain how `SurveyCard` protects callers with typed props.  
**The Concept:** Each survey card is a row on the operator's survey clipboard; props say exactly which columns must be filled.  
**Design Intent Before You Read the Code:** Component props should contain all data and callbacks needed to render without hidden global assumptions.  
**Find It In The Code:** Open `apps/web/modules/survey/list/components/survey-card.tsx:15-115`.

```tsx
// apps/web/modules/survey/list/components/survey-card.tsx:15-30
interface SurveyCardProps {
  survey: TSurveyListItem;     // The card cannot render arbitrary objects.
  environmentId: string;       // Needed to build edit/summary links.
  deleteSurvey: (surveyId: string) => Promise<void>; // Parent owns mutation behavior.
  locale: TUserLocale;         // Date formatting must not guess browser locale.
}
```

**The Aha Moment:** Good props make a component boring to use correctly.  
**Socratic Checkpoint:** Which prop decides links? Which prop controls dates? Which prop is a callback? Where is the survey type defined? Why pass `isReadOnly`?  
**How to self-grade:** Strong answers cite `TSurveyListItem`, `environmentId`, `locale`, `deleteSurvey`, and read-only draft behavior.  
**Connects To:** Mission 14, because these props are the final UI stop in the full-stack trace.

### Mission 7: Following Data Into the App
**Tier:** Junior  
**Time Estimate:** 40 minutes  
**Goal:** Trace response submission from request to pipeline event.  
**The Concept:** A survey response enters like a completed paper form: validate the form, check it belongs to this booth, file it, then notify downstream teams.  
**Design Intent Before You Read the Code:** Public endpoints must validate shape, survey rules, enterprise constraints, uniqueness, metadata, and side effects.  
**Find It In The Code:** Open `apps/web/app/api/v2/client/[environmentId]/responses/route.ts:46-80`, `apps/web/app/api/v2/client/[environmentId]/responses/route.ts:98-138`, `apps/web/app/api/v2/client/[environmentId]/responses/route.ts:204-278`.

```ts
// apps/web/app/api/v2/client/[environmentId]/responses/route.ts:204-278
const validatedInput = await parseAndValidateResponseInput(request, params.environmentId); // Shape gate.
const survey = await getSurvey(responseInputData.surveyId); // Domain lookup.
const validationResponse = await validateResponseSubmission(environmentId, responseInputData, survey); // Survey rules.
const createdResponse = await createResponseForRequest({ request, survey, responseInputData, country }); // Persist.
sendToPipeline({ event: "responseCreated", environmentId, surveyId: responseData.surveyId, response: responseData }); // Notify.
```

**The Aha Moment:** Public write paths are mostly gates before the actual write.  
**Socratic Checkpoint:** Where is JSON body validated? Where is survey existence checked? What triggers pipeline events? What metadata is captured? What error returns on duplicate single-use response?  
**How to self-grade:** Strong answers mention input validation, `getSurvey`, `validateResponseSubmission`, `UAParser` metadata, `UniqueConstraintError`, and pipeline events.  
**Connects To:** Mission 22, because public write routes are prime security audit targets.

### Mission 8: Navigation Is the App's Skeleton
**Tier:** Junior  
**Time Estimate:** 30 minutes  
**Goal:** Explain how one URL reaches real feature code.  
**The Concept:** Navigation is the map respondents and operators walk through; route files are the signs, modules are the rooms.  
**Design Intent Before You Read the Code:** Keep URL conventions discoverable while keeping feature code cohesive.  
**Find It In The Code:** Open `apps/web/app/(app)/environments/[environmentId]/surveys/page.tsx:1-4`, `apps/web/modules/survey/list/page.tsx:23-59`, `apps/web/app/(app)/environments/[environmentId]/surveys/layout.tsx:1-8`.

```tsx
// apps/web/app/(app)/environments/[environmentId]/surveys/layout.tsx:1-8
const SurveysLayout = ({ children }: { children: ReactNode }) => {
  return <SurveysQueryClientProvider>{children}</SurveysQueryClientProvider>; // Route segment installs data cache.
};
```

**The Aha Moment:** A route segment can provide infrastructure, not just pages.  
**Socratic Checkpoint:** What does `[environmentId]` mean? Why does the page re-export? What does the layout wrap? Why install QueryClient at this segment? What would break without it?  
**How to self-grade:** Strong answers mention dynamic segments, module delegation, route-local React Query provider, and hook dependency on QueryClient.  
**Connects To:** Mission 9, because state needs a home and this layout provides one.

