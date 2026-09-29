# Junior Engineer Guide

## Setup And Run

Local setup is reconstructed from the repository scripts and examples. The README lists Node, pnpm, and Docker as prerequisites (`README.md:134-145`). The root scripts expose `pnpm install`, `pnpm db:up`, `pnpm dev`, `pnpm build`, `pnpm test`, `pnpm test:e2e`, and `pnpm dev:setup` behavior through package scripts (`package.json:18-31`, `package.json:46-48`). The environment template defines `WEBAPP_URL`, `NEXTAUTH_URL`, generated secrets, `DATABASE_URL`, and SMTP defaults (`.env.example:9-61`). The setup script copies `.env.example` to `.env` and generates `ENCRYPTION_KEY`, `NEXTAUTH_SECRET`, and `CRON_SECRET` when missing (`scripts/setup-dev-env.sh:126-161`). Docker starts Postgres, MailHog, Valkey, and RustFS (`docker-compose.dev.yml:1-76`).

Recommended local path:

```bash
pnpm install
pnpm dev:setup
pnpm db:up
pnpm db:migrate:dev
pnpm db:seed
pnpm dev
```

The database package owns migration and seed scripts (`packages/database/package.json:36-44`). The web app runs Next dev on port `3000` with Turbopack (`apps/web/package.json:6-9`).

## Folder Orientation

- `apps/web/app`: Next.js route files, route groups, API handlers, layouts, and public product entry points (`apps/web/app/layout.tsx:24-43`, `apps/web/app/api/v3/surveys/route.ts:20-82`).
- `apps/web/modules`: feature modules such as survey, auth, billing, organization, integrations, and reusable UI (`apps/web/modules/survey/list/page.tsx:1-60`, `apps/web/modules/ui/components/button/index.tsx:1-67`).
- `apps/web/lib`: shared web-app services and utilities, including auth helpers, survey service calls, time formatting, action clients, and environment services (`apps/web/lib/utils/action-client/index.ts:14-65`, `apps/web/lib/survey/service.ts:139-162`).
- `packages/database`: Prisma schema, client singleton, migrations, seed, and package scripts (`packages/database/schema.prisma:4-17`, `packages/database/src/client.ts:1-20`, `packages/database/package.json:32-49`).
- `packages/types`: shared Zod schemas and TypeScript types imported by app and packages, including survey schemas (`apps/web/modules/survey/list/types/survey-overview.ts:1-39`).

## One Level Deeper: Frontend

1. `apps/web/app/(app)`: authenticated product shell. Its layout fetches session and user, blocks inactive users, and installs product-wide client providers/widgets (`apps/web/app/(app)/layout.tsx:18-45`).
2. `apps/web/modules/survey/list`: survey dashboard list feature. It contains server page, client list, hooks, API client, filtering, pagination, and tests (`apps/web/modules/survey/list/page.tsx:23-59`, `apps/web/modules/survey/list/hooks/use-surveys.ts:8-51`).
3. `apps/web/modules/ui/components`: shared UI primitives like `Button`, tooltip, status indicators, page wrappers, and navigation pieces (`apps/web/modules/ui/components/button/index.tsx:7-67`, `apps/web/modules/ui/components/survey-status-indicator/index.tsx:13-98`).

## One Level Deeper: Backend

1. `apps/web/app/api`: Next route handlers. v3 survey list and delete routes show the current API pattern (`apps/web/app/api/v3/surveys/route.ts:20-82`, `apps/web/app/api/v3/surveys/[surveyId]/route.ts:10-72`).
2. `apps/web/app/api/v3/lib`: shared API pipeline: auth mode, schema parsing, rate limit, audit log, request id, and response helpers (`apps/web/app/api/v3/lib/api-wrapper.ts:26-58`, `apps/web/app/api/v3/lib/api-wrapper.ts:345-423`, `apps/web/app/api/v3/lib/response.ts:26-59`).
3. `apps/web/modules/survey/list/lib`: service-like query helpers that own survey list filtering and pagination over Prisma (`apps/web/modules/survey/list/lib/survey-page.ts:155-223`, `apps/web/modules/survey/list/lib/survey-record.ts:5-55`).

## Frontend Entry Point Walkthrough

```tsx
// apps/web/app/layout.tsx:24-43
const RootLayout = async ({ children }: { children: React.ReactNode }) => {
  const locale = await getLocale(); // Server-side source of truth for display language.

  return (
    <html lang={locale} translate="no">
      <body className="flex h-dvh flex-col transition-all ease-in-out">
        <NoScriptWarning locale={locale} /> // Product still explains failure if JS is disabled.
        <SentryProvider
          sentryDsn={SENTRY_DSN}
          sentryRelease={SENTRY_RELEASE}
          sentryEnvironment={SENTRY_ENVIRONMENT}
          isEnabled={IS_PRODUCTION}>
          <I18nProvider language={locale} defaultLanguage={DEFAULT_LOCALE}>
            {children} {/* Every page and route group renders here. */}
          </I18nProvider>
        </SentryProvider>
      </body>
    </html>
  );
};
```

