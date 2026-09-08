# ML-741 · Lesson 08 — Feedback Loops and Their Hazards

**Estimated study time:** 4 hours
**Prerequisites:** L02, L06, L07

---

## 1. Orientation

This is the lesson with no analogue in ordinary software engineering, and the one whose failures are
hardest to detect, slowest to manifest, and most damaging when they do.

> **A deployed model changes the world it observes. Its predictions influence actions, actions
> influence outcomes, and outcomes become the training data for the next model. The system is a
> closed loop, and closed loops have dynamics.**

No ordinary component does this. A sorting function does not change the distribution of things you
ask it to sort. A model that decides which transactions to block changes which transactions you
observe outcomes for; a model that decides what to recommend changes what users see and therefore
what they click; a model that scores loan applications changes who gets loans and therefore whose
repayment behaviour you learn from.

The consequences range from the merely wrong to the seriously harmful, and they share a structure:
**the system converges to a state that its own metrics endorse.** Which means your monitoring
(L06) and your offline evaluation (L07) will both tell you everything is fine.

The two things to take from this lesson: **learn to recognise the loop structures**, and **log the
counterfactual** — because almost every mitigation depends on having data about what would have
happened otherwise, and that data must be deliberately collected before you need it.

## 2. Theory

### 2.1 The structure

The loop:

```
model → prediction → action → outcome → logged data → next model
                        ↓
                   world changes
```

The critical property: **you only observe outcomes for the actions taken.** If the model declined a
loan, you never learn whether it would have been repaid. If the recommender did not show an item,
you never learn whether it would have been clicked. This is the **bandit feedback** problem — you
see the reward for the chosen action only — and it is the root of most of what follows.

Which means the training data for the next model is not a sample from the world. It is **a sample
from the world as filtered by the current model's decisions.**

### 2.2 The named loops

**Selection bias / the rejection problem.** A credit model declines applicants; only accepted
applicants generate repayment data; the next model trains only on accepted applicants and cannot
learn about the region it declined. The model becomes confident about a shrinking region and
ignorant everywhere else, and its offline metrics improve because the data has become easier.
Reject inference is the technique for partially addressing it, and exploration (§2.5) is the
structural answer.

**Confirmation loops / the fraud blind spot.** A fraud model flags suspicious transactions;
analysts review only flagged cases; labels come only from flagged cases; the next model learns
"fraud looks like what we already catch". Fraud the model does not catch is unlabelled, so it never
enters training, and the blind spot is self-perpetuating and invisible. This is the same structure
as the credit case and the same fix.

**Popularity / rich-get-richer.** A recommender shows popular items; shown items get engagement;
engagement makes them more popular; the model shows them more. The catalogue collapses toward a
small set, diversity falls, and the metric (click-through on shown items) rises the whole time.
This is the most common recommender failure and it looks like success on every dashboard.

**Filter bubbles and preference amplification.** The model infers a preference, shows more of it,
the user engages with what is shown, and the inference is confirmed and strengthened. The system
does not discover a preference; it *creates* one, and cannot distinguish the two from its own data.

**Degenerate feedback with proxy metrics.** Optimising for a proxy (clicks, watch time) produces
content selected for that proxy rather than for the underlying value (usefulness, satisfaction).
The proxy improves while the thing it proxied for degrades — which is Goodhart's law with an
optimiser attached, running continuously.

**Direct self-fulfilment.** A model predicts that a customer will churn; the system deprioritises
them; they churn. The prediction was "accurate" and the model caused it.

**Cross-system loops.** Two models influencing each other through the world — a pricing model and a
demand model, or two competing firms' models — producing oscillation or runaway dynamics that
neither system's monitoring can see, because each sees only its own half.

### 2.3 Why detection is hard

Every one of these is invisible to the monitoring built in L06 and the evaluation built in L07,
and it is worth being explicit about why:

- **Offline metrics improve.** The data has become more homogeneous and therefore easier to predict.
- **Online metrics improve.** You are measuring the proxy the loop is optimising.
- **There is no drift signal.** The input distribution changes slowly and smoothly, driven by your
  own system, which looks like ordinary gradual drift.
- **No A/B test catches it** if both arms are inside the loop, or if the effect accumulates over a
  longer period than the experiment.
- **The counterfactual is unobserved by construction.** You have no data about the world your model
  is not creating.

