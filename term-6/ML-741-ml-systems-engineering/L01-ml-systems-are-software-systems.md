# ML-741 · Lesson 01 — ML Systems Are Software Systems (And Worse)

**Estimated study time:** 4 hours
**Prerequisites:** SE-521 (all), DI-721 L10

---

## 1. Orientation

The picture most people carry of an ML system is: data goes in, a model is trained, the model makes
predictions. The model is the interesting part.

The picture that matches reality is Sculley et al.'s famous diagram: a small box labelled "ML code"
surrounded by much larger boxes labelled configuration, data collection, feature extraction, data
verification, machine resource management, analysis tools, process management tools, serving
infrastructure, and monitoring. **The model is a few percent of the system.**

That observation is not a complaint about glamour. It is a claim about where the engineering effort
and the failures are, and it is well supported: surveys of deployed ML systems consistently find
that the dominant causes of failure are data problems, infrastructure problems, and organisational
problems — not modelling problems.

This lesson establishes the specific ways ML systems differ from ordinary software, because those
differences generate every subsequent lesson. The central one:

> **In ordinary software, behaviour is specified by code. In an ML system, behaviour is determined
> jointly by code and data — and only one of those has forty years of engineering tooling built
> for it.**

## 2. Theory

### 2.1 The properties that make ML systems different

**Behaviour depends on data.** Two consequences: the data is part of the system's source, so it
must be versioned, reviewed and tested; and a change in the world silently changes your system's
behaviour without anyone deploying anything.

**Correctness is statistical.** There is no assertion that passes. There is a metric, a threshold,
and a distribution of behaviours, which means: you cannot write a unit test that fails when the
model is wrong; "working" is a judgement about a metric on a sample; and the metric you chose is
itself a design decision that encodes what you consider important.

**Failure is silent.** A model given inputs from a distribution it never saw does not raise. It
returns a confident, well-formed, wrong answer. Every other component in your system will accept
that answer and act on it. **This is the single most important operational fact about ML systems**,
and it is why L06 exists.

**Entanglement.** Sculley's **CACE principle**: *Changing Anything Changes Everything.* Because a
model jointly optimises over all its inputs, changing one feature's distribution — improving it,
even — changes the weights on all the others and therefore the model's behaviour everywhere. There
is no modularity within a model, so the usual technique of isolating a change does not apply.

**Feedback loops.** The model's outputs affect the world, which generates the data that trains the
next model (L08). No ordinary component does this.

**The training/serving distinction.** The same logical computation is implemented twice, in
different code, on different data, in different environments. The two diverge, and the divergence is
invisible (§2.3).

### 2.2 The technical debt catalogue

Sculley's paper is a taxonomy, and knowing the names makes the problems findable. The most
consequential:

**Boundary erosion.** ML systems are hard to keep modular, precisely because of entanglement.

**Correction cascades.** A model `A` is not quite right for a new problem, so you build `A'` as a
correction on top of it. Then `A''` on top of that. Now improving `A` breaks everything downstream,
and the system is a stack of corrections nobody can untangle. This is extremely common and the
correct response is almost always to retrain for the new problem rather than to correct.

**Undeclared consumers.** A model's predictions are written somewhere, and other teams start using
them without telling you. You now cannot change the model — its output is a public interface you
never designed and cannot enumerate. This is DI-721 L10's data contract problem with a model in
the middle, and the fix is the same: publish an interface deliberately, and instrument who reads
it.

**Unstable data dependencies.** Your feature depends on another team's output, which changes
without notice. Version the input, or you have coupled your model's behaviour to someone else's
release schedule.

**Underutilised data dependencies.** Features that contribute nothing but must still be computed,
maintained and kept available. They add failure modes and cost with no benefit. Sculley's advice —
leave-one-out evaluation of features, regularly — is rarely followed and always finds something.

**Configuration debt.** ML systems accumulate enormous configuration: features used, data ranges,
hyperparameters, thresholds, verification settings. It is rarely reviewed with the care given to
code, and it changes behaviour just as much. Treat configuration as code: version it, review it,
test it.

