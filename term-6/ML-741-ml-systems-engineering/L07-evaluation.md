# ML-741 · Lesson 07 — Evaluation: Offline, Online, and the Gap

**Estimated study time:** 4.5 hours
**Prerequisites:** L03, L05, L06

---

## 1. Orientation

The most reliably surprising finding in applied machine learning:

> **Models that are better offline are frequently not better online.** The correlation between
> offline metric improvements and business outcome improvements is weak, and sometimes negative.

Teams discover this the hard way: months of work improving AUC by three points, deployed, and the
business metric does not move — or moves the wrong way. The instinct is to blame the deployment.
The reality is that offline evaluation answers a different question from the one the business
asked, and the gap between them is systematic and explicable.

This lesson is about closing that gap: what offline evaluation can and cannot tell you, how to run
an online experiment that gives a trustworthy answer, and what to do about the specific ways ML
experiments are harder than ordinary A/B tests.

The organising claim:

> **An offline metric is a cheap, fast, biased estimate of a quantity you do not care about. An
> online experiment is an expensive, slow, unbiased estimate of a quantity you do.** Use the first
> to decide what to test, and the second to decide what to ship.

## 2. Theory

### 2.1 Why offline and online disagree

Five distinct reasons, each with its own remedy:

1. **The metric is a proxy.** You measure AUC; you care about revenue, retention or harm avoided.
   The mapping between them is unknown and non-monotonic. A model that is better at ranking may be
   worse at the specific decisions that matter.
2. **The evaluation data is not the serving distribution.** It is historical, it is filtered by
   whatever process collected it, and — critically — it reflects the behaviour of the *previous*
   model (L08). Your test set is a sample from a world your new model will change.
3. **Feedback effects.** The model's predictions change user behaviour, which changes the data. No
   offline evaluation can capture this.
4. **The system is not the model.** Latency, feature availability, fallbacks, thresholds and
   business rules all intervene between the model's output and the user's experience. A model 50 ms
   slower may lose more from abandonment than it gains from accuracy.
5. **Selection bias in the logs.** You only observe outcomes for the actions that were taken. You
   have no data on what would have happened for the ones that were not — which is the
   counterfactual problem that L08 develops properly.

The engineering response to all five: **shadow deployment and online experiments are not optional
extras; they are the only way to answer the question that was asked.**

### 2.2 Offline evaluation, done properly

Offline evaluation is still essential — it is how you filter hundreds of ideas down to the few
worth testing. Doing it well:

**Split by time, not randomly**, for anything with temporal structure (L02 §2.1). And keep a
genuine holdout that is evaluated once — everything you evaluate repeatedly becomes a validation
set, and its estimate becomes optimistic in proportion to how often you looked at it.

**Choose the metric to match the decision.** Accuracy is nearly always wrong for imbalanced
problems. Think about what the model's output is *used for*:

- If a threshold is applied, evaluate at that threshold, with precision and recall at the operating
  point you will actually use.
- If the output is a ranking, use ranking metrics (NDCG, MAP) at the cut-off that matters.
- If the probability itself is consumed — for expected-value calculations, for pricing —
  **calibration matters more than discrimination**, and a well-calibrated worse-AUC model can be
  more useful than a sharply-discriminating badly-calibrated one. Measure it (reliability diagrams,
  expected calibration error), because nothing else will tell you.
- If errors have asymmetric costs, build the cost matrix and evaluate expected cost. This is almost
  always closer to the business question than any standard metric.

**Evaluate on slices** (L06 §2.5), always. An aggregate improvement that harms a segment is not an
improvement, and you will not find out from the headline number.

**Report confidence intervals.** A metric on a test set is an estimate with uncertainty. Bootstrap
it (PY-602 L01). Two models differing by less than the interval are not distinguishable, and much
reported progress is inside the interval.

**Compare against real baselines**: the current production model, a simple rule, and a trivial
predictor (majority class, last value, most popular). **The trivial baseline is the most
informative and the most often omitted** — a great many deployed models do not beat "predict the
most common outcome" by enough to justify their existence, and finding that out early is worth a
quarter.

### 2.3 Online experiments

