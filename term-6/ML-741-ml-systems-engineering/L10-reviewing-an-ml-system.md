# ML-741 · Lesson 10 — Reviewing an ML System

**Estimated study time:** 4 hours
**Prerequisites:** L01–L09; CA-731 L10

---

## 1. Orientation

CA-731 L10 gave you a method for reviewing a system across failure, cost, security and operability.
This lesson adds the dimensions that are specific to ML systems, and they are the ones ordinary
review misses entirely — because an ML system can pass every conventional review while being
fundamentally unsound.

The additions:

- **Data**, which is source code that no ordinary review looks at.
- **The silent failure surface**, which conventional reliability review does not address because
  the service is up and returning 200s.
- **Loops**, which have no analogue in ordinary systems.
- **Evaluation validity** — whether the team can actually tell if the system works.
- **Harm and fairness**, which are here because the mechanism is technical.

And one question that should be asked before all of them, and rarely is:

> **Should this be a machine learning system at all?** (L01 §2.6.) The most expensive ML failures
> are projects that should not have existed, and by the time a design review happens the answer is
> usually assumed. Ask it anyway.

## 2. Theory

### 2.1 The review dimensions for an ML system

CA-731's four, plus five:

| Dimension | The question |
|---|---|
| Failure | What breaks, how is it detected, how does it recover? |
| Cost | What does it cost now and at 10×? Which term dominates? |
| Security | Trust boundaries, credentials, blast radius. |
| Operability | Can someone on call at 3 a.m. understand and fix it? |
| **Data** | Where does it come from, is it validated, is it versioned, can it be reproduced? |
| **Silent failure** | How would you know if the model quietly became wrong? |
| **Loops** | How does the system's output change its own future input? |
| **Evaluation validity** | Can this team tell whether a change is an improvement? |
| **Harm** | Who is affected by errors, unevenly, and is that measured? |

The nine are not independent. A system with weak evaluation validity cannot assess its own harm
dimension; a system with no loop monitoring will have its evaluation corrupted over time; a system
with no data versioning cannot investigate a silent failure. **Weakness in the ML-specific
dimensions tends to be self-concealing**, which is why they must be asked about explicitly rather
than emerging from a discussion.

### 2.2 The questions, by dimension

Drawn from the whole course. These are the ones that repeatedly find problems.

**Should this be ML?**
- What is the rule-based baseline, and how much better is the model? (L01 §2.6.)
- Is there enough labelled data, representative of the serving distribution?
- Can success be measured? (L07.)
- Who maintains it in three years, and does that team exist?

**Data**
- Where does every feature come from, and who owns each source? (L02.)
- Is the data validated — schema, distribution, volume, nulls? What happens on failure? Does
  training refuse to run?
- Is it versioned such that a model's exact training data is recoverable? (L03.)
- Is point-in-time correctness enforced, and how do you know? (L02 §2.1.)
- Where do labels come from, what is the delay, what is their noise rate, and what bias does the
  labelling process carry?

**Training/serving consistency**
- Is feature computation shared between paths, or reimplemented? (L01 §2.3.)
- Is there a skew test in CI?
- Are serving-time features logged and used for training?

**Reproducibility and provenance**
- Given the production artifact, can you say what produced it, and reproduce it? (L03.)
- Is the noise floor known? Are claimed improvements larger than it?
- Is preprocessing versioned with the model?

**Serving**
- What is the latency decomposition, and what dominates? (L05.)
- Is there batching, and where is the operating point on the latency/throughput curve?
- What is the cold start time, and does the capacity strategy depend on scaling?
- What happens on a missing feature, a timeout, an unavailable model, an absurd output?
- Can the previous model be served immediately, and has the rollback been tested?

**Silent failure**
- **How would you know if the model became wrong?** Ask it exactly this; the answer is diagnostic.
- Is performance monitored, or only inputs? What is the label delay and therefore the detection
  delay? (L06.)
- Are slices monitored? Which ones, and who chose them?
- Are predictions logged with features and joined to labels? What is the join rate?
- What is the drift alert false-positive rate, and does anyone act on the alerts?

**Loops**
- **How do the model's outputs change what data it sees next?** (L08.)
- Are propensities logged? Is the serving policy stochastic?
- Is there exploration? What does it cost, and who agreed to pay it?
- Is action-space coverage or concentration tracked?
- Is there any measurement collected outside the loop?

**Evaluation validity**
- What is the primary metric and how does it relate to the outcome the system exists to affect?
- Is there a trivial baseline in the comparison?
- Are confidence intervals reported? Is the holdout genuinely held out?
- Is there shadow deployment before canary before A/B? (L07 §2.4.)
- Has an A/A test been run on the experimentation platform?

**Harm**
- Who is affected by a false positive, and by a false negative? Are the costs symmetric?
- Is performance measured across affected groups, and is the *trend* in any gap monitored? (L08
  §2.7.)
