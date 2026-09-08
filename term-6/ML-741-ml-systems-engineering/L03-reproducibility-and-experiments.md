# ML-741 · Lesson 03 — Reproducibility, Experiments, and Provenance

**Estimated study time:** 4.5 hours
**Prerequisites:** L01, L02; SE-511 L09, DI-721 L09

---

## 1. Orientation

Three questions that an ML team should be able to answer immediately and usually cannot:

1. **Which model is in production, and what exactly produced it?** Which commit, which data,
   which hyperparameters, which environment.
2. **Can we produce it again?**
3. **We ran forty experiments last quarter — what did we learn?**

The first is provenance, the second is reproducibility, and the third is the one that reveals
whether the team's experimental process has any value. All three are engineering problems with
known solutions that are cheap to adopt early and expensive to retrofit.

The framing that makes reproducibility tractable, because "reproducible" is used to mean several
different things:

> Reproducibility is a **spectrum of increasingly strong claims**, each with a different cost.
> Decide which one you need — most teams need less than bit-exactness and more than they have.

## 2. Theory

### 2.1 The reproducibility spectrum

From weakest to strongest:

1. **Provenance only.** You can say what produced this artifact — commit, data version,
   configuration, environment — but not necessarily re-run it. Cheap, and it is the floor: without
   it, nothing else is possible and no incident can be investigated.
2. **Re-runnable.** The pipeline can be executed again with the same inputs. Requires the code,
   data and environment to still exist and still work.
3. **Statistically reproducible.** Re-running produces a model with equivalent performance within
   noise. This is what most teams actually need: you can rebuild a model that behaves the same, and
   you can attribute a metric change to a real cause rather than to randomness.
4. **Bit-exact.** Re-running produces a byte-identical artifact. Expensive, and genuinely required
   only in regulated contexts, for debugging a specific discrepancy, or where an audit demands it.

The cost curve is steep between 3 and 4, and the value curve is not. **Aim for 1 and 2 always, 3
routinely, and 4 only where something specific requires it.**

### 2.2 What makes bit-exactness hard

Worth knowing even if you do not pursue it, because these are also the sources of confusing
variance at level 3:

- **Random seeds** in initialisation, shuffling, augmentation, dropout, and any sampling. Every
  library has its own generator; seeding one does not seed the others.
- **Parallel non-determinism.** Floating-point addition is not associative, so summing in a
  different order gives a different result. Multi-threaded and GPU reductions have non-deterministic
  order by default, so results differ run to run on identical inputs.
- **GPU kernels.** Some cuDNN algorithms are non-deterministic; some are selected dynamically based
  on benchmarking at runtime.
- **Library versions.** A change in a numerical library changes results in the last bits, which can
  change a tie-break, which can change a split, which can change everything.
- **Hardware.** Different GPU architectures produce different floating-point results.
- **Data ordering.** A pipeline that reads files in filesystem order produces a different shuffle
  on a different machine.

The techniques: seed everything (and have a single function that does), enable deterministic
algorithm modes where the framework offers them (accepting a performance cost), pin every dependency
version exactly, containerise, fix data ordering explicitly, and — the one people skip — **measure
your run-to-run variance** so you know what "the same" means for your setup. A metric difference of
0.3% is meaningless if your run-to-run standard deviation is 0.5%, and teams routinely celebrate
improvements smaller than their own noise.

### 2.3 Provenance: what must be recorded

For every model artifact, record enough that a stranger could reconstruct the situation:

- **Code**: the commit hash of every repository involved, and whether the working tree was clean.
  An uncommitted change is the most common provenance gap.
- **Data**: the exact version — partition range, snapshot ID, content hash, or log offset (L02
  §2.6). Including the *feature* definitions' version.
- **Configuration**: every hyperparameter, and the full resolved configuration rather than the
  defaults-plus-overrides that produced it. Resolve and record.
- **Environment**: the container image digest (not the tag — tags move), the framework versions, the
  hardware.
- **Randomness**: the seeds.
- **Outputs**: the metrics, on which evaluation set, at which version.
- **Lineage**: which experiment this derived from, and who ran it.

The test: **given only the artifact, can you answer "why does this model behave this way?"** If you
have to ask a person, you have a provenance gap, and that person will eventually leave.

The practical form is a **model registry** with immutable versions, a stage (staging, production,
archived), and this metadata attached — plus, importantly, the ability to answer the reverse
question: given a production incident at time *T*, which model version was serving, and what
produced it?

### 2.4 Experiment tracking, and doing it usefully

Tracking tools (MLflow, Weights & Biases, and others) log parameters, metrics and artifacts per run.
Adopting one is easy; using it well is not, and the difference is entirely in the discipline around
it.

The failures, all common:

- **Logging everything and learning nothing.** Four hundred runs, no structure, no conclusions.
  Nobody can say what was learned.
- **No hypothesis.** A run without a stated expectation cannot confirm or refute anything; it
  generates a number.
- **Comparing runs that differ in several ways at once.** Nothing is attributable.
- **Ignoring variance.** Two runs differing by less than the seed-to-seed variance are the same run.
- **Not recording failures.** The experiments that did not work are the most valuable record you
  have, and they are exactly the ones nobody writes up.