A randomised controlled trial: users are randomly assigned to control (current model) or treatment
(new model), and the difference in the business metric is measured.

The design decisions that matter:

- **The randomisation unit.** Usually the user, not the request — assigning per request means one
  user experiences both models, which contaminates behavioural metrics and violates independence.
  Randomise at the level at which effects can spill over.
- **The metric hierarchy**: one primary metric decided in advance; a small number of secondary
  metrics; and **guardrail metrics** that must not degrade (latency, error rate, revenue, complaint
  rate). Guardrails catch the experiment that improves its target by damaging something else, and
  they should be non-negotiable.
- **Sample size**, computed before the experiment from the minimum effect you care about, the
  metric's variance, and the desired power. Running until it looks significant is p-hacking, and it
  is the single most common experimental malpractice in industry.
- **Duration**: at least one full weekly cycle, and long enough for novelty effects to decay.
  Users respond to change as change; the first days overstate or understate the durable effect.
- **A/A tests**: run the same model against itself. If it shows a significant difference, your
  experimentation infrastructure is broken, and you would rather learn that now. Every serious
  experimentation platform runs continuous A/A tests, and every team that has not run one has more
  confidence in their platform than they have evidence for.

The statistical hazards, briefly and concretely:

- **Peeking.** Checking repeatedly and stopping when significant inflates the false positive rate
  dramatically. Fix with a fixed sample size, or with sequential testing methods designed for
  continuous monitoring (always-valid p-values, group sequential designs).
- **Multiple comparisons.** Twenty metrics at p < 0.05 gives one false positive by chance. Correct,
  or designate one primary metric.
- **Simpson's paradox.** An effect present in every segment can reverse in aggregate when segment
  sizes differ between arms. Check segment balance.
- **Interference / spillover.** In marketplaces and social products, treating one user affects
  control users, which violates the independence assumption and biases the estimate. Cluster
  randomisation or switchback designs address it imperfectly.
- **Novelty and primacy effects.** Both decay; both mislead early readings.

### 2.4 The gap between "it works" and "it is better"

Three distinct questions, and conflating them is a common source of bad decisions:

1. **Does it work in production?** — shadow deployment. Answers: does it run, is it fast enough,
   are the features available, does it agree with the current model, does it produce sane outputs?
   Zero user risk, and it catches the failures offline evaluation cannot.
2. **Is it safe to expose?** — canary with guardrails and automatic rollback. Answers: does it
   break anything?
3. **Is it better?** — an A/B test on the business metric. **Only this answers the question the
   project exists to answer.**

Most teams do 2 and skip 1 and 3. Doing all three, in order, is the discipline.

### 2.5 Bandits, and when they fit

A multi-armed bandit shifts traffic toward better-performing arms as evidence accumulates, rather
than splitting fixed proportions throughout.

**When bandits fit**: the metric is fast (seconds to hours), the cost of showing the worse arm is
real, there are several arms, and you care about cumulative performance during the experiment more
than about a clean estimate at the end. Content selection and creative optimisation are the natural
cases.

**When A/B fits better**: the metric is slow, you need a defensible unbiased estimate of the effect
size for a decision or a stakeholder, there are exactly two arms, or the analysis needs to be
simple enough that people trust it. Bandit data is *adaptively collected*, which makes standard
statistical inference invalid without correction — a subtlety frequently overlooked, and a real
reason to prefer A/B when the estimate itself is the deliverable.

**Contextual bandits** — choosing per user based on features — are effectively an online learning
system, and they inherit all of L08's feedback loop hazards, immediately and severely.

### 2.6 Evaluating what has no ground truth

Some systems have no clean label: recommendations (you cannot know what the user would have done),
generative output (there is no correct answer), and ranking (relevance is a judgement). The
approaches:

- **Human evaluation** with a written rubric, multiple annotators, and a measured inter-annotator
  agreement. If agreement is low, the rubric is the problem, not the annotators — fix the rubric
  before collecting more labels.
- **Pairwise comparison** rather than absolute rating: humans are much more reliable at "which is
  better?" than at "rate this 1–5", and the results aggregate into a ranking (Elo, Bradley–Terry).
