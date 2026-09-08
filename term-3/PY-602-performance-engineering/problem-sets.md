# PY-602 — Problem Sets

**A note on evidence.** This course's standard is a number you produced with the method stated. A
measurement without a noise floor is not a measurement; a percentile without a distribution is not a
percentile; a speedup without a baseline anyone can reproduce is a claim.

**A note on the constructions.** Each set builds on the lessons' §3 constructions. Where a part
names lesson stages, do those first.

**A note on predictions.** Several parts ask you to predict a result before measuring it. Write the
prediction down and keep the wrong ones in your submission. The gap between what you expected and
what happened is the most valuable thing this course produces, and erasing it wastes the exercise.

---

## Problem Set 1 — A Rigorous Optimization Study
**Covers L01–L03 · Budget: 14–18 hours**
*Builds on: L01 §3 (all stages), L02 §3 (all stages), L03 §3 stages 1–7*

The point of this set is **method**. A modest speedup measured impeccably scores far above a
large speedup measured carelessly.

**Part A — The harness (L01 X1).** A reusable benchmarking framework: environment capture,
warm-up, interleaved randomized A/B, raw-sample retention, bootstrap confidence intervals,
HDR histograms, plots, and a markdown report. You will use this for the rest of the program.

**Part B — The study (L01 §3).** All eight steps on a real slow thing: the stated question
and adoption criterion, the baseline distribution (plot it; explain the modes), **the noise
floor** measured by running unchanged code as both A and B, the written prediction, the
change, the result with effect size and interval, the coordinated-omission check, and the
failed attempts.

**Part C — Profiling (L02 §3).** All eight steps: the regime measurement, the sampled flame
graph with your three suspects *and the surprise*, the count confirmation, the complexity
check, `perf stat -d` counters, the import-time report, the differential flame graph, and the
end-to-end result.

**Part D — Memory (L03 §3).** All seven steps: the RSS shape, the reconciliation with the
honest unattributed remainder, the type census, the peak attribution, the ladder table with
**both memory and runtime columns**, the columnar prediction with its error, and the GC
tuning results.

**Part E — Profiler comparison (L02 C2).** Five profilers on one workload: overhead,
attribution, and what each cannot see. Then five situations and which you would use.

**Design note (1,200–1,600 words).** The gap between your prediction and the profile, and
what caused it. What the noise floor told you about which optimizations were even measurable
on your machine. The failed attempts and why they failed. Whether the end-to-end improvement
justified the effort, honestly.

### Marking emphasis

Method. A 40× speedup reported without a noise floor, a stated baseline, and a description of what
varied between runs is not a result. The methodology carries more marks than the speedup.

---

## Problem Set 2 — A Data-Structure Redesign, Predicted and Measured
**Covers L04–L06 · Budget: 14–18 hours**
*Builds on: L04 §3 (all stages), L05 §3 stages 1–7, L06 §3 (all stages)*

**Part A — Prediction (L04 §3).** Compute on paper the bytes per record for the current
representation, then measure. **A prediction within 20% is the bar**; iterate until you meet
it and report what you had failed to account for.

**Part B — The alternatives (L04 §3 steps 3–4).** Predicted and measured memory for six
representations, plus the access-pattern timing table (scan one field, scan all, random
access, key lookup, append). Note where the ranking changes by operation.

**Part C — Vectorization (L05 §3).** All eight stages: the baseline loop, naive NumPy with
the temporary-allocation accounting, `out=` elimination, memory-order check with cache
counters, `numexpr` and Numba, the dtype experiment **with the numerical difference
measured**, the query-engine comparison, and the four-column report (time, memory, LOC,
readability).

**Part D — Cache behaviour (L06 §3).** All nine steps: the working-set prediction, the
counters confirming or refuting it, the traversal-order ratio, the AoS→SoA change with a
prediction derived from cache-line utilization, the dtype shrink, the pass fusion, the
tile-size sweep with its optimum related to your cache sizes, sorting for locality, and the
report of prediction/measurement gaps.

**Part E — The universal principle (L06 C4).** The same locality mistake identified at three
levels — cache, storage, and network — in a system you know, with the cost of each estimated.