**Pipeline jungles.** Data preparation accreted as scrapes, joins and sampling steps added over
years, where recovering an error requires understanding the whole thing. The paper's advice is
blunt and correct: **the jungle can only be avoided by thinking holistically about data collection
and feature extraction, and the cost of a clean rewrite is usually less than the cost of the
jungle's ongoing tax.**

**Dead experimental codepaths.** Branches accumulated from experiments, never removed, each one a
possible interaction. Knight Capital lost $460 million in 45 minutes in 2012 to a dead codepath
that was re-activated by a deployment; the mechanism generalises.

**Glue code and the "plumbing" fraction.** Huge amounts of code exist only to get data in and out
of general-purpose packages. It is where the bugs live and where the maintenance goes.

### 2.3 Training/serving skew

The specific bug class that causes more silent production failures than any other. The same feature
is computed twice: once in the training pipeline (batch, over historical data, in one language and
framework) and once in the serving path (per-request, on live data, often in another). They differ,
and the model sees inputs at serving time that are subtly unlike what it was trained on.

The mechanisms:

- **Different code.** A Python pandas transformation for training and a Java implementation in the
  service. They drift, or were never identical.
- **Different data availability.** A feature computed from a 30-day window in training is computed
  from whatever is in the cache at serving time.
- **Time travel / label leakage.** The training pipeline computes a feature using data that, at
  serving time, would not yet exist. The model appears excellent offline and fails in production.
  This is the most damaging version because it *inflates* offline metrics, so it looks like success.
- **Different defaults.** Missing values imputed one way in training and another in serving.

The mitigations, in descending order of effectiveness:

1. **Compute features once, use them in both paths.** A feature store's actual value is this,
   whatever else is claimed for it (L02).
2. **Share the transformation code** between the two paths, literally the same code.
3. **Log the features computed at serving time** and train on those, so training data is by
   construction what serving produces.
4. **Compare the distributions** of each feature between training and serving continuously, and
   alert on divergence (L06).

The fourth is a detection mechanism rather than a prevention, and every system needs it because the
first three are never completely achieved.

### 2.4 What "testing" means here

Ordinary tests still apply and are still necessary — the pipeline code, the feature transformations,
the serving path are all ordinary software and should have the test discipline of SE-511. But the
model needs additional kinds:

- **Data tests**: schema, ranges, distributions, null rates, cardinalities, and referential
  assumptions. These catch most real failures and are the cheapest to add.
- **Feature tests**: each feature's computation, plus the training/serving equivalence check.
- **Model tests**: performance on a held-out set above a threshold; performance on *slices* (per
  segment, per cohort) so that an improvement in aggregate that harms one group is caught; and
  **behavioural tests** in the CheckList sense — invariance tests (a change that should not affect
  the prediction does not) and directional tests (a change that should move the prediction in a
  known direction does).
- **Infrastructure tests**: training is reproducible, the full pipeline runs end to end, rollback
  works, and the serving path returns the same prediction as the training path for the same input.
- **Monitoring as a test**: the checks that run continuously in production (L06).

The **ML Test Score** (Breck et al.) is a rubric of 28 such tests across data, model, infrastructure
and monitoring. Score your own system on it; most production systems score badly, and the exercise
is uncomfortable and useful.

### 2.5 The lifecycle, and where it differs

Ordinary software: write, test, deploy, monitor, iterate.

ML: define the problem and the metric → collect and label data → build features → train → evaluate
offline → deploy (shadow, canary) → evaluate online → monitor → retrain → repeat.

The differences that matter for engineering:

- **The loop never ends.** A model is not "done"; it decays as the world moves. Retraining is an
  operational obligation, not a project.
- **Offline and online evaluation disagree** (L07), routinely and substantially. A model that is
  better offline is frequently not better online.
- **Deployment is a statistical decision**, not a binary one, so shadow deployments and experiments
  are part of the release process rather than optional extras.