- **Proxy behavioural metrics** — engagement, dwell time, completion — with all of §2.1's caveats
  and the explicit acknowledgement that they are proxies.
- **Counterfactual estimation from logged data** (L08 §2.4), which lets you estimate a new policy's
  performance from data collected under the old one — subject to assumptions that must be checked.
- **Model-based evaluation** (a model judging outputs), which is fast and cheap and must itself be
  validated against human judgement before it is trusted. L09 develops this for LLM systems, where
  it has become standard practice and where its failure modes matter.

The rule regardless of method: **write the evaluation criteria down before looking at the outputs.**
Evaluating generative or ranking output without a rubric produces a judgement about which output
matches the evaluator's mood.

## 3. Construction: evaluate honestly

Build in `mpse/ml741/l07/`, on the L05 serving system and the CA-731 platform.

**Stage 1 — the metric audit.** For your model, write down: what the prediction is used for, what
decision it drives, what the business cares about, and what metric you have been optimising. Then
state the gap. Build the cost matrix for the actual decision and evaluate your model by expected
cost. Compare the ranking of your last five experiments under the old metric and the new one — if
the ranking changes, you have been optimising the wrong thing.

**Stage 2 — the baselines.** Evaluate: the trivial predictor, a hand-written rule, the current
production model, and your model. Report all four with bootstrap confidence intervals. Then state
honestly whether your model's advantage justifies its cost of ownership.

**Stage 3 — calibration.** Produce a reliability diagram and expected calibration error for your
model. Then apply calibration (Platt scaling or isotonic regression, fitted on a held-out set) and
re-measure. Then determine whether calibration matters for your use — it does if a probability is
consumed, and it does not if only a ranking is.

**Stage 4 — slices and intervals.** Evaluate on eight slices with confidence intervals. Find the
slice where your model is worst relative to the baseline. Then determine whether your last model
improvement helped or harmed that slice.

**Stage 5 — shadow deployment.** From L05 Stage 9: route production traffic to both models, log
both predictions, and produce the comparison report — agreement rate, an analysis of the
disagreements (are they concentrated in a slice?), latency comparison, and feature availability
differences. Then find something the shadow deployment revealed that offline evaluation had not.

**Stage 6 — the experiment.** Design an A/B test properly: primary metric, secondary metrics,
guardrails, randomisation unit, minimum detectable effect, computed sample size, and duration.
Write the analysis plan *before* running. Then run it (against simulated users with a defined
underlying effect, so you know the truth) and analyse it according to the plan.

**Stage 7 — the hazards, demonstrated.** With your simulator, produce each: peeking (run 100
simulated A/A experiments with continuous checking and report the false positive rate, then the same
with a fixed sample size); multiple comparisons; Simpson's paradox with unbalanced segments; and a
novelty effect that decays. For each, show the wrong conclusion and the correct method.

**Stage 8 — the A/A test.** Run the same model against itself through your real experimentation
pipeline, twenty times. Report how many showed a "significant" difference at p < 0.05. If it is far
from one in twenty, your pipeline has a bug — find it. This stage has found real bugs in real
companies' experimentation platforms.

**Stage 9 — no ground truth.** Take a task without a clean label (ranking or generation). Write an
evaluation rubric, have two people (or two independent processes) apply it, measure agreement,
revise the rubric, and re-measure. Then build a pairwise comparison harness and compare its
reliability against the absolute rating. Report which produced more consistent judgements.

## 4. Failure modes

- **Optimising a proxy metric without checking it correlates with the outcome.**
- **Random splits on temporal data.** Optimistic and wrong (L02).
- **A holdout evaluated repeatedly.** It is a validation set now.
- **No trivial baseline.** The model may not be beating "predict the most common outcome".
- **No confidence intervals.** Differences inside the noise reported as improvements.
- **Aggregate-only evaluation.** The slice regression is invisible.
- **Ignoring calibration when probabilities are consumed.**
- **Peeking and stopping when significant.** The most common experimental malpractice.
- **Randomising per request rather than per user.** Contaminated behavioural metrics.
- **No guardrail metrics.** The experiment improves its target by damaging something else.
- **No A/A test.** Unvalidated experimentation infrastructure trusted for years.
- **Reading an experiment before novelty decays.**
- **Standard inference on bandit-collected data.** The data is adaptively collected; the inference
  is invalid.