**Design note (1,500–2,000 words).** Where your predictions were wrong and why (this is the
assessed centrepiece — a correct prediction demonstrates a working mental model). Which
change gave the largest end-to-end improvement and whether it was the one you expected. The
representation you chose and the access pattern that justified it. What you would tell a
colleague about when *not* to do this redesign.

### Marking emphasis

The prediction. Predict the memory and time cost before measuring, in writing. The marks are in
the gap analysis, and a wrong prediction well explained scores above a right one unexplained.

---

## Problem Set 3 — An Extension and a Capacity Model
**Covers L07–L09 · Budget: 16–20 hours**
*Builds on: L07 §3 stages 1–6, L08 §3 (all stages), L09 §3 (all stages)*

**Part A — Rule out the alternatives (L07 §3 step 0).** Numba, PyPy, and a representation
change, each measured. **If Numba gets you 80% of the way, say so** — that is a successful
outcome and the design note should own it.

**Part B — The extension (L07 §3).** Steps 1–8: the Cython progression (untyped → typed →
unsafe → `nogil`) with the `cython -a` observations, both PyO3 variants (copying and
borrowing) with the copy cost isolated, the two-thread scaling, the **end-to-end number with
the Amdahl calculation**, and the packaging set up with `cibuildwheel` and timed.

**Part C — I/O (L08 §3).** All nine steps: the syscall histogram, the cold/warm read table,
the write table **with the durability column** (the assessed part), the queue-depth curve,
the serialization swap, the compression crossover, and the end-to-end result.

**Part D — Capacity (L09 §3).** All nine steps: the four limits with the binding one
identified, the open-loop characteristic curve, the USL fit with the *physical* cause of α
and β located, the capacity model compared against what you actually run, the cost per
million requests, the three degraded-mode curves, and the observability gaps closed.

**Part E — The recommendation.** Should the extension ship? With: the end-to-end speedup, the
effort, the packaging burden, the contributor cost, the maintenance commitment, and — from
Part D — whether it changes the instance count. A recommendation of "no, use Numba and do not
ship an extension" is a full-credit answer.

**Design note (1,500–2,000 words).** The Amdahl reality check: how much did the 20× function
speedup change the program? Which of the four capacity limits actually binds, and what that
means for where optimization effort should go. What the degraded-mode curves revealed. Your
stopping criterion, and whether you have met it.

### Marking emphasis

Honesty about the model. A capacity model is a set of assumptions with arithmetic attached. State
the assumptions, state which is most likely to be wrong, and say what would falsify the model.

---

## Term 3 build artifact (shared with PY-601)

The high-throughput service, with a complete performance report:

- The characteristic curve, normal and degraded.
- The capacity model with the binding constraint identified.
- The profile that located the bottleneck, and the differential after the change.
- The memory model, predicted and measured.
- Before/after end-to-end numbers with full methodology: machine, OS, Python build, medians
  and spread, statistical treatment, and the noise floor.
- **An honest account of what did not work.** Every optimization study has failed attempts;
  a report without them is not credible.

---

## Course position paper (1,500 words)

**Driving question: where does the time actually go?**

Claim, grounds, rebuttal, limits. Grounds from your own measurements: the prediction/profile
gap in PS1, the prediction accuracy in PS2, and the Amdahl reality check in PS3. The rebuttal
must state fairly the position that performance engineering is largely a waste of effort in
modern software — that hardware is cheap, developer time is not, and the correct response to
a slow service is a larger instance — and then answer it. Note that this position is right
more often than engineers like to admit, and an answer that concedes where it is right will
score higher than one that does not.

---

## Submission checklist (per set)

- [ ] Code in `courses/py602/psN/`, runnable from a clean clone.
- [ ] Every measurement reproducible: the script, the raw samples, and `scripts/envinfo.py`
      output committed.
- [ ] Medians and spread, never a single run. Noise floor stated.
- [ ] Every plot has axes, units, and the machine/version in the caption.
- [ ] Predictions recorded *before* measurements, in the commit history.
- [ ] `NOTE.md` — the design note.
- [ ] `MARK.md` — self-marked, one line per criterion.
- [ ] `log/failures.md` updated with every prediction you got wrong.