The signals that *do* indicate a loop, and which should be monitored deliberately:

- **Diversity or coverage of the action space falling over time.** The fraction of the catalogue
  ever recommended, the fraction of applications approved, the variety of outcomes.
- **The model's training data becoming increasingly self-similar.**
- **Growing divergence between the proxy metric and any independent measure** — the proxy rises
  while satisfaction surveys, retention, or complaint rates move the other way.
- **Concentration**: a Gini coefficient or entropy over the distribution of actions taken. This is
  the single most useful loop metric and almost nobody tracks it.

### 2.4 Counterfactual reasoning and off-policy evaluation

The formal framing, from Bottou et al.: your logged data was collected under a **logging policy**
(the old model), and you want to evaluate a **target policy** (the new one). The techniques:

- **Inverse propensity scoring (IPS)**: weight each logged outcome by 1/P(the logging policy chose
  this action). Unbiased *if* the logging policy was stochastic and its probabilities were recorded.
  High variance when the propensities are small.
- **Doubly robust estimation**: combine IPS with a model of the reward; unbiased if either the
  propensity model or the reward model is correct. Lower variance, and the practical default.
- **Direct method**: model the reward and use it to evaluate the new policy. Low variance, biased by
  the reward model's errors.

The requirement that all of these depend on, and which must be built in advance:

> **The logging policy must be stochastic, and its action probabilities must be recorded with every
> decision.** A deterministic policy provides no basis for estimating what another policy would have
> done, because there is no variation to learn from.

This is the single most important engineering consequence of the lesson: **log propensities.** It
costs almost nothing at decision time and it is the difference between being able to evaluate a
change from logged data and having to run a live experiment for everything. Teams that did not do
this cannot retrofit it — the data does not exist.

### 2.5 Exploration

The structural fix for the loops in §2.2: deliberately take actions the current model would not
choose, to generate data about them.

- **ε-greedy**: with probability ε, act randomly. Crude, simple, effective, and easy to explain to a
  stakeholder — which matters, because exploration means deliberately doing something suboptimal
  and someone will ask why.
- **Thompson sampling**: sample from the posterior over rewards and act greedily on the sample.
  Elegant, efficient, and naturally produces the stochasticity §2.4 requires.
- **Upper confidence bound**: act optimistically with respect to uncertainty.
- **Random holdout**: a small fraction of traffic served by a random or deterministic non-model
  policy, permanently. Provides an unbiased reference against which the model's real value can be
  measured — which is otherwise unmeasurable once the loop is established.

The exploration cost is real and quantifiable: you are deliberately making some decisions worse. The
argument for paying it is that **without exploration your data collapses onto your current policy
and the system loses the ability to improve or even to know how well it is doing.** Frame it to
stakeholders as the cost of measurement rather than as a loss, because that is what it is.

Where exploration is not acceptable — credit decisions, medical triage, anything where a
deliberately worse decision harms someone identifiable — the honest position is that off-policy
evaluation will be limited, and that limitation should be stated rather than papered over with
methods whose assumptions do not hold.

### 2.6 Design responses

Beyond exploration:

- **Diversity and coverage constraints** in the serving policy: guarantee a fraction of
  recommendations from outside the model's top choices; guarantee a minimum approval rate in some
  region. Direct, blunt, and effective.
- **Multiple objectives.** Optimising one proxy is what makes Goodhart's law bite. Optimise a
  combination — engagement *and* diversity *and* satisfaction — even though it is harder to tune.
- **Independent measurement.** A metric collected outside the loop: a survey, a random-sample human
  evaluation, a holdout population. **A system that only measures itself cannot detect that it is
  drifting away from what it was supposed to do.**
- **Cap the loop's speed.** Retraining daily on data your model generated yesterday tightens the
  loop; retraining monthly with a longer data window loosens it. Loop dynamics depend on the gain
  and the delay, and slowing the cycle is a legitimate control.
- **Human review of the loop's aggregate effects**, periodically, looking at what the system is
  doing to its population rather than at its metrics.

### 2.7 The harm dimension

Feedback loops are the mechanism by which ML systems produce unfair outcomes at scale, and this
belongs in an engineering course because the mechanism is engineering, not ethics-in-the-abstract.

