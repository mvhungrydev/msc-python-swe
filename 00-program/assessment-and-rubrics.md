# Assessment and Rubrics

You are both the student and the examiner. That is workable, but only if the criteria are
written down in advance and applied without negotiation. This document is the criteria.

## 1. The four instruments

| Instrument | Frequency | What it measures |
|---|---|---|
| **Self-check questions** | Every lesson | Retention of the model |
| **Problem sets** | Every course (3–4 per course) | Application under constraint |
| **Written exam** | Every course | Recall and synthesis without tooling |
| **Term artifact + position paper** | Every term | Judgement and defensibility |

## 2. Self-check questions

At the end of every lesson. Answer *aloud, from memory, without notes.* Speaking is the
test: you will discover which parts of your understanding are word-shaped rather than
model-shaped.

**Pass condition:** you can answer every question and, for each, produce one follow-up
detail that was not in the question. If you cannot, the lesson is not complete. Re-read
the theory section, wait a day, try again.

Do not move to the next lesson with more than one unresolved self-check question behind
you. Debt compounds faster here than anywhere else in the program.

## 3. Problem set rubric

Every problem set is marked on five criteria, 0–4 each, for a maximum of 20.

| Criterion | 0 | 2 | 4 |
|---|---|---|---|
| **Correctness** | Doesn't work | Works on stated cases | Works, with the edge cases identified *before* testing |
| **Justification** | No reasoning given | Asserts a choice | Argues the choice against a named alternative and states the cost |
| **Precision of language** | Vague, uses terms loosely | Mostly correct terminology | Terms used exactly as defined in the course; distinctions honoured |
| **Testing** | None or trivial | Reasonable unit tests | Tests that would have caught the bug you almost wrote; properties where applicable |
| **Economy** | Sprawling, incidental complexity | Reasonable | The smallest thing that could work, and you can say what you left out and why |

**16–20:** distinction. **12–15:** pass. **8–11:** marginal — redo the weakest criterion.
**< 8:** re-study the lesson before continuing.

The criterion people cheat on is **Justification**. Working code with no argument is an
undergraduate deliverable. Graduate work states *why this and not that*.

## 4. Written exam rubric

Each course has `exam.md`: typically 4–6 questions, 3 hours, closed book, no interpreter.
Sit it under real conditions, at a desk, with a timer.

Mark it **one week later**, when you have forgotten what you meant to say. This is
important: your memory of your intention will otherwise mark the paper for you.

Per question, 0–10:

- **0–2** Off-topic or fundamentally confused.
- **3–4** Recalls vocabulary; cannot deploy it. Describes rather than analyses.
- **5–6** Correct standard answer. Reproduces the lesson.
- **7–8** Correct, precise, with an example not given in the lesson, and states at least
  one limitation of the answer.
- **9–10** As above, plus connects the answer to material from a different course, or
  identifies a case where the standard answer is wrong.

**Pass:** ≥ 60%. **Distinction:** ≥ 75% with no question below 5.

A useful calibration: at the master's level, "correct" is a 6, not a 10. The marks above 6
are for *judgement*, and judgement means naming the limits of what you just claimed.

## 5. Term artifact rubric

Each term produces a shipped thing. It is assessed on:

| Dimension | Weight | Test |
|---|---|---|
| **It exists and runs** | 20% | A stranger can clone it and run it from the README alone |
| **Design is argued** | 25% | ADR log or design doc; at least three decisions with rejected alternatives |
| **Verification** | 20% | Tests meaningful, not decorative; you can name what they don't cover |
| **Operability** | 15% | Logging, errors, and failure modes are considered; it can be diagnosed |
| **Restraint** | 10% | You can point to something you deliberately did not build |
| **Written defence** | 10% | A 1,500-word position paper answering the term's driving question |

**Pass:** ≥ 60% overall with nothing at zero. Do not perfect these. Ship at "defensible".

## 6. The position paper

One per term, 1,500 words, answering the term's driving question in your own words with
your own examples.

Structure:

1. **Claim** — one sentence. Falsifiable. Not "concurrency is hard".
2. **Grounds** — the argument, with evidence from your own code or measurements.
3. **Rebuttal** — the strongest case *against* your claim, stated fairly, then answered.
4. **Limits** — where your claim stops being true.

Section 3 is the whole exercise. If you cannot construct a serious opposing case you do
not yet understand the material well enough to have an opinion about it.

Get one competent person to read it. Their job is to attack section 2.

## 7. Oral defence (capstone only)

For CAP-799 you arrange a real defence: 30 minutes presenting, 45 minutes of questions,
with two or three people who know the domain and are willing to be difficult. Colleagues
count. Give them a rubric:

- Does the candidate know the limits of their own system?
- When pressed on a design decision, do they defend it with reasons or with authorship?
- Can they answer "what would you do differently" concretely?
- When they do not know, do they say so?

The last one is the actual test.

## 8. Honesty protocol

Self-assessment fails in one specific way: you mark generously because the alternative is
more work. Three countermeasures:

1. **Write the answer before you check it.** Never grade an answer you formed while
   reading the key.
2. **Grade cold.** A week's delay minimum for exams, a day for problem sets.
3. **Keep the failures visible.** `log/failures.md`: every self-check you could not answer
   and every problem set below 12. Review it at term end. It should be long. A short
   failure log means you are not being honest or not working at the edge of your ability,
   and both are the same problem.

## 9. Progress tracking

`appendices/progress-tracker.md` has a table to fill in. One row per lesson: date studied,
self-check pass/fail, problem set score, notes. Fill it in the same day.

Do not use a fancier system. The tracker's only job is to make stalling visible.
