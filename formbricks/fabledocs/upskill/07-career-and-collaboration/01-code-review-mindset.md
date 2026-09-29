# Code Review Mindset

## The five layers (review in this order)

1. **Does it work?** — happy path, obvious errors. (Table stakes; CI catches most.)
2. **Is it correct?** — edge cases, concurrency, failure modes. The quota race (Pattern 12) lives here; so does every kata's Blocking finding.
3. **Will it stay correct?** — tests that pin behavior, types that forbid misuse, invariants in the DB not in comments.
4. **Does it fit?** — house patterns: does a new mutation use `authenticatedActionClient` + `checkAuthorizationUpdated`? New cache key through the registry? New user-facing string through `t()`? Fit-violations are future bugs even when currently correct.
5. **Is it kind to future maintainers?** — naming honesty (a `check*` that mutates fails this — `utils.ts:33`), comment quality (the SSRF comments in `pipeline/route.ts:96-104` are the gold standard: they explain *why* and name the attack), diff size.

Ranking discipline: one layer-2 finding outranks ten layer-5 nits. Post them in that order, and label severity explicitly (this repo's katas use Blocking/Important/Optional — keep that vocabulary).

## Repo-specific review checklist

- [ ] Mutation → the five-layer auth stack present (Flow 4)? Count actions vs `checkAuthorizationUpdated` calls.
- [ ] Any id from input resolved up before use (IDOR)?
- [ ] New endpoint → wrapper used (`withV1ApiWrapper`/v2 equivalent), rate-limit config considered?
- [ ] Side effects: post-commit? awaited or deliberately not (with comment)? logged with context?
- [ ] Cache: key via `createCacheKey`, TTL justified, invalidation story stated (even if "TTL-only, because…")?
- [ ] DB: new query rides an index (which?); new constraint has an error mapping; migration additive?
- [ ] Public contract (client API, SDK, webhook payload): additive-only, or versioned?
- [ ] Tests at the right layer; no `waitForTimeout`; module-state reset hooks if needed.
- [ ] i18n `t()` for user-facing strings; `pnpm i18n:validate` will fail CI otherwise (__inferred__).
- [ ] PII in logs? (Response `data`, emails — log ids, not bodies.)
- [ ] Widget/js-core touched → bundle size mentioned? UMD cache-busting implications?

## Example comments (calibrated tone)

Blocking, with path: *"This fetch of a user-supplied URL skips `validateAndResolveWebhookUrl` — that reopens SSRF (see the defense-in-depth at pipeline/route.ts:96-172). Could we route through the same validate+pin helper? Happy to pair on the payload-formatting part if that's the friction."*

Important, as question: *"`getSurvey` here runs before the authz check — is existence-leak acceptable for this resource, or should we reorder? I see both orders in the codebase, so maybe worth a convention note either way."*

Optional, self-aware: *"Nit (feel free to ignore): `rateLimitConfigs` uses `as const` — a `satisfies Record<string, TRateLimitConfig>` would catch shape drift while keeping the literals. Fine as-is."*

Anti-patterns to purge from your reviews: verdicts without reasons ("this is wrong"), style opinions stated as rules, rewriting the author's approach in comments when the approach works, and silence on what's *good* (name one strength per review — it's information, not flattery: it tells the author what to keep).

## Receiving review

Respond to every comment (fix, push back with evidence, or explicitly defer with a ticket). Push-back template: *"I considered that — went this way because [evidence/anchor]. If you feel strongly I'll switch, but wanted to flag [cost]."* Two rounds of disagreement → take it to a call/issue; comment threads radicalize.

Interview angle: code-review rounds grade exactly layers 2–4 plus tone. The phrase that consistently lands: *"I'd rank this one blocking because [failure scenario], these two as follow-ups."* Ranking aloud = seniority signal. Practice on the katas with a timer ([08-interview-prep/05](../08-interview-prep/05-debugging-and-code-review-rounds.md)).