- Is there recourse — can an affected person contest or understand a decision?
- Is there a human in the loop for consequential decisions, and can that human actually verify?

### 2.3 What good looks like

A short description of a system that would pass, useful as a target:

- Data is contracted, validated, versioned; training refuses to run on data that fails validation.
- Features are defined once and used in both paths, with a skew test in CI.
- Training is reproducible to within a known noise floor, with complete provenance recorded
  automatically, and the reverse lookup from a timestamp to a model version works.
- Serving has a measured latency decomposition, a justified batching operating point, tested
  fallbacks for every failure case, and a rollback that has been exercised.
- Monitoring covers infrastructure, service, data quality, drift and performance by slice, with
  alerts that fire on the right thing and a false-positive rate the team tolerates.
- Predictions are logged with features and joined to labels; the join rate is known and monitored.
- Feedback loops are enumerated, with concentration metrics tracked, propensities logged, and some
  exploration or an out-of-loop measurement.
- Evaluation is offline-to-shadow-to-canary-to-experiment, with a validated experimentation
  platform and a metric connected to the business outcome.
- Harm is measured by group, as a trend, and someone owns the number.

Most production systems fail at more than half of these. That is the normal state, and the review's
job is to produce a *ranked* list of what to fix, not a list of everything wrong.

### 2.4 The ML Test Score as a review instrument

Breck et al.'s rubric (L01 §2.4) gives 28 concrete tests across data, model, infrastructure and
monitoring, scored by whether each is present and automated. Its virtues as a review tool: it is
specific enough to be checkable, it produces a number that can be tracked over time, and it is
externally defined so it is harder to argue with than a reviewer's opinion.

Its limits: it predates the current maturity of feedback loop and fairness practice, it says
nothing about evaluation validity in the L07 sense, and a score can become a target (Goodhart, L08
§2.2). Use it as a floor and this lesson's nine dimensions as the review.

### 2.5 Reviewing the team, not only the system

An uncomfortable but necessary part. Some questions are about capability rather than design, and
they predict outcomes better than the architecture does:

- **Who is on call for this model?** If nobody, it is unowned, and unowned models degrade.
- **When was it last retrained, and by whom, using what process?** If the answer is "a person, by
  hand, in a notebook", the process does not survive that person leaving.
- **What happens when the person who built it leaves?** Is the provenance and documentation
  sufficient?
- **How long from noticing a problem to deploying a fix?** If it is weeks, monitoring has less
  value than it appears to.
- **Has the rollback ever been used?**
- **Does the team know its own noise floor?** A team that does not cannot evaluate its own work.

### 2.6 Writing the review

The structure from CA-731 L10 §2.5, with the ML sections added: data and its lineage, training and
serving consistency, reproducibility and provenance, monitoring and silent failure, evaluation
methodology, feedback loops, and harm analysis.

Two things specific to reviewing ML systems:

**Rank ruthlessly.** Every ML system violates many of §2.3's points. A review listing forty findings
produces no action. Rank by risk × likelihood ÷ cost-to-fix and present five, with the rest in an
appendix.

**Lead with the silent failure question.** If the team cannot answer "how would you know if the
model became wrong?", nothing else in the review matters very much, because they cannot detect
whether any of their other work is holding. That question, asked first, sets the priority for
everything else.

## 3. Construction: review

Build in `mpse/ml741/l10/`. Documentary, like CA-731 L10, and the documents are the deliverables.

**Stage 1 — the design document.** Write the full document for your term artifact ML system, with
CA-731 L10's ten sections plus the five ML-specific ones from §2.1. Use the numbers you measured in
L02–L09 rather than assertions.

**Stage 2 — the ML Test Score.** Score your system on all 28 tests. Report the score. Compare
against the score you got in L01 Stage 5 and account for the difference.

**Stage 3 — the nine-dimension review, on yourself.** Work through §2.2's questions against your own
system and record every one you cannot answer well. Then answer the lead question — "how would you
know if the model became wrong?" — in a paragraph, honestly, including the detection delay.

**Stage 4 — the ranked findings.** Turn Stage 3's list into ranked findings: evidence, risk if
unaddressed, cost to fix, recommendation. Present the top five as you would to an engineering lead,
with the remainder in an appendix.

**Stage 5 — the pre-mortem, ML flavoured.** It is eighteen months from now and the system has caused
a serious problem. Write the incident report in past tense. Then write a *second* one where the
failure was silent — nobody noticed for four months — and identify what would have had to exist to
catch it earlier. The second is the more valuable exercise.

**Stage 6 — review someone else's.** Take a real ML system: a colleague's, an open-source project
with a documented ML component, or a published architecture write-up. Review it with §2.2. Produce
a written review with ranked, labelled concerns. Record which of your concerns rested on a
misunderstanding.

