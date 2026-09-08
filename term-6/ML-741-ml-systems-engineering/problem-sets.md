# ML-741 — Problem Sets

The three problem sets build one ML system on the CA-731 platform, and then subject it to the review
of L10. Do them in order.

**A note on what is being assessed.** Not model quality. A mediocre model with excellent
engineering around it scores well here; an excellent model in a notebook scores badly. The
deliverables that matter are the measurements, the tests, the monitoring and the honest accounts of
what did not work.

**A note on simulation.** Several parts — feedback loops especially — use simulators rather than
real systems, because the phenomena take months to manifest in production and you need ground truth
to measure against. This is not a compromise; it is the only way to see these effects at all, and
the discipline of building a simulator whose truth you know is itself worth learning.

**A note on the model.** Use a simple model (gradient-boosted trees on tabular data is ideal) for
Problem Sets 1–2. Nothing in this course requires a large model, and a fast-training model lets you
run the ten-seed experiments, the drift simulations and the loop simulations that the assessment
actually depends on.

---

## Problem Set 1 — A Reproducible Pipeline With Data Contracts
**Covers L01–L04 · Budget: 22–26 hours**

**Part A — The naive system and its audit (L01).** Stages 1–3: the notebook-and-Flask system built
deliberately badly; the full system diagram with the ML-code fraction computed and per-box ownership
and test coverage marked; and the technical debt audit against Sculley's catalogue with specific
evidence per pattern.

**Part B — Skew and leakage (L01).** Stage 4: all four training/serving skew mechanisms introduced
and measured, with the label-leakage case shown inflating the offline metric and then failing in
production, plus a written account of how it would have been detected.

**Part C — Tests (L01).** Stages 5–7: the ML Test Score before and after implementing the five
cheapest missing tests; data tests with five injected corruptions plus one your checks missed and a
new check for it; and behavioural tests including one that fails with your reasoned verdict on
whether the model or your expectation was wrong.

**Part D — Should this be ML? (L01).** Stage 8: the rule-based baseline measured against your model,
with the total cost of ownership of each and an honest conclusion.

**Part E — Point-in-time correctness (L02).** Stages 1–2: the leakage gap quantified between the
naive and correct training sets, and the as-of join implemented twice with the fast version verified
against the naive oracle.

**Part F — Features as software (L02).** Stages 3–4: the feature module with types, tests,
documentation and a rendered dependency graph; the shared training/serving computation with a CI
skew test; and the deliberate break caught by that test.

**Part G — Validation and labels (L02).** Stages 5–6: five validation categories wired to block
training, with six injected faults including a subtly wrong unit conversion and a check that catches
it; and label generation with realistic delay, with the computed retraining cadence and its
downstream consequences written up.

**Part H — Reproducibility (L03).** Stages 1–4: the ten-run noise floor; seeding and the residual
variance explained; bit-exactness achieved with the deterministic-mode cost measured and a judgement
about paying it; and automatic provenance recording including the dirty-working-tree check.

**Part I — Registry and experiments (L03).** Stages 5–7: the model registry with the
timestamp-to-version reverse lookup tested against a simulated incident; a month-old model
reproduced from its record alone with everything that went wrong listed; and eight experiments with
stated hypotheses against the noise floor, with conclusions including negatives and a five-bullet
summary.

**Part J — Training (L04).** Stages 1–5: the profile with the bottleneck classified; the input
pipeline optimisation sequence with utilisation after each change; the memory/throughput curves;
mixed precision with its quality effect against the noise floor; and checkpointing with resumption
verified equivalent — including the thing you forgot to checkpoint.

**Part K — The retraining pipeline (L04).** Stage 8: automated retraining with an evaluation gate,
demonstrated blocking promotion of a model trained on deliberately degraded data.

**Design note (1,500–2,000 words).** The leakage gap you measured and what produced it. What the
noise floor revealed about improvements you had previously believed in. What reproducing a month-old
model exposed about your provenance. And the honest answer to Part D: should this system be machine
learning at all?

---

## Problem Set 2 — Serving, Monitoring, and an Honest Evaluation
**Covers L05–L07 · Budget: 24–28 hours**

