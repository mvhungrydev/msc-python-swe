# CAP-799 — Capstone

**Credits:** 30 · **Nominal hours:** ~200 · **Prerequisites:** all thirteen courses
**Deliverables:** a system, a written defence, and an oral examination you arrange for yourself

---

## 1. What the capstone is for

Every course in this programme has given you a technique and a build artifact. The capstone asks a
different question:

> **Can you take a problem nobody has decomposed for you, choose the techniques, defend the
> choices, build the thing, evaluate it honestly, and write it up to a standard a stranger would
> accept?**

That is the difference between someone who has completed a curriculum and someone who has a
master's degree. The coursework demonstrated competence at exercises with known answers; the
capstone demonstrates judgement on a problem where nobody tells you what the answer is, or whether
one exists.

Three things it is specifically testing:

1. **Scoping.** Most capstone failures are scoping failures — a project that was too large, or too
   small, or whose difficulty was in the wrong place. Choosing well is the first assessed act.
2. **Judgement under constraint.** You will not have time to do everything the programme taught.
   Choosing what to do properly, what to do adequately, and what to leave undone with a stated
   reason is the skill.
3. **Honest evaluation.** The most common failure in self-directed work is a write-up that reports
   what was built and not what was learned, and that quietly omits the parts that did not work. A
   capstone that reports a negative result rigorously is worth more than one that reports a positive
   result loosely.

## 2. What makes a capstone project suitable

A suitable project has all five of these. Check each explicitly before starting; the check takes an
hour and saves months.

**A question, not just a system.** "Build a distributed cache" is a project. "Does a CRDT-based
cache invalidation scheme reduce staleness enough to justify its complexity against a
changelog-driven one, for this workload?" is a capstone. The system is the apparatus; the question
is the work.

**Genuine difficulty in the right place.** The hard part should be a design or engineering question
this programme has equipped you to reason about — consistency, concurrency, storage layout,
verification, cost, evaluation. If the hard part is learning an unfamiliar API, the difficulty is in
the wrong place.

**Something that can be measured.** You must be able to state what would count as an answer. A
benchmark, a proof, a checked property, an experiment. If you cannot say in advance what evidence
would settle the question, refine the question.

**Achievable in the time.** 200 hours is roughly 12 weeks at 16 hours a week. That is enough for a
substantial system with an honest evaluation, and not enough for a substantial system with a
*thorough* evaluation of everything. Scope to the evaluation, not to the build.

**Something you can defend for an hour.** You will be asked why every significant decision was made
and what the alternatives were. If you would struggle to talk about the project for an hour, it is
either too small or you do not understand it yet.

## 3. What is not suitable

Stated plainly, because these are the projects people choose:

- **A tutorial reimplementation.** Rebuilding Redis by following a guide demonstrates persistence,
  not judgement.
- **A survey.** Reading about ten approaches and writing them up is not a capstone; it is a chapter
  of one.
- **A project whose success is guaranteed.** If you already know the answer, there is nothing to
  find out.
- **A project whose success depends on something outside your control** — an API that might be
  withdrawn, data you have not confirmed you can obtain, hardware you do not have.
- **A project with no evaluation plan.** "I will build it and see" produces a system and no
  findings.
- **Something enormous with a shallow evaluation.** The most common failure mode: three months of
  building, four days of measuring, and nothing established.

## 4. The shape of the work

A 12-week structure that works. Adjust the calendar, keep the proportions.

| Phase | Weeks | Output |
|---|---|---|
| **Proposal** | 1 | The question, the plan, the evaluation design, the risks |
| **Groundwork** | 2–3 | The measurement harness *first*; the skeleton system |
| **Build** | 4–8 | The system, with weekly written notes |
| **Evaluate** | 9–10 | The experiments, run and analysed |
| **Write** | 11–12 | The defence document, then the oral |

Two things about that table matter more than the rest.

**Build the measurement harness before the system.** The single most common capstone failure is
reaching week 10 with a working system and no way to evaluate it. If you cannot measure the naive
version in week 2, you will not measure the sophisticated one in week 10. This is PY-602 L01's
discipline applied to your own project.

**Write weekly.** A paragraph a week on what you did, what surprised you, and what you decided.
Twelve of those paragraphs are the raw material for the defence document, and they capture the
reasoning you will otherwise forget. Use `appendices/learning-log-template.md`.

## 5. The proposal

Write it before starting. Two pages. It is assessed by whether the project survives contact with
it — a proposal that you abandon in week 4 was a bad proposal, and knowing that in week 1 is worth a
month.

1. **The question.** One sentence, in the form "does X, or under what conditions does X".
2. **Why it is not already answered.** What you looked for and what you found. This is a genuine
   literature check, not a formality.
3. **What you will build**, and specifically what the *minimum* system is that can answer the
   question. The minimum, not the ideal.
4. **How you will evaluate it.** The measurements, the baselines (including the trivial one — ML-741
   L07 §2.2 applies to your own work), the statistical treatment, and what result would count as
   answering the question in each direction.
5. **What could go wrong**, and what you would do about each. The most useful section, and the one
   people skip.
6. **What you will not do**, explicitly. The scope boundary is a decision and it should be visible.
7. **The week-by-week plan**, with the week-2 measurement harness marked.