The discipline that makes tracking worth its cost:

1. **State the hypothesis before the run**, in the run's own metadata. "Adding the merchant-risk
   feature will improve recall at fixed precision by at least 2 points."
2. **Change one thing.** If you must change several, you are exploring, not testing — label it as
   such.
3. **Run the baseline with multiple seeds first** and record the variance. Every subsequent
   comparison is against that noise floor.
4. **Record the conclusion**, including "no effect" and "worse". A quarterly review of conclusions
   is what turns runs into knowledge.
5. **Keep a decision log**: not the runs, but what you decided and why. This is the artifact that a
   new team member reads, and the runs are its evidence.

### 2.5 Hyperparameter search, honestly

- **Grid search** wastes effort on unimportant dimensions; Bergstra and Bengio showed **random
  search** dominates it for the same budget, because most hyperparameters do not matter and random
  search covers the ones that do more densely.
- **Bayesian optimisation** models the response surface and is more sample-efficient; worth it when
  each trial is expensive.
- **Hyperband / ASHA** allocate budget by early-stopping poor trials, which is usually the biggest
  practical win because it converts a fixed budget into far more trials.

The engineering points, which matter more than the algorithm choice:

- **Search on a validation set, evaluate on a test set you touch once.** Search *is* fitting; a
  test set you have optimised against is a validation set.
- **Report the search budget.** "We got 0.92" means something different after 5 trials than after
  5,000, and comparisons across different budgets are not comparisons.
- **The search is part of the pipeline** and must be reproducible too, including the search's own
  randomness.
- **Beware the multiple comparisons problem.** Run enough trials and one will look good by chance.
  The fix is a held-out set evaluated once, and scepticism proportional to the number of trials —
  this is the same statistical hazard as p-hacking, and the ML community is not immune to it.

### 2.6 Model artifacts and formats

The artifact itself needs engineering attention:

- **Pickle is not a deployment format.** It executes arbitrary code on load (a genuine security
  issue if the artifact crosses a trust boundary), it is fragile across library versions, and it
  ties the artifact to the training environment. It is fine as a within-run intermediate and wrong
  as a production artifact.
- **ONNX** provides a portable graph representation, decoupling training framework from serving
  runtime. Conversion is not always faithful — verify numerically after converting, on real inputs,
  and treat a conversion as a change requiring evaluation.
- **Framework-native saved formats** (SavedModel, TorchScript) are usually the pragmatic choice
  when serving with the same framework.
- **The artifact is more than the weights.** The preprocessing, the feature definitions, the
  post-processing and the thresholds are all part of the deployed behaviour. **Package them
  together, version them together, and deploy them together** — a mismatch between a model version
  and its preprocessing version is a nasty, quiet production bug.
- **Model size and load time** are operational properties: they determine cold start, scaling speed
  and memory footprint (L05).

### 2.7 The environment

Reproducibility fails at the environment more often than anywhere else:

- **Pin exactly, including transitive dependencies.** A lockfile, not a requirements file with
  ranges (SE-511 L09).
- **Containerise, and reference images by digest.** A tag is a mutable pointer.
- **Record hardware** where results depend on it.
- **Keep old environments buildable** for as long as you might need to reproduce a model — which
  means an image registry retention policy that matches your model retention policy, and those two
  policies are usually set by different people who have never spoken.

## 3. Construction: make it reproducible

Build in `mpse/ml741/l03/`, on the L02 pipeline.

**Stage 1 — measure the variance.** Train your model ten times, changing nothing but the seed.
Report the mean and standard deviation of your primary metric. **This number is your noise floor**,
and every later comparison is against it. Most teams have never computed it, and many of their past
"improvements" were inside it.

**Stage 2 — seed everything.** Write one `set_all_seeds(n)` function covering Python's `random`,
NumPy, the framework, and any data loader workers. Re-run the ten-run experiment and report the new
variance. Then find the remaining source of non-determinism if the variance is not zero — it will
usually be parallel reduction order.

**Stage 3 — pursue bit-exactness, and cost it.** Enable deterministic algorithms, fix data ordering,
pin versions, and containerise. Achieve byte-identical artifacts across two runs. Then measure the
training slowdown deterministic mode cost you. Write 300 words on whether you would pay it in
production, and under what circumstances you would.

**Stage 4 — provenance.** Implement recording of everything in §2.3, automatically, as part of
training. Include the check that refuses to train from a dirty working tree (or records the diff).
Then take an artifact from three weeks ago and answer, from the record alone: what code, what data,
what config, what environment, what metrics.

**Stage 5 — the registry.** Build (or adopt) a model registry with immutable versions, stages, and
the provenance attached. Then implement the reverse lookup: given a timestamp, which model version
was serving? Test it against a simulated incident.

**Stage 6 — reproduce from the record.** Take a model trained a month ago (simulate the passage of
time by changing dependency versions and data in between) and reproduce it from its provenance
record alone. Record everything that went wrong. This exercise reliably finds the gaps in Stage 4,
which is why it is separate from it.