**Part A — Latency (L05).** Stages 1–2: the seven-component latency decomposition at p50/p95/p99
with the dominant term identified and stated prominently; the single-request baseline with
utilisation and cost per thousand predictions.

**Part B — Batching (L05).** Stages 3–4: the throughput/latency surface across batch size and wait
time with the SLO operating point marked; and adaptive plus length-bucketed batching measured under
a bursty load profile with the padding waste eliminated reported.

**Part C — Optimisation (L05).** Stages 5–6: compilation speedup with numerical verification and any
prediction class changes reported; and FP16 and INT8 quantisation with latency, throughput, memory
and accuracy **on at least four slices**, plus a written decision about the worst slice.

**Part D — Caching and failure (L05).** Stages 8, 10: the prediction cache with hit rate, cost
saving, and the stale-cache bug demonstrated; and every failure case from §2.7 implemented, tested,
with its accuracy cost measured and added to your chaos runs.

**Part E — Logging and layers (L06).** Stages 1–2: prediction logging with the label join and its
measured join rate; and the seven monitoring layers implemented with a statement of what each
catches that the layer below does not.

**Part F — Drift detection (L06).** Stages 3–5: the drift injector covering all three kinds and
three temporal shapes; the four detectors compared in a detection matrix confirming that none
detects concept drift; and the year-long false-alarm measurement with thresholds tuned to a
tolerable rate and the sensitivity lost reported.

**Part G — Performance under delay (L06).** Stage 6: delayed ground truth, a proxy metric, and
random-sample human labelling with confidence intervals, all compared against eventual truth with
detection-time and cost reported for each.

**Part H — Slices and alerting (L06).** Stages 7–9: the aggregate-stable/slice-degraded
demonstration with automated slice discovery; the alerting policy replayed over six simulated months
with three incidents and three benign drifts, reporting true positives, false positives and
detection delays against a naive policy; and the response runbook tested on someone unfamiliar with
the system.

**Part I — Offline evaluation (L07).** Stages 1–4: the metric audit with the experiment re-ranking
under a cost-based metric; four baselines including the trivial one with bootstrap intervals and
your honest judgement; calibration measured, corrected, and a decision about whether it matters; and
the slice analysis with your last improvement's effect on the worst slice.

**Part J — Online evaluation (L07).** Stages 5–8: shadow deployment with the comparison report and
something it revealed that offline evaluation had not; a fully specified experiment with a
pre-written analysis plan, executed and analysed; all four statistical hazards demonstrated with
wrong and right conclusions; and the twenty-run A/A test with its result.

**Design note (1,500–2,000 words).** The gap between what you were optimising and what the system
exists to do. What the shadow deployment found. What the A/A test told you about your
experimentation infrastructure. And the honest answer to: given the label delay and the monitoring
you built, how long would this system be wrong before you knew?

---

## Problem Set 3 — Feedback Loops, an LLM System, and a Review
**Covers L08–L10 · Budget: 22–26 hours**

**Part A — The loop simulator (L08).** Stages 1–3: the simulated world with known ground truth; the
popularity loop's three-curve chart (rising CTR, falling coverage, falling true satisfaction); and
the fraud blind spot with its measured detection time under L06's monitoring — which will be never.

**Part B — Rejection and detection (L08).** Stages 4–5: the credit rejection loop with the shrinking
approved region and its opportunity cost against ground truth; and the four loop metrics compared
against L06's drift detectors on detection time.

**Part C — Exploration and off-policy evaluation (L08).** Stages 6–7: ε-greedy at three levels with
cost and benefit, and the ε maximising long-run true satisfaction compared against what the
short-run metric would have chosen; and IPS plus doubly robust estimation validated against
simulator truth, with the near-deterministic logging policy failure demonstrated.

**Part D — Fixes and fairness (L08).** Stages 8–9: four mitigations compared on true satisfaction
per unit of complexity; and the disparate impact loop with the widening gap, stable aggregate
accuracy, the monitoring that catches it, and the intervention with its honest aggregate cost.

**Part E — The golden set first (L09).** Stages 1–2: fifty real questions with grading criteria
including hard, ambiguous and out-of-scope cases, built *before* any pipeline code; and the naive
baseline with cost and latency per query.