The structure, in predictive policing as the clearest documented case: the model predicts crime
where past arrests occurred; police are deployed there; more arrests occur there because that is
where the police are; the data confirms the prediction. The model's accuracy improves on its own
data while its relationship to actual crime degrades. Lum and Isaac demonstrated this concretely,
and the same structure appears in hiring, credit, content moderation and healthcare resource
allocation.

The engineering points that follow:

- **A model trained on the outcomes of a biased process learns the bias as ground truth**, and the
  loop entrenches it.
- **Aggregate metrics will not show it** (L06 §2.5) — slice monitoring is a technical requirement
  with an ethical consequence.
- **"The model is just predicting what happens" is false when the model influences what happens.**
  This is the sentence to have ready when someone offers it.
- Disparate impact must be measured across affected groups, over time, and specifically as a *trend*
  — a loop's signature is a gap that widens.

The honest scope: this course teaches you to recognise the mechanism and build the measurement.
Deciding what is acceptable is a broader question involving people who are not engineers — and the
engineer's professional obligation is to make the effect visible and to say so plainly when it is
not.

## 3. Construction: build a loop, watch it fail, and fix it

Build in `mpse/ml741/l08/`. Simulation is the right tool here: real loops take months to manifest,
and a simulator lets you run years in minutes and know the ground truth.

**Stage 1 — the simulator.** Build a world with a true underlying process: users with real
preferences, items with real qualities, and a defined relationship between them. Then a system: a
model, a serving policy, logging, and retraining. Because you defined the truth, you can measure
what the system's own metrics cannot.

**Stage 2 — the popularity loop.** Run a recommender that shows top-predicted items and retrains on
observed engagement. Run for 200 simulated cycles. Plot: the system's click-through rate (which
will rise), catalogue coverage (which will fall), and **true user satisfaction against the ground
truth** (which will fall). The three curves on one chart is the lesson's central artifact.

**Stage 3 — the fraud blind spot.** Model a fraud detector where only flagged transactions are
labelled. Introduce a fraud pattern the initial model does not catch. Show that it is never learned,
that the model's measured precision stays excellent, and that true fraud losses rise. Then measure
how long it takes to notice with L06's monitoring — it will be never.

**Stage 4 — the rejection problem.** Model credit decisions where only approved applicants generate
repayment data. Show the approved region shrinking over cycles, and the model becoming confident
and wrong about the excluded region. Measure the opportunity cost against the ground truth.

**Stage 5 — the loop metrics.** Implement the §2.3 signals: action-space coverage, entropy or Gini
concentration, training data self-similarity, and proxy-versus-independent-measure divergence. Run
them against Stages 2–4 and report how early each detects the loop. Compare against L06's drift
detectors, which will detect nothing.

**Stage 6 — exploration.** Add ε-greedy exploration to Stage 2 at ε = 0.01, 0.05 and 0.2. Plot the
same three curves for each. Report the exploration cost (short-term metric loss) and the benefit
(long-term true satisfaction). Find the ε that maximises long-run true satisfaction, and note how it
compares to what the short-run metric would have chosen.

**Stage 7 — propensity logging and off-policy evaluation.** Make the serving policy stochastic and
log propensities. Then implement IPS and doubly robust estimators, and use logged data from policy A
to estimate the performance of policy B. Compare the estimates against B's true performance (which
your simulator knows). Report the error and the variance. Then repeat with a nearly-deterministic
logging policy and show the estimator failing — that failure is the argument for §2.4's requirement.

**Stage 8 — the fixes.** Implement diversity constraints, a multi-objective serving policy, a
permanent random holdout, and a slowed retraining cadence. Measure each against the Stage 2
baseline on true satisfaction. Report which helped most per unit of complexity and short-term metric
cost.

**Stage 9 — the disparate impact loop.** Add two population groups with a small initial difference
in the *data* (not in the underlying truth). Run the loop and measure the outcome gap over time.
Show it widening. Then show that aggregate accuracy is stable throughout. Then implement the
monitoring that would have caught it — the trend in the per-group gap — and the intervention that
reduces it, with an honest statement of what that intervention costs on the aggregate metric.

## 4. Failure modes

- **Not knowing the loop exists.** The default state.
- **Deterministic serving policy with no propensity logging.** Off-policy evaluation becomes
  impossible, permanently, and cannot be retrofitted.
