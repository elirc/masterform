# Learning Rubrics

Observable behaviors, not vibes. Grade yourself per skill; "interview-ready" is the bar for walking into a mid-level loop confident on that row.

| Skill | Junior (you can…) | Mid (you can…) | Senior (you can…) | Interview-ready when… |
| --- | --- | --- | --- | --- |
| Codebase navigation | find a named file via the system map | locate the code for any product behavior unaided in <10 min | predict where code *should* live and spot misplacements | you narrate Flow 1 or 2 without notes, with anchors |
| TS/JS runtime | explain await/event loop abstractly | explain the unawaited-pipeline tradeoff with the serverless caveat | design delivery semantics and defend fail-open/closed per component | 08/01 cards Q1-Q8 at "mid" aloud, 3+ at "senior" |
| React/Next | build a CRUD page | place state correctly (server snapshot vs UI vs derived) and explain the editor's choices | identify state-architecture fixes over memo band-aids; argue the actions-vs-REST split | 08/02 Q1-Q6 at "mid"; the three-cache answer is automatic |
| Data modeling | read the schema | apply the columns-vs-Json rule to a new feature and write the migration plan | design the zero-downtime change incl. the Json-drift twist | you can whiteboard the entity map and defend two constraint choices |
| API design | add an endpoint following a sibling | fill the seven-layer validation table for your endpoint unaided | write the versioning/deprecation policy; spot contract breaks in review | 08/03 Q1-Q7 at "mid" |
| Concurrency | define a race condition | find the quota check-then-act and write the interleaving | choose between four fixes with contention costs; know when to accept the race | Story 2 told in <2 min with atomicity≠isolation stated |
| Async/reliability | draw the pipeline sequence | classify every hop's delivery semantics | design the outbox migration with rollout/rollback | the four-sentence reliability story (03-arch/04) is memorized *and understood* |
| Security | run the pre-merge checklist | catch Kata 3's both blockers; write a cross-tenant test | reason per-audience trust models (author vs respondent vs operator) | you volunteer SSRF/timing-attack examples unprompted in design answers |
| Caching | name the three layers | compute worst-case staleness; justify TTL-only vs invalidation per entity | design the invalidation choke points; call the stampede question | 08/03 Q10 at senior level |
| Testing | write a happy-path unit test | pick the right layer per behavior; write Recipes 2 & 4 green | delete redundant tests with justification; design the kill-9 acceptance test | you have one *real* test you wrote here to talk through |
| Debugging | reproduce and bisect with guidance | run scenarios 1-3 method-first, aloud | conclude "this is architecture" when it is, with evidence | rounds A-D pass 4/5 on the self-score |
| Code review | leave correct comments | rank Blocking/Important/Optional reliably; kind tone | block-with-a-path; persuade toward designs (Kata 8) | review rounds A-B pass, incl. the defense |
| Communication | write the 6-section PR body | write the outbox RFC with fair alternatives | scope disagreements to the smallest defensible change | two STAR stories under 2 min, recorded and reviewed |

## Self-assessment checklist (monthly)

- [ ] I re-derived (not re-read) one key flow this month and updated my notes where the code moved.
- [ ] I completed ≥2 tickets or 1 mid-level ticket, with PR bodies to template.
- [ ] I did ≥2 timed simulations and logged the 5-point score.
- [ ] My risk-register copy has at least one Confidence upgrade/downgrade I earned by testing.
- [ ] I can name my two weakest rows above and my next action for each is scheduled, not aspirational.

The honest failure mode of self-study is grading comprehension as competence. The rubric's antidote: every "you can…" is something you *did*, with an artifact (recording, PR, test, doc) — no artifact, no checkmark.
