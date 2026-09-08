# ML-741 · Lesson 06 — Monitoring, Drift, and Silent Degradation

**Estimated study time:** 5 hours
**Prerequisites:** L01, L02, L05; DS-701 L10, CA-731 L07

---

## 1. Orientation

Every previous lesson has pointed here. L01's central operational fact:

> **A model given inputs from a distribution it has never seen does not raise an exception. It
> returns a confident, well-formed, wrong answer, and every component downstream accepts it.**

Ordinary monitoring will not catch this. The service is up, latency is normal, the error rate is
zero, and the model has been producing garbage for six weeks. This lesson is about building the
monitoring that does catch it, and about being precise concerning what each signal can and cannot
tell you.

The organising distinction, which is the lesson's core content:

> There are **three different things that can go wrong**, they have **different detection
> mechanisms**, and they warrant **different responses**. Confusing them produces monitoring that
> alerts constantly and detects nothing.

1. **Input drift**: the incoming data distribution has changed. Detectable immediately. Does not by
   itself mean anything is wrong.
2. **Prediction drift**: the model's output distribution has changed. Detectable immediately. A
   consequence of input drift, and equally not proof of a problem.
3. **Performance degradation**: the model has become less accurate. **This is the thing you
   actually care about, and you cannot measure it until labels arrive** (L02 §2.4).

The whole difficulty of ML monitoring is contained in that asymmetry: **the signal you can measure
now is not the one you care about, and the one you care about arrives late or never.**

## 2. Theory

### 2.1 The layers of monitoring

Bottom to top, in increasing usefulness and increasing delay:

