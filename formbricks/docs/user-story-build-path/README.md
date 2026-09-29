# User Story Build Path

Use these stories as real tickets. Start at Story 1 and do not skip ahead until you can explain what changed, why it changed, how you verified it, and what you would watch in review.

Difficulty progression:

- Stories 1-3 are easy UI and display changes touching one or two files.
- Stories 4-5 add UI features that read existing data and touch several files.
- Stories 6-7 add API or state behavior and require stronger tracing.
- Stories 8-9 are full-stack changes touching UI, backend, and persistence.
- Story 10 is an expert design task with caching, auth, and performance implications.

A story is done when every acceptance criterion is independently verifiable, the touched files match repo patterns, user-facing text uses i18n, dates use shared formatting helpers, tests are updated at the correct layer, and you can explain the change from route to persistence.

Use AI to get unstuck without outsourcing the work:

```text
I'm working on Story X. I'm stuck on Y. Here is what I've tried: Z.
Don't give me the solution. Ask me questions that help me figure it out.
Point me to the files or concepts I should inspect, and challenge my assumptions.
```