- **Rollback means keeping the previous model and its data**, and being able to serve it — which is
  a real infrastructure requirement people discover during an incident.

### 2.6 When not to use machine learning

A course on ML systems should say this clearly, because the most expensive ML failures are projects
that should not have existed.

ML is the wrong tool when:

- **A rule would do.** If the logic is expressible in a few dozen rules that a human can state,
  write them. They are debuggable, testable, explainable and free. Zinkevich's first rule is "don't
  be afraid to launch a product without machine learning" for exactly this reason.
- **You do not have the data.** Not "we have data" — labelled data, of the right kind, in enough
  volume, representative of the serving distribution. Most projects that fail, fail here.
- **You cannot measure success.** Without a metric that correlates with the outcome you actually
  want, you cannot tell whether the model helps.
- **The cost of a wrong answer is high and unbounded**, and there is no human in the loop or
  containment mechanism.
- **You cannot maintain it.** A model requires ongoing data, retraining, monitoring and expertise. A
  team that cannot commit to that will ship a model that silently degrades to worse than the rule
  it replaced.

The honest framing: **ML adds capability and a large, permanent maintenance obligation.** Both
belong in the decision.

## 3. Construction: audit an ML system

Build in `mpse/ml741/l01/`. Much of this lesson's work is analysis; the code comes from L02
onwards. Use an existing ML system if you have access to one; otherwise build a deliberately naive
one here (a churn or fraud classifier on a public dataset, trained in a notebook and served by a
small Flask app) and audit that — building the naive version first is itself instructive, because
you will make most of the mistakes without trying.

**Stage 1 — the naive system.** Train a model in a notebook, save it as a pickle, and serve it from
an API that computes features inline. Deploy it. This is the system most ML projects actually have,
and everything below is measured against it.

**Stage 2 — the diagram.** Draw the full system: every data source, transformation, storage
location, training step, artifact, and serving component. Then mark the box that is "ML code" and
compute its fraction of the whole. Then mark, for each box, who owns it and what tests it has.

**Stage 3 — the debt audit.** Go through §2.2's catalogue and, for each pattern, state whether your
system has it, with the evidence. Be specific: name the undeclared consumer, name the unstable
dependency, count the configuration parameters that are not under review.

**Stage 4 — training/serving skew, demonstrated.** Deliberately introduce each of the four skew
mechanisms in your naive system, one at a time, and measure the effect on prediction quality against
a correct baseline. The time-travel one is the important one: construct a feature that leaks label
information, show the offline metric improving dramatically, and then show the production
performance being poor. Write up how you would have detected it — the answer involves comparing
offline metric improvements against online results and being suspicious of large ones.

**Stage 5 — the ML Test Score.** Score your system against Breck et al.'s 28 tests. Report the
score honestly. Then implement the five cheapest missing tests and re-score.

**Stage 6 — data tests.** Implement schema, range, distribution, null-rate and cardinality checks on
your input data (Great Expectations, or your own — writing your own first is more instructive). Then
corrupt the data in five distinct ways and verify each check fires. Then find a corruption none of
your checks catch, and add a check for it.

**Stage 7 — behavioural tests.** Write invariance tests (changing an irrelevant field does not
change the prediction) and directional tests (increasing a feature that should increase risk does
increase the score) for your model. Then find one that fails, and decide whether the model is wrong
or your expectation was — this decision is the interesting part, and it will not always come out the
way you expect.

**Stage 8 — the "should this be ML?" analysis.** For your system's problem, write the rule-based
baseline: a handful of thresholds, written by hand. Measure it against your model. Report the gap.
Then compute the total cost of ownership of each — the model's data, retraining, monitoring and
expertise against the rules' maintenance — and state honestly whether ML is justified for this
problem. Sometimes it will not be, and reaching that conclusion with evidence is the exercise.

## 4. Failure modes

- **Treating the model as the system.** The model is a few percent of it.
- **The notebook as production.** Unversioned, untested, unreproducible, and it will be running in
  eighteen months.
