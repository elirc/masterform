# Validation, Auth, and Permissions

## Map of every validation layer

| Layer | Mechanism | Anchor | Catches |
| --- | --- | --- | --- |
| Transport shape | Zod `safeParse` on params/body | `responses/route.ts:50-70`; `parseAndValidateJsonBody` (`app/lib/api/parse-and-validate-json-body.ts`) | Malformed JSON, wrong types, bad CUIDs |
| Resource existence | fetch-or-404 | `responses/route.ts:224-227` | Dangling ids |
| Tenancy / scoping | explicit equality check | `responses/lib/utils.ts:18-20`; `storage/route.ts:76-80`; pipeline `:73-79` | Cross-environment access |
| Domain state | status machines | survey must be `inProgress` (`utils.ts:22-26`) | Writes to closed resources |
| Field/business rules | per-question validation | `validateResponseData` (`modules/api/lib/validation.ts`), other-option length (`route.ts:108-122`) | Invalid answers |
| Invariants | DB constraints | `@@unique([surveyId, singleUseId])` `schema.prisma:186` | Races the app can't see |
| Abuse | rate limits | `rate-limit-configs.ts` | Volume attacks |

The order is cost-ordered and blast-radius-ordered. When you add an endpoint, fill this table for it — an empty row is a finding.

## Authentication: three universes

1. **Dashboard humans** — NextAuth sessions. Credentials flow (`authOptions.ts:190-320`) is the hardened reference: IP rate limit → DoS cap → constant-time verify (CONTROL_HASH) → uniform errors → optional TOTP/backup codes (consumed one-time, :298-306). SSO/SAML lives in `modules/ee/sso`.
2. **Machines** — `x-api-key` hashed lookup with per-environment permission grants (`api/v1/auth.ts:16-53`; schema `:773-835`). Authentication ≠ authorization: the route must still match grants to the touched resource.
3. **The widget/public** — *no identity at all*. "Auth" = possession of environmentId + resource-state checks + optional single-use tokens/reCAPTCHA. Correctly modeled as validation, not authentication.

Plus the internal fourth: `CRON_SECRET` shared-secret for the pipeline (`pipeline/route.ts:31-36`) — instance-level trust, one secret, no rotation story visible (risk-register entry).

## Authorization: the composed model

For dashboard mutations, `checkAuthorizationUpdated` (`action-client-middleware.ts:94-121`) evaluates an **OR-list of grants**:
- `organization` grant: membership role ∈ listed roles (owner/manager/member/billing).
- `projectTeam` grant: user's aggregated project permission ≥ minPermission (read < readWrite < manage, weights `:39-48`).
- `team` grant: team role ≥ minPermission (contributor < admin).

Worked example (`updateSurveyAction`, `editor/actions.ts:253-267`): org owner/manager **OR** projectTeam ≥ readWrite. A `member` with no team grant: denied. A member on a team with `manage` on this project: allowed. Draw this until it's reflexive.

On top: **license gates** (can this org use follow-ups/spam-protection at all — `:269-287`) and **audit logging** (`withAuditLogging` wrapper, `:251`) with old/new object capture. Five layers total on one mutation: session → input schema → role/permission → license → audit.

## Tenant isolation and IDOR

The IDOR shape: authenticated user supplies someone else's resource id. Defenses here:
- Server actions: ids in input → resolve *upward* to organizationId (`getOrganizationIdFromSurveyId`) → check membership grants. The resolve-then-check idiom is the isolation mechanism; any action skipping it is vulnerable. Audit trick: grep an action file for `checkAuthorizationUpdated` — count should match exported actions (review kata 3 practices this).
- Client API: environmentId in URL + equality checks against fetched resources.
- Management API: `environmentPermissions` array must be consulted per-route.

What a junior misses vs what a senior checks:
- Junior: "it has auth middleware" → assumes done. Senior: middleware proves *who*; per-resource checks prove *may*; looks for the resolve-up call in every action.
- Junior: validates the happy body shape. Senior: asks what happens with a *valid-shaped* id belonging to another tenant (the only interesting case).
- Junior: sees `strict()` Zod schemas. Senior: notices `checkOrganizationAccess` can return validation errors as a *value* (`action-client-middleware.ts:55-62`) and traces that odd path.
- Junior: trusts the UI hiding buttons. Senior: tests the action/endpoint directly (EE gates especially — UI hides, API must enforce).

## Interview angle

- "AuthN vs authZ" — answer with the three universes + composed grants; 60 seconds, concrete.
- "How do you prevent IDOR in a multi-tenant app?" — resolve-up-then-check idiom with the survey example ([08-interview-prep/03](../08-interview-prep/03-api-and-data-modeling-questions.md) Q5).
- "Where do you put validation?" — the seven-layer table; the senior twist is naming which layer *the DB* owns and why.

Drill: pick `deleteSurveyAction` (or any action in `editor/actions.ts` you haven't read). Before reading: write the grant list you'd expect. Then compare, and check what id it resolves upward from. Any mismatch → say whether it's a bug or your model being wrong (usually the latter — update the model).
Self-grade — Basic: found the checks. Solid: predicted roles correctly. Strong: you can articulate why *delete* might warrant stricter grants than *update* (blast radius) and whether this repo agrees.