1. **Infrastructure**: CPU, memory, GPU, disk. Necessary, tells you nothing about model quality.
2. **Service**: latency, throughput, error rate, saturation (CA-731 L07's golden signals). Still
   nothing about quality.
3. **Data quality**: schema violations, null rates, ranges, volume (L02 §2.5). **This layer catches
   most real production ML failures**, and it is available immediately.
4. **Input drift**: distributional change in features.
5. **Prediction drift**: distributional change in outputs.
6. **Performance**: accuracy, precision, recall, calibration — requires labels.
7. **Business**: the outcome the model exists to influence. The ultimate measure, the most delayed,
   and the most confounded.

The practical guidance: **layers 1–3 catch most failures and are cheap; layer 6 is what you care
about and is delayed; layers 4–5 are the bridge and are the most commonly misused.** A system with
excellent drift detection and no data quality checks has built the sophisticated thing and skipped
the effective one.

### 2.2 The kinds of drift, named properly

Using the standard decomposition of P(X, y) = P(y | X) · P(X):

- **Covariate shift**: P(X) changes, P(y | X) does not. The input distribution moved but the
  relationship is intact. A model may still be fine — if the new inputs are within the region it
  learned well. Example: your product launches in a new city, so the geography feature's
  distribution changes, but the relationship between features and outcome is unchanged.
- **Label shift / prior probability shift**: P(y) changes. Fraud rates rise seasonally. Often
  correctable by recalibration rather than retraining.
- **Concept drift**: P(y | X) changes — **the relationship itself has changed**. This is the
  serious one: the same inputs now imply a different outcome. Fraudsters change tactics; consumer
  behaviour shifts after an event. **No amount of input monitoring detects concept drift**, because
  the inputs may look identical. Only labels reveal it.

By temporal shape: **sudden** (a deployment upstream, a policy change, a pandemic), **gradual**
(behaviour evolving), **incremental** (slow steady movement), and **recurring** (seasonality —
which is not drift and must not be alerted on, and which is why a reference window shorter than
your seasonal period produces constant false alarms).

The point to hold on to: **the dangerous kind is the one you cannot see in the inputs.** Which
means input drift monitoring is a useful early warning and never a sufficient one.

### 2.3 Detecting distributional change

For **numeric** features:

- **Kolmogorov–Smirnov**: maximum difference between empirical CDFs. Non-parametric and standard.
  At production sample sizes it will find statistically significant differences that are practically
  meaningless — significance is not magnitude.
- **Population Stability Index**: a binned, symmetric divergence. Conventional thresholds of 0.1
  (moderate) and 0.25 (significant) are folklore and should be calibrated on your own data.
- **Wasserstein distance**: sensitive to how far the distribution moved, not only that it moved.
  Usually more interpretable than KS for continuous features.
- **Simple summary statistics with control limits**: mean, standard deviation, quantiles tracked
  over time. Less sophisticated, more interpretable, and often sufficient.

For **categorical** features: chi-squared, new-category detection (a genuinely important and simple
check), and cardinality changes.

For **multivariate** drift, which is what actually matters because features move together:

- **Domain classifier**: train a classifier to distinguish reference from current data. If it can,
  they differ; its AUC quantifies how much, and its feature importances tell you *which* features
  moved. This is the most practically useful multivariate technique and it is easy to implement.
- **Reconstruction error** from an autoencoder or PCA fitted on reference data.
- **Drift in the model's own embedding or output space**, which is closer to what the model
  actually perceives.

The engineering problems that determine whether any of this works:

- **Choosing the reference window.** Too short and it captures noise; too long and it lags. It must
  span at least one full seasonal cycle or seasonality is reported as drift, permanently.
- **The multiple testing problem.** Testing 200 features daily at p < 0.05 produces ten false alarms
  a day. Correct for it, or use magnitude thresholds instead of significance.
- **Statistical significance is not practical significance.** With a million samples, everything is
  significantly different. **Alert on effect size, not on p-values.**
- **Alert fatigue is the failure mode**, and it is near-universal in drift monitoring. A drift alert
  that fires weekly and never requires action will be ignored within a month, at which point the
  system provides negative value because it creates a false sense of coverage.

### 2.4 Performance monitoring when labels are delayed

The core problem. Approaches:

- **Wait for labels.** Correct, and delayed by exactly the label delay. Do it anyway, as the ground
  truth against which everything else is validated.
- **Proxy metrics.** Something correlated and faster — click-through as a proxy for satisfaction, a
  downstream conversion. Available sooner and biased in ways you must understand (L02 §2.4).
- **Human labelling of a sample.** Label a small random sample quickly. Costs money, gives an
  unbiased estimate with a computable confidence interval, and is under-used. Note it must be a
  *random* sample — labelling the cases that looked wrong tells you nothing about the rate.
- **Confidence and calibration monitoring.** A well-calibrated model's confidence distribution
  shifting is a signal — with the substantial caveat that modern neural networks are frequently
  poorly calibrated, so this must be validated before it is trusted.
- **Performance estimation without labels** (importance weighting under a covariate-shift
  assumption, or methods such as confidence-based accuracy estimation). These work **only when the
  assumption holds** — specifically, only under covariate shift, not under concept drift, which is
  precisely the case you most need to detect. Useful, and dangerous if the assumption is not
  checked.

The honest summary: **there is no way to detect concept drift without labels.** Everything else is
early warning. Design your system so labels arrive as fast as they can, and treat the label delay
as a first-class operational property with an owner (L02 Stage 6).

### 2.5 Slices, and why aggregates lie

Aggregate performance can be stable while the model fails badly for a subpopulation. This is the
single most common way a model is quietly harmful, and it is invisible in every aggregate metric.

So: **monitor by slice.** Define slices along dimensions that matter — customer segment, geography,
device, language, tenure, and any protected attribute your context and regulations require. For
each, track the metric and alert on divergence from the aggregate.

The engineering issues are real: slice count explodes combinatorially, small slices have noisy
metrics, and alerting per slice produces noise. Practical approach: define a bounded set of slices
that matter, require a minimum sample size before alerting, and use an automated slice-discovery
technique (SliceFinder-style search for underperforming subpopulations) to find the slices you did
not think to define — which is where the surprises are.

### 2.6 What to do when drift is detected

The response depends on which of §2.2's kinds you have, which is why the distinction matters:

- **Input drift, performance stable**: note it, do not act. This is the common case and it is why
  alerting on input drift alone produces fatigue.
- **Input drift, performance degraded**: retrain on recent data. Covariate shift with a model that
  did not generalise.
- **No input drift, performance degraded**: concept drift. Retraining on recent data is the
  response, and the more important question is *what changed in the world* — because concept drift
  usually has a cause worth understanding, and sometimes the cause is your own system (L08).
- **New categories or schema changes**: a data pipeline problem, not a model problem. Fix upstream.
- **A sudden step change**: almost always an upstream deployment, not a change in the world. Check
  what shipped before you retrain — this check has saved a great many unnecessary retraining
  cycles.

And the response must be gated: **automatic retraining requires an automatic evaluation gate**
(L04 §2.6). Drift-triggered retraining that deploys without a gate can turn a data pipeline bug into
a corrupted production model in one automated cycle.

### 2.7 Logging for debuggability

You cannot debug what you did not record. For each prediction, log: the request ID and timestamp,
the **feature values actually used** (this is what makes skew detection and later retraining
possible), the model version, the prediction and its confidence, the latency, and — when it arrives
— the label, joined back on the request ID.

That last join is the mechanism that makes everything in §2.4 possible, and building it late is
much harder than building it first.

The constraints: sample if volume makes full logging impractical (but sample *randomly*, and
stratify to over-sample rare classes and slices you care about); apply retention policies; and
handle personal data properly, because a prediction log is a rich record of individuals and is
subject to the same rules as any other personal data (DI-721 L09's crypto-shredding is relevant).

## 3. Construction: build the monitoring that catches silent failure

Build in `mpse/ml741/l06/`, on the L05 serving system.

**Stage 1 — prediction logging.** Log every prediction with its features, version, output and
latency, and implement the label join when labels arrive. Verify the join rate — if 20% of
predictions never get a label joined, find out why, because that is your monitoring's blind spot.

**Stage 2 — the layered dashboard.** Implement all seven layers of §2.1 for your system. Then
answer, for each layer: what failure does this catch that the layer below does not?

**Stage 3 — the drift laboratory.** Build an injector that produces each drift type on demand:
covariate shift (shift one feature's distribution), label shift (change the class balance), concept
drift (change the relationship between a feature and the label), plus sudden, gradual and seasonal
temporal shapes. This harness is what makes the rest of the lesson measurable.

**Stage 4 — detection, compared.** Implement KS, PSI, Wasserstein and a domain classifier. Run each
against every injected drift type and produce the detection matrix: which method detects which
drift, at what magnitude, with what delay. **The key result to confirm and internalise: no input
drift method detects concept drift.** Verify it yourself.

**Stage 5 — false positives.** Run all four detectors against *stable* data with normal seasonal
variation for a simulated year. Count the false alarms. Then tune thresholds and reference windows
to get the false alarm rate to something a team would tolerate — perhaps one per month — and report
what detection sensitivity you lost. This trade-off is the actual design of a drift monitor, and
skipping it is why most drift monitoring is ignored.

**Stage 6 — performance monitoring under delay.** With your L02 label delay, implement: the
delayed ground-truth metric, a proxy metric, and a random-sample human-labelling simulation with
confidence intervals. Compare all three against the eventual truth. Report how much earlier each
would have caught a real degradation, and at what cost.

**Stage 7 — slices.** Define eight slices. Then construct the failure: an aggregate metric that is
stable while one slice degrades badly. Show your aggregate monitoring missing it and your slice
monitoring catching it. Then implement automated slice discovery and see whether it finds a
degraded slice you had not defined.

**Stage 8 — the alerting policy.** Design alerts that fire on the right thing: not on input drift
alone; on performance degradation; on data quality failures; on slice divergence; and on the
combination of input drift *plus* a degradation signal. Then replay six months of simulated
production including three real incidents and three benign drifts, and report the alerts: true
positives, false positives, and the detection delay for each incident. Compare against a naive
"alert on any KS test p < 0.05" policy.

**Stage 9 — the response runbook.** Write it: for each alert type, the diagnostic steps, the
decision tree from §2.6, and the action. Then run a simulated incident against it with someone who
has not seen your system, and revise based on where they got stuck.

## 4. Failure modes

- **Ordinary service monitoring only.** The model degrades silently for weeks.
- **Alerting on input drift alone.** Constant alerts, no action, ignored within a month.
- **Alerting on p-values at production sample sizes.** Everything is significant.
- **A reference window shorter than the seasonal period.** Seasonality reported as drift forever.
- **Believing input drift monitoring covers concept drift.** It cannot, by construction.
- **Aggregate metrics only.** The subpopulation failure is invisible.
- **Label join not built.** No ground truth, so nothing can be validated.
- **Label join rate unmeasured.** A silent gap in coverage.
- **Label-delay-blind expectations.** Detection cannot be faster than labels, whatever the dashboard
  suggests.
- **Sampling non-randomly for human labelling.** Biased estimate presented as ground truth.
- **Performance estimation methods used under concept drift.** Their assumption is exactly what has
  failed.
- **Drift-triggered retraining with no gate.** Automates the deployment of bad models.
- **Retraining before checking what shipped upstream.** A step change is usually a deployment.

## 5. Exercises

### Warm-up (30 min)

1. Give the three things that can go wrong, their detection mechanisms, and why the asymmetry
   between them is the whole difficulty.
2. Distinguish covariate shift, label shift and concept drift, with an example of each, and say
   which is undetectable from inputs.
3. Explain why statistical significance is the wrong basis for a drift alert at production sample
   sizes.

### Core (3.5 h)

4. Complete Stages 1–3, including the label join rate and the drift injector.
5. Complete Stage 4 and deliver the detection matrix, confirming that no input method detects
   concept drift.
6. Complete Stage 5 and report the false alarm rate before and after tuning, with the sensitivity
   lost.
7. Complete Stage 7 and deliver the aggregate-stable/slice-degraded demonstration.

### Challenge

8. Complete Stages 6, 8 and 9: the three performance-monitoring approaches compared against
   eventual truth, the six-month alerting replay with detection delays, and the runbook tested on
   someone else.
9. Build a **degradation early-warning system** that combines signals rather than alerting on each:
   input drift magnitude, prediction drift, confidence distribution shift, data quality violations,
   and slice divergence, weighted into a single score with a calibrated threshold. Validate it on
   your drift laboratory: measure detection delay and false alarm rate against the best single
   signal. Then — the part that matters — **test it against concept drift with no input drift**, and
   report honestly what it can and cannot do. A combined score that claims to detect concept drift
   from inputs alone is claiming something impossible; the useful version says clearly which
   failures it covers and which require labels, and a monitoring system that is honest about its
   blind spot is worth more than one that is not.

## 6. Self-check

1. State the central operational fact about model failure and why ordinary monitoring misses it.
2. Give the three things that can go wrong and the detection mechanism for each.
3. Give the seven monitoring layers and say which catches most real failures.
4. Define covariate shift, label shift and concept drift formally.
5. Why is concept drift undetectable from inputs?
6. Name four numeric drift tests and the multivariate technique that also tells you which features
   moved.
7. Give three engineering problems that make drift detection produce false alarms.
8. Give four approaches to performance monitoring under label delay, with the limitation of each.
9. Why do aggregate metrics lie, and what is the practical approach to slices?
10. Give the decision tree for responding to a drift alert.

## 7. Primary sources

- **Gama, Žliobaitė, Bifet, Pechenizkiy & Bouchachia, "A Survey on Concept Drift Adaptation"
  (ACM Computing Surveys, 2014)** — the definitive taxonomy; §2.2 is drawn from it.
- **Breck et al., "The ML Test Score" (IEEE Big Data 2017)**, monitoring section.
- Polyzotis et al., "Data Validation for Machine Learning" (MLSys 2019) — with L02.
- Rabanser, Günnemann & Lipton, "Failing Loudly: An Empirical Study of Methods for Detecting Dataset
  Shift" (NeurIPS 2019) — what actually works, empirically; read it before choosing a method.
- Chung, Krishnan & Kraska, "Slice Finder: Automated Data Slicing for Model Validation" (ICDE 2019).
- Guo, Pleiss, Sun & Weinberger, "On Calibration of Modern Neural Networks" (ICML 2017) — before
  trusting confidence as a signal.
- Garg et al., "Leveraging Unlabeled Data to Predict Out-of-Distribution Performance" (ICLR 2022) —
  and its assumptions.
- Huyen, *Designing Machine Learning Systems*, chapter 8.
- DS-701 L10 and CA-731 L07 for the alerting discipline this lesson assumes.

---

**Previous:** [L05](L05-serving-and-inference.md) · **Next:**
[L07 — Evaluation: Offline, Online, and the Gap](L07-evaluation.md)