**Stage 7 — be reviewed.** Have someone review your Stage 1 document. Respond to every concern:
accept, reject with reasoning, or defer with an owner. Note which you had already found in Stage 3.

**Stage 8 — the team review.** Answer §2.5's questions honestly about your own system, including
the ones about what happens when you stop working on it. Then write what would have to change for
the answers to be good.

**Stage 9 — the checklist.** Produce a one-page ML system review checklist for your organisation,
adapted from §2.2 and usable by someone who has not taken this course. Use it on a real system and
revise it based on what it missed.

## 4. Failure modes

- **Reviewing an ML system with an ordinary software checklist.** Passes while being unsound.
- **Not asking whether it should be ML.** By review time it is assumed.
- **Reviewing the model and not the data.** The data is the source.
- **Not asking the silent failure question.** The single highest-value question in the review.
- **No loop analysis.** The most consequential ML-specific risk, routinely unexamined.
- **Accepting "we monitor drift" as an answer.** Ask what it detects, what it misses, and whether
  anyone acts on it.
- **Not asking about evaluation validity.** A team that cannot measure improvement cannot improve.
- **Forty unranked findings.** No action results.
- **Reviewing the system and not the team.** Ownership and process predict outcomes better than
  architecture.
- **Treating fairness as out of scope.** The mechanism is technical and the measurement is
  engineering.

## 5. Exercises

### Warm-up (30 min)

1. Give the nine review dimensions and explain why weakness in the ML-specific ones is
   self-concealing.
2. Give the single question to lead with, and explain why the rest of the review depends on its
   answer.
3. Give five questions from the loops dimension.

### Core (3 h)

4. Complete Stages 1–3: the full design document, the ML Test Score with the delta from L01
   explained, and your honest answer to the lead question including the detection delay.
5. Complete Stage 4 and deliver the top five ranked findings.
6. Complete Stage 5, both pre-mortems, with the emphasis on the silent one.
7. Complete Stage 8 and write what would have to change.

### Challenge

8. Complete Stages 6, 7 and 9: reviewing someone else's system with your misunderstanding ratio,
   being reviewed with a recorded response to every concern, and the organisational checklist
   revised after real use.
9. Take a **published ML system** — a paper describing a deployed system, a detailed engineering
   blog post about a production model, or a well-documented open-source ML application — and write a
   full nine-dimension review. Identify: what it does well; what it does not address and whether
   that is a gap or a reasonable scope decision; what the feedback loops are and whether they are
   acknowledged; whether the evaluation described would actually support the claims made; and what
   you would ask the authors. Then, if the system is old enough, check what actually happened to it
   — many published ML systems have public postmortems, deprecations or follow-up papers, and
   comparing your review against the outcome is the best available calibration of your judgement.

## 6. Self-check

1. Give the nine dimensions of an ML system review.
2. Why should "should this be ML?" be asked even at design review?
3. Give five data-dimension questions.
4. What is the lead question, and why does everything else depend on it?
5. Give five loop-dimension questions.
6. What does evaluation validity mean and how do you assess it?
7. What does the ML Test Score give you and what does it miss?
8. Give five questions about the team rather than the system.
9. Why must findings be ranked ruthlessly?
10. Describe what a system that passes this review looks like, in five points.

## 7. Primary sources

- **Breck et al., "The ML Test Score" (IEEE Big Data 2017)** — the rubric; use it.
- **Sculley et al., "Hidden Technical Debt in Machine Learning Systems" (NeurIPS 2015)** — re-read
  it now, at the end of the course; it will read differently.
- CA-731 L10 and its sources — the general review method this lesson extends.
- Mitchell et al., "Model Cards for Model Reporting" (FAT* 2019) and Gebru et al., "Datasheets for
  Datasets" (CACM 2021) — documentation standards worth adopting; both make review much easier.
- Shankar et al., "Operationalizing Machine Learning: An Interview Study" (2022) — what
  practitioners actually struggle with; a good source of review questions.
- Paleyes, Urma & Lawrence, "Challenges in Deploying Machine Learning" (ACM CSUR 2022).
- Selbst et al., "Fairness and Abstraction in Sociotechnical Systems" (FAT* 2019) — why the
  technical and organisational dimensions cannot be separated.
- Klein, "Performing a Project Premortem" (HBR 2007).

---

**Previous:** [L09](L09-llm-systems.md) ·
**Next:** [Problem sets](problem-sets.md) · [Exam](exam.md)

*This completes ML-741. The course began by claiming that a model is a few percent of an ML system
and that the rest is where the engineering is. If the review you just wrote spends most of its
length on the data, the monitoring, the loops and the evaluation — and comparatively little on the
model — then the claim has been demonstrated rather than asserted, which was the point.*