- **No data tests.** Most production ML failures are data failures, and they are the cheapest to
  catch.
- **Training/serving skew undetected.** The single most common silent failure.
- **Label leakage.** Inflates offline metrics, so it looks like success until it reaches production.
- **Correction cascades.** A stack of models correcting each other that nobody can change.
- **Undeclared consumers.** Your model's output became an interface without your knowledge.
- **Configuration unreviewed.** It changes behaviour as much as code does.
- **Dead experimental codepaths.** Interactions nobody has considered, waiting to be re-activated.
- **No rollback plan for a model.** Discovered during an incident.
- **Building ML where rules would do.** The most expensive failure, and it happens before any code
  is written.

## 5. Exercises

### Warm-up (30 min)

1. Give five properties that distinguish ML systems from ordinary software, with a consequence of
   each.
2. State the CACE principle and explain why it defeats the usual technique of isolating a change.
3. Give the four mechanisms of training/serving skew and the four mitigations, saying which is
   prevention and which is detection.

### Core (3 h)

4. Complete Stages 1–3: the naive system, the diagram with the ML-code fraction computed, and the
   debt audit with specific evidence per pattern.
5. Complete Stage 4, including the label leakage demonstration and how you would have detected it.
6. Complete Stages 5–6: the ML Test Score before and after, and the data tests including the
   corruption none of them caught.
7. Complete Stage 8 and state, with numbers, whether ML is justified for your problem.

### Challenge

8. Complete Stage 7, including at least one behavioural test that fails and your reasoned decision
   about whether the model or the expectation was wrong.
9. Take a real ML system — at your organisation, or a well-documented open-source one — and write a
   **full technical debt audit** using Sculley's taxonomy plus the ML Test Score. For each finding:
   the evidence, the risk if unaddressed, the cost to fix, and a recommendation. Then rank the
   findings by risk × likelihood ÷ cost and present the top five as you would to an engineering
   lead. The valuable part of this exercise is the ranking: everything in the catalogue is a
   problem, and the skill is knowing which ones to spend a quarter on.

## 6. Self-check

1. What fraction of an ML system is ML code, and what is the rest?
2. Give five properties that make ML systems different from ordinary software.
3. What is entanglement / CACE, and what does it prevent you from doing?
4. Define correction cascades and undeclared consumers, and give the fix for each.
5. What is training/serving skew, and which of its mechanisms is most damaging and why?
6. Why does label leakage look like success?
7. Give the five categories of testing an ML system needs.
8. What are behavioural tests, and what do invariance and directional tests each check?
9. Give three ways the ML lifecycle differs from ordinary software's, with an engineering
   consequence of each.
10. Give five conditions under which ML is the wrong tool.

## 7. Primary sources

- **Sculley et al., "Hidden Technical Debt in Machine Learning Systems" (NeurIPS 2015)** — eight
  pages; read it twice.
- **Breck, Cai, Nielsen, Salib & Sculley, "The ML Test Score: A Rubric for ML Production Readiness"
  (IEEE Big Data 2017)** — use it on your own system.
- **Zinkevich, *Rules of Machine Learning: Best Practices for ML Engineering* (Google)** — 43 rules;
  the first twenty are about not needing ML and about engineering discipline.
- Ribeiro, Wu, Guestrin & Singh, "Beyond Accuracy: Behavioral Testing of NLP Models with CheckList"
  (ACL 2020) — where behavioural testing comes from.
- Amershi et al., "Software Engineering for Machine Learning: A Case Study" (ICSE-SEIP 2019).
- Sambasivan et al., "'Everyone wants to do the model work, not the data work': Data Cascades in
  High-Stakes AI" (CHI 2021).
- Huyen, *Designing Machine Learning Systems*, chapters 1–2.
- Paleyes, Urma & Lawrence, "Challenges in Deploying Machine Learning: A Survey of Case Studies"
  (ACM Computing Surveys, 2022).

---

**Next:** [L02 — Data Pipelines, Features, and Contracts](L02-data-pipelines-and-features.md)
