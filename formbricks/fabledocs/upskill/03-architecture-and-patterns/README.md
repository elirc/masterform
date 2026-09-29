# 03 — Architecture and Patterns

The judgment module: what the layers are, where state lives, how authorization composes, what happens when things fail — and where this codebase's design is strong, risky, or contested.

1. [01-boundaries-and-layers.md](01-boundaries-and-layers.md) — who owns what, and where the leaks are
2. [02-data-model-and-persistence.md](02-data-model-and-persistence.md) — the schema, the Json bet, transactions, safe schema change
3. [03-validation-auth-and-permissions.md](03-validation-auth-and-permissions.md) — every validation layer; authn vs authz; tenant isolation
4. [04-side-effects-async-and-reliability.md](04-side-effects-async-and-reliability.md) — the pipeline, webhooks, emails, and delivery semantics
5. [05-pattern-catalog.md](05-pattern-catalog.md) — 16 recognition cards
6. [06-architecture-critique.md](06-architecture-critique.md) — strengths, risks, and what I'd change owning this for 3 months (doubles as system-design interview prep)

Exit criteria: you can argue *both sides* of the three big bets (Json survey documents, HTTP-self-call pipeline, TTL-only caching) with anchors.