Then do the thing that makes this work: **give the proposal to someone technical and ask them to
find the reason it will fail.** Record what they say.

## 6. The defence document

8,000–12,000 words. Not a report of what you built — an argument for what you found. The structure:

1. **Abstract.** 200 words: the question, what you did, what you found. Written last.
2. **Introduction.** The problem, why it matters, the question, and what this document claims.
3. **Background and related work.** What is already known, with citations. Where your question sits
   relative to it. This section demonstrates that you read before you built.
4. **Design.** What you built and — the assessed part — **why**, with alternatives considered and
   rejected. Use CA-731 L10's review dimensions as the checklist: failure, cost, security,
   operability, and any domain-specific ones.
5. **Implementation.** Enough that a reader could reproduce it. The interesting parts only; not a
   code walkthrough.
6. **Evaluation.** The method, the baselines, the measurements, the statistical treatment, the
   threats to validity. **This is the section that determines the grade.**
7. **Results and discussion.** What you found, what it means, and what it does not mean. Negative
   and null results reported as prominently as positive ones.
8. **Limitations.** What your evidence does not establish. Bounds, assumptions, scale, workload
   representativeness. Be specific — "further work is needed" is not a limitation.
9. **Reflection.** What you would do differently. What the programme prepared you for and what it
   did not. This section is where a self-directed degree earns its credibility.
10. **References.** Primary sources.

Appendices: the proposal, the weekly log, the full results, and instructions to reproduce.

## 7. Standards

The document should meet the standard of a good MSc dissertation, which means:

**Every claim is supported.** By a measurement, a proof, a citation, or an argument. "It is faster"
without a number is not a claim.

**Every measurement has uncertainty.** Confidence intervals, repeated runs, and a stated noise floor
(ML-741 L03 §2.1 applies here as much as to models). A difference smaller than your variance is not
a difference.

**Every design decision has an alternative.** And a reason it was rejected.

**Threats to validity are stated.** What about your setup might make the result not generalise? A
benchmark on one machine, a synthetic workload, a single dataset — name them.

**The limitations section is real.** An examiner reads it first, because it tells them whether you
understand your own work.

**Sources are primary.** The papers, not the blog posts about the papers. The specification, not the
tutorial.

## 8. The oral examination

Arrange one. This is the part self-directed learners skip and it is the part that most demonstrates
mastery.

**Who.** Someone technical who has not worked on the project — a senior colleague, a
domain-experienced friend, a mentor, someone from a relevant community. Give them the document a
week ahead and this section.

**Format.** 60–90 minutes: a 20-minute presentation, then questions.

**Instructions to your examiner**, which you should hand over verbatim:

> Your job is not to be kind. Ask why every significant decision was made and what the alternatives
> were. Push on the evaluation: are the baselines fair, is the measurement sound, does the data
> support the conclusion? Ask what the limitations are and whether the candidate has understated
> them. Ask what they would do differently. Ask at least three questions you expect them not to be
> able to answer, and note how they handle not knowing — "I don't know, and here is how I would find
> out" is the correct answer and a good sign. A candidate who is never at a loss has probably chosen
> too easy a project.

**Afterwards**, write a page: the questions you could not answer, what you would change, and what
the examination revealed about the work. That page goes in the appendix, and it is the most honest
artifact in the whole degree.

## 9. Assessment

Grade yourself against this, honestly, before the oral and again after.

| Criterion | Weight | What distinction looks like |
|---|---|---|
| **Question and scoping** | 15% | A genuine question, well-chosen, achievable, and stated precisely enough to be answered |
| **Technical execution** | 25% | The system works, is well-engineered, and its difficulty is in the right place |
| **Evaluation** | 25% | Fair baselines, sound method, uncertainty quantified, threats to validity stated |
| **Judgement** | 15% | Decisions defended, alternatives considered, trade-offs named |
| **Write-up** | 15% | Clear, complete, honest; a stranger could reproduce and evaluate it |
| **Defence** | 5% | Questions answered or honestly not answered |

The pass condition: **an informed stranger, reading only your document, could tell what you found,
whether to believe it, and what it does not establish.**

## 10. Project catalogue

See `project-catalogue.md` for twenty worked project ideas across the programme's areas, each with
its question, its minimum system, its evaluation design, and its principal risk. They are starting
points; a project of your own devising, arising from a problem you actually have, is better.

## 11. After

Two things worth doing, and both are ways of closing the loop on a self-directed degree.

**Publish something.** A write-up, a talk, an open-source release, a blog post with the measurements
in it. Work that has been exposed to strangers is work that has been tested, and it is the closest
substitute available for peer review.

**Write the honest account of the whole programme.** What you learned, what you skipped, what you
would tell someone starting it. Put it in `appendices/errata.md` or alongside your log. The
programme was built without knowing you; the account is how it improves.

---

*A master's degree is not a certificate. It is the demonstrated ability to take on a problem nobody
has structured for you, choose an approach, defend it, and report honestly on what you found —
including when what you found was that your approach did not work. Everything in the previous six
terms was preparation for doing that once, unaided. Do it once and the qualification is real,
whoever did or did not sign anything.*