**Part F — Retrieval (L09).** Stage 3: retrieval evaluated separately with recall@5 and recall@20;
three chunking variants, hybrid retrieval and reranking each measured; and the retrieval-improvement
to end-to-end-improvement conversion rate reported.

**Part G — Validation and judging (L09).** Stages 4–5: structured output with grounding validation
and the fabrication rate before and after; bounded retry with its cost; and the LLM judge validated
against fifty hand-graded outputs, with the rubric fixed and agreement re-measured, plus two biases
demonstrated and mitigated.

**Part H — Cost, injection, containment (L09).** Stages 6–9: the cost/quality Pareto frontier with a
justified operating point; the prompt injection demonstration with mitigations and honest residual
risk; all five failure containment cases including retrieval-finds-nothing with your baseline's
fabrication rate on it; and the regression suite with the fix-one-break-three experiment.

**Part I — The agent comparison (L09).** Stage 10: the agent and the constrained workflow compared
over fifty runs on success rate, cost, latency, variance and debuggability, with your shipping
decision defended.

**Part J — The review (L10).** Stages 1–5: the full design document with all fifteen sections; the
ML Test Score with the delta from Problem Set 1 explained; the nine-dimension self-review with your
honest paragraph answering "how would you know if the model became wrong?" including the detection
delay; the top five ranked findings; and both pre-mortems, with emphasis on the silent one.

**Part K — Reviewing and being reviewed (L10).** Stages 6–9: a written review of someone else's ML
system with your misunderstanding ratio; your document reviewed with a recorded response to every
concern; the team questions from §2.5 answered honestly with what would have to change; and the
one-page organisational checklist revised after real use.

**Design note (2,000–2,500 words).** Trace one prediction from the data that produced the model,
through training, serving, monitoring and back into the data that trains the next model — naming at
each hop what guarantee holds and what could silently corrupt it. Then: the feedback loop in your
system that you would find hardest to persuade an organisation to pay to mitigate, and the argument
you would make. And the question the course exists to make answerable: **what breaks when this model
becomes a dependency, and how long before anyone notices?**

---

## Course position paper (1,500 words)

Choose one:

1. **"The model is a few percent of an ML system, and the industry's attention is allocated in
   roughly inverse proportion to where the failures are."** Defend or refute using your own audit
   and at least two primary sources.
2. **"Most feature stores are adopted for reuse and deliver only consistency — and consistency
   could have been had for a hundredth of the cost."** Argue it from your L02 Stage 9 assessment.
3. **"Offline evaluation is a filter, not a decision procedure, and treating it as the latter is the
   most common cause of ML projects that ship and do nothing."** Use your L07 measurements.
4. **"Any deployed model that lacks exploration and propensity logging has permanently forfeited the
   ability to know how well it is doing."** Defend or refute, addressing the cases where exploration
   is genuinely unacceptable.
5. **"In LLM systems, evaluation is the entire engineering problem and prompt engineering is a
   distraction."** Argue from your L09 experience, and address the strongest counter-argument.
6. **"An ML system with no answer to 'how would you know if the model became wrong?' should not be
   in production, regardless of how good it is."** Defend or refute, being specific about what
   answers are adequate.

The structure is the one from `00-program/assessment-and-rubrics.md`: claim, grounds, the strongest
rebuttal you can construct, and the limits of your position. A paper that does not name a condition
under which its claim fails has not made a claim.

---

## Submission checklist (per set)

- [ ] Code in `courses/ml741/psN/`, runnable from a clean clone via the README.
- [ ] Tests passing, with a stated coverage figure *and* a sentence on what coverage does not tell
      you here.
- [ ] `mypy --strict` and `ruff` clean, or every exception documented with a reason.
- [ ] All measurements reproducible: every metric is reported against the noise floor from L03, and slice metrics accompany every aggregate.
- [ ] Charts and tables as files, not as descriptions — a plot referred to but not produced does not
      count.
- [ ] `NOTE.md` — the design note.
- [ ] `MARK.md` — self-marked against the five-criterion rubric with one line of justification per
      criterion.
- [ ] `log/failures.md` updated with everything you got wrong on the way, including the predictions
      that were incorrect.