- **Evaluating generative output with no written rubric.** A judgement about the evaluator's mood.

## 5. Exercises

### Warm-up (30 min)

1. Give five reasons offline and online evaluation disagree, with a remedy for each.
2. Give the three distinct questions that shadow, canary and A/B testing answer, and say which most
   teams skip.
3. Explain peeking and why it inflates the false positive rate, and give two correct alternatives.

### Core (3.5 h)

4. Complete Stages 1–2: the metric audit with the experiment re-ranking, and the four baselines
   with intervals plus your honest judgement.
5. Complete Stages 3–4: calibration measured and corrected, with a decision about whether it
   matters; and the slice analysis with your last improvement's effect on the worst slice.
6. Complete Stage 6: a fully specified experiment design with a pre-written analysis plan, executed
   and analysed.
7. Complete Stage 8 — the A/A test — and report the result.

### Challenge

8. Complete Stages 5, 7 and 9: the shadow deployment finding something offline evaluation missed;
   all four statistical hazards demonstrated with wrong and right conclusions; and the rubric-based
   evaluation with the pairwise comparison.
9. Build an **experimentation platform**: randomisation with a hashed assignment that is stable per
   user and independent across experiments, exposure logging, metric computation with variance
   reduction (CUPED, using pre-experiment data to reduce variance — it often halves the required
   sample size and is under-used), sequential testing so continuous monitoring is valid,
   automatic guardrail monitoring with an abort, and an analysis report. Validate it with
   continuous A/A tests and with experiments of known simulated effect size, reporting measured
   power against theoretical power. Then run one real experiment through it and write the decision
   memo — because the platform's output is a decision, and a platform that produces numbers nobody
   can act on has not finished the job.

## 6. Self-check

1. State the relationship between offline and online evaluation in one sentence each.
2. Give five reasons they disagree.
3. When does calibration matter more than discrimination?
4. Why is the trivial baseline the most informative one, and why is it usually omitted?
5. What are guardrail metrics and what do they catch?
6. Why randomise per user rather than per request?
7. Explain peeking, multiple comparisons, Simpson's paradox and interference, each in one sentence.
8. What does an A/A test validate, and what does a failure of one mean?
9. When do bandits fit better than A/B tests, and what statistical complication do they introduce?
10. Give four approaches to evaluating output with no ground truth, and the rule that applies to all
    of them.

## 7. Primary sources

- **Kohavi, Tang & Xu, *Trustworthy Online Controlled Experiments: A Practical Guide to A/B
  Testing* (2020)** — the standard reference; chapters on trust, pitfalls and metrics are the core.
- **Kohavi et al., "Seven Rules of Thumb for Web Site Experimenters" (KDD 2014)** and "Online
  Controlled Experiments at Large Scale" (KDD 2013).
- Deng, Xu, Kohavi & Walker, "Improving the Sensitivity of Online Controlled Experiments by
  Utilizing Pre-Experiment Data" (WSDM 2013) — CUPED.
- Johari, Koomen, Pekelis & Walsh, "Always Valid Inference: Continuous Monitoring of A/B Tests"
  (2017) — the principled answer to peeking.
- Guo, Pleiss, Sun & Weinberger, "On Calibration of Modern Neural Networks" (ICML 2017).
- Niculescu-Mizil & Caruana, "Predicting Good Probabilities with Supervised Learning" (ICML 2005).
- Jeunen, "Revisiting Offline Evaluation for Implicit-Feedback Recommender Systems" (RecSys 2019) —
  the offline/online gap, measured, in a domain where it is severe.
- Bakshy, Eckles & Bernstein, "Designing and Deploying Online Field Experiments" (WWW 2014).

---

**Previous:** [L06](L06-monitoring-and-drift.md) · **Next:**
[L08 — Feedback Loops and Their Hazards](L08-feedback-loops.md)