- **No exploration.** Data collapses onto the current policy; the system cannot improve or measure
  itself.
- **Training only on labelled-because-actioned data.** The blind spot is self-perpetuating.
- **Trusting offline metrics inside a loop.** They improve as the data becomes self-similar.
- **A single proxy objective.** Goodhart's law with an optimiser running continuously.
- **No independent measurement.** The system only measures itself.
- **No coverage or concentration metric.** The single most informative loop signal, untracked.
- **Tight retraining cycles.** High gain, short delay: exactly the conditions for runaway dynamics.
- **Aggregate-only fairness measurement.** Widening group gaps are invisible.
- **"The model just predicts what happens."** False whenever the model influences what happens.

## 5. Exercises

### Warm-up (30 min)

1. Draw the feedback loop and explain the bandit feedback property and why it biases training data.
2. Name six loop types with an example of each.
3. Explain why offline metrics improve inside a loop, and why no drift detector fires.

### Core (3 h)

4. Complete Stages 1–3 and deliver the three-curve chart for the popularity loop and the fraud
   blind spot demonstration with its detection time.
5. Complete Stage 5 and report how early each loop metric detected each loop, against L06's
   detectors.
6. Complete Stage 6 and report the exploration cost/benefit and the optimal ε.
7. For a real system you know, identify at least one feedback loop, state whether anyone monitors
   for it, and design the metric that would reveal it.

### Challenge

8. Complete Stages 4, 7 and 8: the rejection problem with its opportunity cost, off-policy
   evaluation with the near-deterministic failure demonstrated, and the four fixes compared.
9. Complete Stage 9, then write a **feedback loop analysis** for a real deployed system — yours, or
   a well-documented one — covering: every loop you can identify with its structure; which are
   currently monitored and which are not; what data would be needed to detect each; whether
   propensities are logged and what that forecloses; what exploration exists, if any, and what it
   would cost to add; and a prioritised set of recommendations. Then present the one recommendation
   you expect the most resistance to, with the argument you would make — because the hardest part of
   this material is not detecting the loop, it is persuading an organisation to deliberately make
   some decisions worse in order to keep being able to measure itself.

## 6. Self-check

1. Draw the loop and state the bandit feedback property.
2. Name six loop types with their mechanisms.
3. Why does the popularity loop look like success on every dashboard?
4. Give five reasons feedback loops are hard to detect.
5. Give four signals that do indicate a loop, and say which is most informative.
6. What must be true of the logging policy for off-policy evaluation to work?
7. Compare IPS, doubly robust and direct-method estimation.
8. Why is exploration necessary, and how would you justify its cost to a stakeholder?
9. Give five design responses beyond exploration.
10. Explain the predictive policing loop and why "the model just predicts what happens" is false.

## 7. Primary sources

- **Bottou et al., "Counterfactual Reasoning and Learning Systems: The Example of Computational
  Advertising" (JMLR 2013)** — the foundational treatment; long, difficult, and the single most
  valuable paper for this lesson.
- **Sculley et al., "Hidden Technical Debt in Machine Learning Systems" (NeurIPS 2015)**, the
  feedback loop sections — direct and hidden loops named.
- **Lum & Isaac, "To Predict and Serve?" (Significance, 2016)** — the predictive policing loop,
  demonstrated empirically.
- Ensign, Friedler, Neville, Scheidegger & Venkatasubramanian, "Runaway Feedback Loops in Predictive
  Policing" (FAT* 2018) — the formal dynamics.
- Chaney, Stewart & Engelhardt, "How Algorithmic Confounding in Recommendation Systems Increases
  Homogeneity and Decreases Utility" (RecSys 2018) — Stage 2's result, in the literature.
- Swaminathan & Joachims, "Counterfactual Risk Minimization" (JMLR 2015); Dudík, Langford & Li,
  "Doubly Robust Policy Evaluation and Learning" (ICML 2011).
- Joachims, Swaminathan & Schnabel, "Unbiased Learning-to-Rank with Biased Feedback" (WSDM 2017).
- Manheim & Garrabrant, "Categorizing Variants of Goodhart's Law" (2018) — the failure modes of
  optimising a proxy, taxonomised.

---

**Previous:** [L07](L07-evaluation.md) · **Next:**
[L09 — LLM Systems: Retrieval, Evaluation, and Cost](L09-llm-systems.md)