**Stage 7 — experiment discipline.** Run a series of eight experiments with stated hypotheses,
one variable each, against the Stage 1 noise floor. Record conclusions including the negative ones.
Then write the quarterly summary: what was learned, in five bullet points. If the eight runs support
fewer than three conclusions, the experimental design was weak, and diagnosing why is part of the
exercise.

**Stage 8 — hyperparameter search, done properly.** Run random search and ASHA on the same budget
and compare. Report the search budget with the result. Then demonstrate the multiple comparisons
hazard: show that the best-of-N validation score is optimistically biased relative to the held-out
test score, and quantify the gap as a function of N.

**Stage 9 — the artifact.** Package model, preprocessing, feature definitions and thresholds as one
versioned artifact. Convert to ONNX and verify numerical equivalence on a real input sample —
report the maximum discrepancy and decide whether it matters. Then construct the version-mismatch
bug: serve model v2 with preprocessing v1, and measure the damage.

## 4. Failure modes

- **No noise floor.** Improvements smaller than the seed variance are celebrated as real.
- **Uncommitted changes at training time.** The most common provenance gap.
- **Container images referenced by tag.** The tag moves; the reproduction differs.
- **Unpinned transitive dependencies.** Yesterday's environment cannot be rebuilt.
- **Data version not recorded.** The model cannot be reproduced, explained or defended.
- **Pickle in production.** Fragile and a code-execution risk across a trust boundary.
- **Preprocessing versioned separately from the model.** A quiet, damaging mismatch.
- **No reverse lookup from time to model version.** Incident investigation is guesswork.
- **Tracking with no hypotheses.** Hundreds of runs, no knowledge.
- **Negative results unrecorded.** The team repeats failed experiments annually.
- **Test set used during search.** It is a validation set now; report accordingly.
- **Search budget unreported.** The number is uninterpretable.
- **Image retention shorter than model retention.** Old models are unreproducible by policy.

## 5. Exercises

### Warm-up (30 min)

1. Give the four levels of the reproducibility spectrum and say which you would target for a
   typical production system and why.
2. Give five sources of non-determinism, and explain why floating-point non-associativity matters.
3. Explain why a noise floor must be established before any improvement can be claimed.

### Core (3.5 h)

4. Complete Stages 1–3 and report the noise floor, the post-seeding variance, and the cost of
   deterministic mode with your judgement about paying it.
5. Complete Stages 4–5: automatic provenance and a registry with the reverse lookup tested.
6. Complete Stage 6 and list everything that went wrong reproducing a month-old model.
7. Complete Stage 7 and deliver the eight experiments with conclusions and the five-bullet summary.

### Challenge

8. Complete Stages 8–9, including the multiple-comparisons quantification and the ONNX numerical
   verification and version-mismatch demonstration.
9. Build a **reproducibility auditor** for an ML repository: given a model artifact, verify
   automatically that (a) its provenance record is complete, (b) the referenced commit exists and
   the tree was clean, (c) the referenced data version is still retrievable, (d) the container image
   digest is still pullable, and (e) a re-run produces a model within the noise floor. Report a
   pass/fail per artifact with the specific gap. Then run it against every model you have ever
   trained in this course and report the pass rate — which will be lower than you expect, and the
   pattern of failures tells you which discipline slipped first when you were busy.

## 6. Self-check

1. Give the four levels of reproducibility and their relative costs.
2. Give five sources of non-determinism in training.
3. What must be recorded for provenance, and what is the test for completeness?
4. Why is a container tag insufficient?
5. Why must the noise floor precede any comparison?
6. Give five failures of experiment tracking discipline.
7. Why does random search beat grid search at the same budget?
8. What is the multiple comparisons hazard in hyperparameter search, and what controls it?
9. Why is pickle wrong as a production artifact format?
10. Why must preprocessing be versioned with the model, and what happens when it is not?

## 7. Primary sources

- **Bergstra & Bengio, "Random Search for Hyper-Parameter Optimization" (JMLR 2012).**
- **Li et al., "Hyperband" (JMLR 2018)** and Li et al., "A System for Massively Parallel
  Hyperparameter Tuning" (ASHA, MLSys 2020).
- Pineau et al., "Improving Reproducibility in Machine Learning Research" (JMLR 2021) — the NeurIPS
  reproducibility programme's findings; read for what actually fails.
- Sculley et al., "Winner's Curse? On Pace, Progress, and Empirical Rigor" (ICLR workshop 2018) —
  short, and directly about §2.4 and §2.5.
- Gundersen & Kjensmo, "State of the Art: Reproducibility in Artificial Intelligence" (AAAI 2018).
- The MLflow documentation on the model registry and the model format; DVC's documentation on data
  versioning.
- The ONNX documentation, particularly on operator support and conversion verification.
- Zinkevich, *Rules of Machine Learning*, rules 1–15 — mostly about this lesson's discipline.

---

**Previous:** [L02](L02-data-pipelines-and-features.md) · **Next:**
[L04 — Training Infrastructure](L04-training-infrastructure.md)