## Backend Entry Point Walkthrough

```ts
// apps/web/app/api/v3/surveys/route.ts:20-82
export const GET = withV3ApiWrapper({
  auth: "both", // Browser session or x-api-key can call this endpoint.
  handler: async ({ req, authentication, requestId, instance }) => {
    const searchParams = new URL(req.url).searchParams; // Read query string from the request URL.
    const parsed = parseV3SurveysListQuery(searchParams); // Convert URL strings into typed filters.
    if (!parsed.ok) return problemBadRequest(requestId, "Invalid query parameters", { invalid_params: parsed.invalid_params, instance });

    const authResult = await requireV3WorkspaceAccess(authentication, parsed.workspaceId, "read", requestId, instance);
    if (authResult instanceof Response) return authResult; // Authorization failure becomes an HTTP response.

    const surveyPagePromise = getSurveyListPage(authResult.environmentId, {
      limit: parsed.limit,
      cursor: parsed.cursor,
      sortBy: parsed.sortBy,
      filterCriteria: parsed.filterCriteria,
    });
    const totalCountPromise = parsed.includeTotalCount
      ? getSurveyCount(authResult.environmentId, parsed.filterCriteria)
      : Promise.resolve(null);

    const [surveyPage, totalCount] = await Promise.all([surveyPagePromise, totalCountPromise]);
    return successListResponse(surveyPage.surveys.map(serializeV3SurveyListItem), {
      limit: parsed.limit,
      nextCursor: surveyPage.nextCursor,
      totalCount,
    }, { requestId, cache: "private, no-store" });
  },
});
```

## TypeScript Orientation

- Zod schemas define runtime validation and TypeScript types together: `ZSurveyOverviewFilters` validates survey filters, while `TSurveyOverviewFilters` is inferred from it (`apps/web/modules/survey/list/types/survey-overview.ts:1-39`).
- Prisma selected payload types keep query results aligned with selected fields: `surveySelect` defines selected columns, and `TSurveyRow` is inferred from it (`apps/web/modules/survey/list/lib/survey-record.ts:5-22`).
- Generic wrappers carry parsed schema types into handlers: `TV3ParsedInput` maps Zod schemas to handler input types (`apps/web/app/api/v3/lib/api-wrapper.ts:28-58`).

## React Component Anatomy 1: Simple Button

```tsx
// apps/web/modules/ui/components/button/index.tsx:7-67
const buttonVariants = cva("inline-flex items-center justify-center ...", {
  variants: {
    variant: { default: "...", destructive: "...", secondary: "..." }, // Design variants are centralized.
    size: { default: "h-9 px-4 py-2", sm: "h-8 rounded-md px-3 text-xs", icon: "h-9 w-9" },
    loading: { true: "cursor-not-allowed opacity-50" },
  },
});

const Button = React.forwardRef<HTMLButtonElement, ButtonProps>(
  ({ loading, asChild = false, disabled, children, ...props }, ref) => {
    const Comp = asChild ? Slot : "button"; // Allows Button styles on links or custom children.
    return (
      <Comp ref={ref} disabled={loading || disabled} {...props}>
        {loading ? <><Loader2 className="animate-spin" />{children}</> : children}
      </Comp>
    );
  }
);
```

## React Component Anatomy 2: Survey Card

```tsx
// apps/web/modules/survey/list/components/survey-card.tsx:23-115
export const SurveyCard = ({ survey, environmentId, isReadOnly, deleteSurvey, locale }: SurveyCardProps) => {
  const { t } = useTranslation(); // User-facing labels must come from i18n.
  const linkHref = useMemo(() => {
    return survey.status === "draft"
      ? `/environments/${environmentId}/surveys/${survey.id}/edit`
      : `/environments/${environmentId}/surveys/${survey.id}/summary`;
  }, [survey.status, survey.id, environmentId]); // Derived navigation depends only on these three values.

  const CardBody = (
    <div className="grid w-full grid-cols-8 ...">
      <div className="truncate">{survey.name}</div>
      <SurveyStatusIndicator status={survey.status} /> {/* Status rendering is delegated. */}
      <div>{survey.responseCount}</div> {/* Count came from grouped response query. */}
      <div>{formatDateForDisplay(survey.createdAt, locale)}</div> {/* Locale is explicit, not browser default. */}
    </div>
  );

  return isReadOnly && survey.status === "draft" ? CardBody : <Link href={linkHref}>{CardBody}</Link>;
};
```

## React Component Anatomy 3: Survey List

