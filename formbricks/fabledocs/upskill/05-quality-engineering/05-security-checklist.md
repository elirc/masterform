# Security Checklist

The threat map of this codebase, each row anchored, ending with the pre-merge checklist. This file pairs with the security cards in [08-interview-prep](../08-interview-prep/README.md) — every row here is an interview answer with evidence.

## Threat-by-threat map

| Threat | Where it applies | Defense in this repo | Grade/notes |
| --- | --- | --- | --- |
| **Authorization gaps / IDOR** | server actions, management API | resolve-up + `checkAuthorizationUpdated` (`action-client-middleware.ts:94-121`); per-env API-key grants (`api/v1/auth.ts:16-43`) | Strong pattern, per-action discipline — audit new actions (Kata 3) |
| **Cross-tenant leakage** | all client APIs | environment equality checks (`utils.ts:18-20`, `storage/route.ts:76-80`, pipeline `:73-79`) | Consistently applied at three layers — the house invariant |
| **Input validation** | every boundary | Zod-first everywhere; `parseAndValidateJsonBody` | Strong |
| **SSRF** | webhook delivery | validate + DNS-pin + manual redirect + timeout (`pipeline/route.ts:96-177`); loud escape hatch env var | Exemplary; also check integration URL handling (Sheets/Notion tokens vs URLs) — investigate |
| **XSS** | survey content rendered on customer pages; response data in dashboard | React/Preact escaping by default; survey content is *attacker-authorable* (a malicious survey creator targets respondents) — check `customHeadScripts` (`schema.prisma:416-417`)! | `customHeadScripts` is by-design script injection for survey owners — the EE `checkExternalUrlsPermission` (`editor/actions.ts:287`) gates related vectors; understand the trust model before calling it a bug |
| **CSRF** | server actions, cookie-auth'd routes | Next.js server-action origin checks; NextAuth cookies; API routes use header keys (immune) | Verify any *custom* cookie-auth'd POST route separately |
| **Injection (SQL)** | Prisma parameterizes | raw SQL only in migrations | Low risk; grep `$queryRaw` before asserting zero |
| **Open redirect** | post-survey `redirectUrl` (`schema.prisma:350`), auth callbackUrl | `callback-url.ts` in auth lib; external-URL permission for surveys | Check how redirectUrl is validated at render time — investigate |
| **Secrets** | env vars | `constants.ts`/`env.ts` centralization; `ENCRYPTION_KEY` for at-rest tokens (2FA codes `authOptions.ts:281-306`, single-use `single-use.ts:23-27`) | No secrets in code found; pnpm postinstall allow-list guards supply chain |
| **User enumeration / timing** | login, forgot-password | control hash constant-time (`authOptions.ts:232-235`); uniform errors; forgot-password rate limit 5/hr | Exemplary |
| **Brute force / DoS** | auth, uploads, API | Lua rate limiter + budgets (`rate-limit-configs.ts`); 128-char bcrypt cap | **Fail-open** (R3) — the known weak edge |
| **File upload abuse** | storage routes | presigned URLs, size caps by license, filename sanitization (`modules/storage/utils.ts`), 5/min mint limit | Check content-type allow-listing depth before trusting |
| **Webhook auth (outbound)** | consumers verifying us | Standard Webhooks signatures when secret set (`pipeline/route.ts:146-154`) | Optional per-webhook — consumers without secrets get unsigned calls |
| **Inbound integration webhooks** | `api/v1/webhooks`, billing | Stripe signature verification (find it in `api/billing` before trusting) | Verify per-provider |
| **Session security** | NextAuth config | cookie flags, session strategy in `authOptions.ts` (read the `session`/`cookies` sections) | Read once; know your answer for "how are sessions stored?" |
| **Audit** | mutations | `withAuditLogging` + EE audit-logs module | Presence is a differentiator; scope is EE |

## The trust-model insight worth an interview minute

Formbricks has an unusual boundary: **survey creators are semi-trusted authors whose content executes in respondents' browsers** (`customHeadScripts`, external URLs, redirects). The EE gates around external URLs/whitelabel exist because the *product* must decide how much a paying org can do to its respondents. When you review features here, ask "does this let a survey author attack a respondent?" — that question is unique to this domain and shows you model threats per-audience, not per-OWASP-list.

## Pre-merge security checklist (use on every PR you write here)

- [ ] New endpoint/action: which of the three auth universes? Session actions use `authenticatedActionClient` + `checkAuthorizationUpdated`; API routes check key grants; public routes justify their publicness in the PR body.
- [ ] Every id from input is resolved up to an owner and checked — no raw `findUnique(id)` → use.
- [ ] Zod schema on every input; no `req.json()` without a parse.
- [ ] Cross-tenant test written (Recipe 4) for anything taking a resource id.
- [ ] New outbound fetch to user-influenced URL → SSRF pattern (validate/pin/no-redirect/timeout) or explicit justification.
- [ ] New secrets via validated env (`env.ts`), never constants; encrypted at rest if stored (follow `symmetricEncrypt` precedent).
- [ ] Errors: public messages generic; details only in logs (match `getUnexpectedPublicErrorResponse`).
- [ ] Rate limit: does this create abusable volume (email sends, uploads, expensive queries)? Add config in `rate-limit-configs.ts`.
- [ ] Migrations: no data exposure via new defaults; PII columns documented.
- [ ] Audit logging on mutations of shared resources.

Drill: run the checklist retroactively against Kata 3's fake PR and against one *real* recent action file of your choice. Two findings minimum on the kata; on real code, expect zero-to-one — write it as a question, not an accusation.
