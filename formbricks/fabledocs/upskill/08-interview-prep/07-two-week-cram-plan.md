# Two-Week Cram Plan

For a candidate with a mid-level fullstack interview in ~14 days. Assumes evenings + weekends (~2.5h weekdays, ~5h weekend days). Everything cites curriculum files; nothing here requires having done the eight-week path.

**Daily constants (30 min of every session):** two question cards aloud, timed 90s each, from the rotation below; one STAR story told once.

## Week 1 — Build the evidence base

**Day 1 (Sat, 5h):** 00-fast-track in full — run the app, trace Flows 1–2, do the teach-back. Read [01-cartography/01-system-map](../01-codebase-cartography/01-system-map.md) + [03-domain-glossary](../01-codebase-cartography/03-domain-glossary.md). Evening: STAR story 1 drafted (it's available immediately).
**Day 2 (Sun, 5h):** Flows 3–7 ([05-key-flows](../01-codebase-cartography/05-key-flows.md)) with the drills. Skim the [pattern catalog](../03-architecture-and-patterns/05-pattern-catalog.md) — read cards 1-4, 9, 12 deeply, headline the rest.
**Day 3:** [08/01 JS/TS/Node](01-js-ts-node-deep-dive.md) Q1–Q8 aloud. Annotation drills 1–2.
**Day 4:** 08/01 Q9–Q15. Annotation drills 3–4.
**Day 5:** [08/03 API/data](03-api-and-data-modeling-questions.md) Q1–Q7. Trace table 2 (persistence) filled by hand.
**Day 6 (Sat, 5h):** First full **system-design mock** ([08/04](04-system-design-from-this-repo.md)) — 40 min, recorded, no notes. Review against the step-by-step. Then read [03-architecture/06 critique](../03-architecture-and-patterns/06-architecture-critique.md) — your gaps in the mock are covered there.
**Day 7 (Sun, 3h): CHECKPOINT.** Re-record the 3-minute Flow-1 teach-back and compare to Day 1. Score yourself: Can I narrate Flows 1–3 without notes? Are 20+ cards at "mid" level aloud? Two STAR stories under 2 min? **If any is no, Week 2 mornings repair that before adding new material.**

## Week 2 — Simulate and sharpen

**Day 8:** [08/02 frontend](02-frontend-framework-questions.md) Q1–Q6. Fake-code contrasts 1–5 (write the review comments).
**Day 9:** 08/02 Q7–Q12. **Timed debugging round A** (webhooks, 25 min, aloud) from [08/05](05-debugging-and-code-review-rounds.md).
**Day 10:** 08/03 Q8–Q13. **Timed review round A** (30 min, written then defended aloud to a wall — seriously).
**Day 11:** **Timed debugging round B + C** back-to-back (40 min). Weakest question-family from your tracking → redo those cards.
**Day 12:** Second **system-design mock** — this time a *variation* (pick "guarantee webhook delivery" — it chains Q2+Q11 and the outbox story). STAR stories 2 and 5 rehearsed with the fill-ins you have.
**Day 13 (Sat, 4h):** Full loop simulation, interleaved: 45-min system design + 25-min debugging D + 20-min review C + 30-min behavioral (four stories). One sitting, real breaks, recorded.
**Day 14 (Sun, 2h): CHECKPOINT + taper.** Listen to Day-13 recordings; write three index cards of *your own best sentences* (the ones that sounded senior — you'll reuse them live). Re-read only: the golden rule (08/README), your risk-ledger sentences (critique file), the reliability paragraph in [03-architecture/04](../03-architecture-and-patterns/04-side-effects-async-and-reliability.md). No new material. Sleep.

## Card rotation for the daily 30 minutes

Weight by round likelihood: 08/01 (×3/week), 08/03 (×3), 08/02 (×2), plus every card you graded below "mid" recurring until it isn't. Track on paper: card id → date → J/M/S self-grade.

## If you only have one week

Days 1, 2, 6, 9, 12, 13, 14 in that order. The system-design mock and one debugging round are non-negotiable; raw card count is the first thing to sacrifice — depth on 20 cards beats coverage of 55.

## The night before

Nothing new. One teach-back aloud (Flow 1), your three index cards, and the golden rule: **example + tradeoff + failure mode, every answer.** You've studied a production system most candidates haven't — lead with it.