```tsx
// apps/web/modules/survey/list/components/survey-list.tsx:55-124
useEffect(() => {
  const storedFilters = globalThis.window.localStorage.getItem(FORMBRICKS_SURVEYS_FILTERS_KEY_LS);
  const parsedFilters = parseStoredSurveyFilters(storedFilters, currentProjectChannel);
  if (storedFilters && !parsedFilters) {
    globalThis.window.localStorage.removeItem(FORMBRICKS_SURVEYS_FILTERS_KEY_LS); // Bad old state is discarded.
    setSurveyFilters(initialFilters);
  } else if (parsedFilters) {
    setSurveyFilters(parsedFilters); // Good old state restores the dashboard view.
  }
  setIsFilterInitialized(true); // Query waits until filters are known.
}, [currentProjectChannel]);

const query = useSurveys({
  workspaceId: environment.id,
  limit: surveysPerPage,
  filters: normalizedFilters,
  enabled: isFilterInitialized, // Prevents a fetch with half-initialized state.
});
```

## Backend Route Anatomy 1: Health

```ts
// apps/web/app/health/route.ts:1-3
export async function GET() {
  return Response.json({ status: "ok" }); // Minimal route handler for uptime checks.
}
```

## Backend Route Anatomy 2: V3 Survey List

```ts
// apps/web/app/api/v3/surveys/route.ts:49-68
const surveyPagePromise = getSurveyListPage(environmentId, { limit, cursor, sortBy, filterCriteria });
const totalCountPromise = includeTotalCount
  ? getSurveyCount(environmentId, filterCriteria)
  : Promise.resolve(null);
const [surveyPage, totalCount] = await Promise.all([surveyPagePromise, totalCountPromise]); // Data and count run together.

return successListResponse(surveyPage.surveys.map(serializeV3SurveyListItem), {
  limit,
  nextCursor: surveyPage.nextCursor,
  totalCount,
}, { requestId, cache: "private, no-store" });
```

## Backend Route Anatomy 3: Public Response Submission

```ts
// apps/web/app/api/v2/client/[environmentId]/responses/route.ts:204-278
export const POST = async (request: Request, context: Context): Promise<Response> => {
  const validatedInput = await parseAndValidateResponseInput(request, (await context.params).environmentId);
  if ("response" in validatedInput) return validatedInput.response; // Validation failures return early.

  const survey = await getSurvey(validatedInput.responseInputData.surveyId);
  if (!survey) return responses.notFoundResponse("Survey", validatedInput.responseInputData.surveyId, true);

  const validationResponse = await validateResponseSubmission(environmentId, responseInputData, survey);
  if (validationResponse) return validationResponse; // Survey rules protect stored data quality.

  const createdResponse = await createResponseForRequest({ request, survey, responseInputData, country });
  sendToPipeline({ event: "responseCreated", environmentId, surveyId: responseData.surveyId, response: responseData });
  return responses.successResponse({ id: responseData.id, ...quotaObj }, true);
};
```

## Domain Glossary

- Organization: top-level tenant and billing/collaboration container (`packages/database/schema.prisma:655-682`).
- Project or workspace: product/application grouping under an organization, with environments and styling (`packages/database/schema.prisma:617-653`).
- Environment: development or production context where surveys, contacts, action classes, tags, webhooks, integrations, and API-key permissions live (`packages/database/schema.prisma:571-600`).
- Survey: configurable questionnaire with status, blocks, questions, endings, targeting, languages, styling, and response rules (`packages/database/schema.prisma:344-417`).
- Response: submitted survey data, metadata, contact relation, tags, quota links, and uniqueness fields (`packages/database/schema.prisma:147-190`).
- Contact: environment-specific person who may receive/respond to surveys (`packages/database/schema.prisma:126-145`).
- Action class: a tracked event or trigger that can launch surveys; survey editing can create one through an authenticated action (`apps/web/modules/survey/editor/actions.ts:475-518`).
- Segment: saved targeting definition for contacts (`packages/database/schema.prisma:939-960`).
- API key environment permission: per-environment API-key access level (`packages/database/schema.prisma:763-817`).

## Junior Socratic Checkpoint

1. Why is the survey route file only four lines?
2. Where is the first authenticated layout check?
3. Which file turns survey list filters into URL search params?
4. Which file turns database survey rows into list items with response counts?
5. What prevents the first survey list query from running before local-storage filters load?
6. Which schema proves that responses belong to surveys?
7. Where does a public response submission trigger downstream pipeline events?

## How To Self-Grade

Strong answers cite exact paths and include ownership: route files expose URLs, modules own features, hooks own client data fetching, API routes own validation/auth, and Prisma schema owns durable relationships. A strong answer also mentions that the survey list is scoped by environment/workspace, user-facing text uses `t()`, and public response submission validates both input shape and survey rules before persistence.

