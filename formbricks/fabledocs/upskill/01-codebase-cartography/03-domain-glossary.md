# Domain Glossary

The product nouns, with code locations. Multi-tenant SaaS vocabulary is interview vocabulary — several of these (environment, display, segment) mean something different here than in generic dev-speak.

| Term | Means here | Code home | Watch out |
| --- | --- | --- | --- |
| **Organization** | Top tenancy unit; owns billing + members | `schema.prisma:667-717` | Billing lives in a Json `OrganizationBilling`-typed column relationship (`:691`) |
| **Project** | Product/workspace inside an org; holds styling defaults, languages | `schema.prisma:628-665` | Called `workspaceId` in some analytics code (`editor/actions.ts:298`) — same thing, two names |
| **Environment** | *Production or development* variant of a project; the unit all client APIs key on | `schema.prisma:582-626` | NOT an env var, NOT a deploy environment. Every widget request carries `environmentId` in the URL |
| **Survey** | The form itself: blocks, endings, logic, styling, all as Json | `schema.prisma:344-421` | `type`: `link` \| `app` \| `website` changes which flows apply |
| **Block / Element / Question** | Survey structure units. Blocks contain elements; older code says "questions" | `Survey.blocks` Json (`:361`), helpers `getElementsFromBlocks` (`apps/web/lib/survey/utils.ts`) | Mid-migration vocabulary: `questions` column still exists (`:359`) alongside `blocks` — read both before assuming |
| **Ending** | Post-submit screen (thank-you, redirect); quotas can force a specific ending | `Survey.endings` (`:363`), `SurveyQuota.endingCardId` (`:429+`) | |
| **Response** | One person's answers; `finished` marks completion; partial responses are real rows | `schema.prisma:158-190` | `data` is Json keyed by element id |
| **Display** | The *event of showing* a survey to someone (impression), whether or not they answer | `schema.prisma:239-262` | One display can link to at most one response (`Response.displayId @unique`) |
| **Contact** | A known end-user (identified via SDK `setUserId`/attributes) | `schema.prisma:134-157` | EE-gated at submission time (`responses/route.ts:82-96`) |
| **Contact Attribute (Key)** | Key/value traits on contacts, used for targeting | `schema.prisma:68-133` | |
| **Segment** | Saved filter over contacts (targeting audiences) | `schema.prisma:947-968` | Segments can be private per-survey |
| **Action Class** | A trackable event (code or no-code) that can trigger in-app surveys | `schema.prisma:499-536` | "Action" in SDK-land = user behavior, not server action! |
| **Trigger** | Joins Survey ↔ ActionClass: "show when this action fires" | `schema.prisma:263-297` | |
| **Single-use link** | Link-survey URL with encrypted `suId`/`suToken` allowing exactly one response | `single-use.ts:14-80`, `Survey.singleUse` Json (`:395`) | Uniqueness enforced by DB (`:186`) |
| **Quota** | Response-count cap with conditions; can screen out or end survey | `schema.prisma:429-471`, `modules/ee/quotas/` | EE. `ResponseQuotaLink.status`: screenedIn/screenedOut |
| **Follow-up** | Automated email sent after a response, addressed statically or from an answer | `schema.prisma:472-498`, `modules/survey/follow-ups/` | EE-permissioned |
| **Webhook** | Customer-configured HTTP callback on response events | `schema.prisma:43-66`, delivery in pipeline route | Standard-Webhooks signed if secret set |
| **Integration** | Built-in third-party sync (Sheets, Airtable, Notion, Slack) | `schema.prisma:537-561`, `pipeline/lib/handleIntegrations.ts` | Config is a Json blob per type |
| **Pipeline** | Internal event fan-out: responseCreated/responseFinished → side effects | `app/lib/pipelines.ts`, `api/(internal)/pipeline/` | Not a queue — an HTTP self-call |
| **Environment state** | The cached config payload the SDK fetches (surveys + actionClasses + project) | `environmentState.ts:19-71` | Three cache layers |
| **Membership / Role** | User↔Org with role: owner, manager, member, billing | `schema.prisma:719-743` | Roles are org-level; project access can come via Teams instead |
| **Team / ProjectTeam** | EE: user groups granted per-project permissions (read/readWrite/manage) | `schema.prisma:1010-1070`, `modules/ee/teams/` | The OR-side of `checkAuthorizationUpdated` |
| **API key** | Machine credential with per-environment permission grants | `schema.prisma:773-835`, `api/v1/auth.ts` | Org-only keys exist (`organizationAccess`) |
| **License / EE** | Enterprise feature gate, checked per-organization, cached | `modules/ee/license-check/` | Gates features AND limits (upload size) |
| **Instance** | One self-hosted deployment (telemetry unit) | `apps/web/lib/instance.ts` | |

Near-synonyms that bite:
- **workspace ≈ project** (analytics events say workspace; schema says Project).
- **person / user / contact**: `User` = dashboard account (`schema.prisma:908`); `Contact` = surveyed end-user; older comments say "person" for contact (`:158` comment "person_id optional" migration names).
- **action** = end-user behavior event (ActionClass) in SDK context, but "server action" = Next.js RPC in dashboard context. Disambiguate by directory.
- **question vs element vs block**: treat "element" as current, "question" as legacy alias — verify per-file which vocabulary it uses.

Drill: close this file and write the containment chain from Organization down to Response, including where Display and Contact attach. Check against `schema.prisma`. Interview version: "model a multi-tenant survey product" — this table is your answer sheet.
