# PY-602 · Lesson 01 — Measurement: Methodology and Statistics

**Estimated study time:** 4 hours
**Prerequisites:** none

---

## 1. Orientation

Knuth's line is quoted constantly and almost always truncated. In context (1974):

> "There is no doubt that the grail of efficiency leads to abuse. Programmers waste enormous
> amounts of time thinking about, or worrying about, the speed of noncritical parts of their
> programs... **We should forget about small efficiencies, say about 97% of the time:
> premature optimization is the root of all evil.** Yet we should not pass up our
> opportunities in that critical 3%. A good programmer will not be lulled into complacency by
> such reasoning, he will be wise to look carefully at the critical code; but only after that
> code has been identified."

The quoted fragment is used to dismiss performance work. The full passage says something
sharper: **optimize the critical 3%, and identify it by measurement.** Both halves matter,
and the second half is the one this lesson is about.

The reason to start with statistics rather than with profilers: most performance work is
invalidated not by using the wrong optimization but by *believing a bad measurement*.

## 2. Theory

### 2.1 The methodology

Before any measurement:

1. **State the question.** "Is this faster?" is not one. "Does change X reduce p99 latency of
   endpoint Y under workload Z by more than 10%?" is.
2. **State the workload.** Real distribution, real sizes, real cardinality. A benchmark on
   uniform random data when production data is Zipf-distributed measures a different program.
3. **State the metric.** Throughput and latency are different, often opposed, questions.
   So are mean, p50, p99, p999, and maximum. Choose before measuring, or you will choose
   afterwards to suit the result.
4. **Predict.** Write down what you expect and why. A prediction that is wrong is the most
   informative outcome available, and if you do not write it down first you will not notice
   you were wrong.
5. **Measure the baseline.** Enough runs to know its variance.
6. **Change one thing.**
7. **Re-measure identically.**
8. **Decide** with a stated criterion, including effect size, not only significance.

Steps 4 and 8 are the ones people skip and the ones that separate engineering from
tinkering.

### 2.2 Throughput versus latency

They are not the same and improving one frequently harms the other:

- **Batching** improves throughput (amortized per-item overhead) and worsens latency (items
  wait for a batch).
- **Caching** improves both — until the cache introduces a p99 stall on miss+fill.
- **Adding workers** improves throughput to a point, then increases latency through
  contention and queueing (PY-601 L09 §2.6).
- **Compression** trades CPU for bandwidth: better throughput on a slow link, worse latency
  on a fast one.

**Which you care about is a product question.** Answer it before optimizing, and write the
answer down.

### 2.3 Distributions, not averages

Latency distributions are right-skewed and often multi-modal (cache hit versus miss; fast
path versus slow path). The mean is therefore a poor summary and can be dominated by a tail
you did not intend to measure.

Report: **p50, p90, p99, p999, and max**, plus the number of samples. If you report one
number, report p99, because that is what users experience often enough to notice.

Three traps specific to percentiles:

- **Percentiles do not average.** The mean of two servers' p99s is not the p99 of the pair.
  Aggregate the *raw* samples or use a mergeable sketch (t-digest, HDR histogram, DDSketch).
  Averaging percentiles across instances is a very common monitoring error, and it always
  understates the tail.
- **Percentiles do not compose across a call chain.** A request touching three services each
  at p99 = 10 ms does not have p99 = 30 ms; it is worse, because of the probability of
  hitting at least one slow hop (PY-601 L09 §2.6).
- **Coordinated omission.** If your load generator waits for a response before sending the
  next request, then when the system stalls the generator stops issuing — so the slow period
  is *under-sampled* and the measured tail is far better than reality. Gil Tene's argument;
  the fix is to send at a fixed rate regardless of response, and to record latency from the
  *intended* send time. Most naive benchmarks have this bug, and it typically understates
  p99 by an order of magnitude.

### 2.4 Variance and its sources

Two runs of identical code differ. Sources, roughly in order of size:

- **Thermal and frequency scaling.** A laptop throttles after ~30 seconds of load. Modern
  CPUs boost then drop; the first run is faster than the tenth.
- **Other processes**, including your editor, browser, indexer, and antivirus.
- **Address-space layout randomization** and environment size. Changing an environment
  variable's length shifts stack alignment and can change performance by several percent.
  This is the astonishing result in Mytkowicz et al. (2009) — measured effects larger than
  the "improvements" being published.
- **Memory allocator state**, fragmentation, and page-cache warmth.
- **CPython specialization warm-up** (PY-501 L08 §2.6). The first few thousand iterations
  run unspecialized.
- **GC pauses** (PY-501 L09 §2.2), which are correlated with the object graph, not with your
  loop.
- **Turbo, hyperthreading, and NUMA placement.**
- **Virtualization noise** on cloud instances — often the largest source, and the least
  controllable.

Mitigations, in order of value:

```bash
# Linux
sudo cpupower frequency-set -g performance      # fix the governor
taskset -c 2,3 python bench.py                  # pin cores
sudo sysctl -w kernel.randomize_va_space=0      # for experiments only
nice -n -20 ...                                 # priority
```

Plus: close everything; run on mains power; discard warm-up runs; interleave A and B
alternately rather than running all of A then all of B (this cancels drift); and randomize
the order.

**On a laptop or a cloud VM, accept that you cannot control this** and compensate with more
samples and a robust statistic — and *say so* in the report.

### 2.5 Statistics you actually need

**Use the median and a robust spread**, not the mean and standard deviation. Timing
distributions have heavy right tails; the mean is dragged by outliers that represent
interference rather than your code.

- **Median** for the central tendency.
- **Median absolute deviation (MAD)** or the interquartile range for spread.
- **Minimum** is defensible for pure CPU microbenchmarks — it is the run least contaminated
  by interference, which is why `timeit` reports it. It is *wrong* for anything where
  variability is part of the phenomenon (I/O, GC, JIT, cache effects).

**Comparing two versions.** Do not eyeball two medians:

1. Collect n ≥ 30 samples of each, interleaved.
2. Compute the difference in medians.
3. Bootstrap a confidence interval for that difference (resample with replacement 10,000
   times, take the 2.5th and 97.5th percentiles of the differences). Twenty lines of code, no
   distributional assumptions.
4. Report the effect size and the interval, not a p-value.

```python
import random, statistics

def bootstrap_diff(a: list[float], b: list[float], n: int = 10_000) -> tuple[float, float, float]:
    obs = statistics.median(b) - statistics.median(a)
    diffs = []
    for _ in range(n):
        ra = [random.choice(a) for _ in a]
        rb = [random.choice(b) for _ in b]
        diffs.append(statistics.median(rb) - statistics.median(ra))
    diffs.sort()
    return obs, diffs[int(0.025 * n)], diffs[int(0.975 * n)]
```

**State a practical threshold in advance.** "We will adopt this if it improves p99 by more
than 5% with the interval excluding zero." Otherwise you will accept a 1.5% improvement that
is within noise, and next quarter someone will "improve" it back.

### 2.6 Microbenchmarks and their traps

```python
import timeit
timeit.repeat("f(x)", setup="...", repeat=10, number=10000)
```

`timeit` disables the cycle collector by default and reports totals per repetition. Take the
minimum across repetitions for CPU-bound code, and know why.

The traps, each of which invalidates results silently:

- **Dead code elimination.** Not by CPython (no optimizer to speak of), but constant folding
  is real (PY-501 L08 §2.4): `timeit("2**10")` measures a `LOAD_CONST`.
- **Not measuring what you think.** `timeit("[x for x in range(1000)]")` includes building
  the range and the list; if you meant to measure the loop body, you measured allocation.
- **No warm-up.** On CPython 3.11+, specialization needs thousands of iterations. Measuring
  cold measures a different interpreter.
- **Cache effects.** A benchmark whose working set fits in L1 tells you nothing about
  production, where it does not.
- **Unrealistic data.** Sorted input to a sort; all-distinct keys to a dict; short strings
  where production has long ones.
- **Measuring in isolation.** A function that is 40% faster alone may be irrelevant in a
  program where it is 2% of the time — Amdahl again.

`pyperf` handles several of these properly (multiple processes to average over ASLR,
warm-up, calibration, and a statistical report). For anything you will publish or act on,
prefer it to hand-rolled `timeit`.

### 2.7 Clocks

| Clock | Use |
|---|---|
| `time.perf_counter()` | **the default for elapsed time.** Monotonic, highest resolution |
| `time.perf_counter_ns()` | integer nanoseconds; avoids float rounding for short intervals |
| `time.monotonic()` | monotonic, coarser; for timeouts |
| `time.process_time()` | CPU time of this process — excludes sleep and I/O wait |
| `time.thread_time()` | CPU time of this thread |
| `time.time()` | **wall clock — never for measurement.** NTP can move it backwards |

Measuring `perf_counter` and `process_time` together tells you whether time went to CPU or
to waiting, which is the first branch of PY-601 L01's decision tree and takes one extra line.

Resolution: check `time.get_clock_info("perf_counter")`. Intervals shorter than a few
microseconds need many iterations per sample, not a better clock.

### 2.8 The optimization loop

1. **Is it too slow?** Against a stated requirement. If there is no requirement, stop.
2. **Where does the time go?** Profile (L02). Do not guess.
3. **Is there a better algorithm?** An O(n log n) replacing an O(n²) beats every constant
   factor (CS-621). Check this before anything else.
4. **Can the work be avoided?** Cache, precompute, lazy-evaluate, push down to the database,
   or answer a cheaper question.
5. **Can it be vectorized or batched?** One dispatch for a million elements (L05).
6. **Can it be parallelized?** (PY-601.)
7. **Can it be moved to C/Rust?** (L07.)
8. **Micro-optimize.** Last, and least.

Most people start at 8. The ordering is by expected payoff per unit effort, and it is
roughly logarithmic: steps 3–4 routinely give 10–1000×, step 5 gives 10–100×, step 6 gives
2–8×, step 7 gives 2–50×, step 8 gives 1.05–1.5×.

## 3. Construction: a defensible optimization study

Take a real slow thing — a report generation, an import, an endpoint.

**Step 1 — the question and the criterion.** Write both down, before measuring. Include the
threshold at which you would adopt the change.

**Step 2 — the harness.** A script that: records the environment (`scripts/envinfo.py` from
the lab setup), warms up, runs A and B interleaved n ≥ 30 times each in a randomized order,
records raw samples to a file, and prints medians with a bootstrap interval. Commit the raw
data alongside the analysis.

**Step 3 — characterize the baseline.** Not just "how slow" — the *distribution*. Plot a
histogram. Is it bimodal? (Almost certainly, and finding out why is often the whole
optimization.) Measure `perf_counter` versus `process_time` to see whether it is CPU or
waiting.

**Step 4 — quantify the variance floor.** Run the *unchanged* code as both A and B. The
measured "difference" is your noise floor. **Any improvement smaller than this is not
measurable on this machine**, and knowing the number stops you chasing it. This step takes
ten minutes and almost nobody does it.

**Step 5 — predict.** Write down the expected improvement and the reasoning. Then profile
(L02) and revise your prediction. The gap between prediction and profile is the most
educational artifact in this course.

**Step 6 — change one thing. Re-measure. Report.** Effect size, interval, and the decision
against the criterion from step 1.

**Step 7 — the coordinated-omission check.** If this is a service benchmark, verify your load
generator sends at a fixed rate rather than waiting. If it does not, fix it and re-measure —
and report both numbers, because the difference is usually startling.

**Step 8 — write what did not work.** Every optimization study has failed attempts. Publish
them; they are more useful to the next person than the one that worked.

## 4. Failure modes

- **Optimizing without a requirement.** Infinite work, no criterion for stopping.
- **Guessing the bottleneck.** The famous one, and still the most common.
- **Means and standard deviations on latency data.**
- **Averaging percentiles.**
- **Coordinated omission.** Understates p99 by an order of magnitude.
- **One run of each.**
- **A and B measured at different times.** Drift and thermal state confound them.
- **`time.time()` for measurement.**
- **No warm-up on CPython 3.11+.**
- **Unrealistic data or an unrealistically small working set.**
- **Not measuring the noise floor.**
- **Accepting an improvement within noise.**
- **Microbenchmarking a function that is 2% of runtime.**
- **Not recording the environment**, making the result unreproducible and therefore an
  anecdote.
- **Reporting only successes.**

## 5. Exercises

### Warm-up (30 min)

**W1.** Measure the same unchanged code as A and B, 50 samples each, interleaved. Report the
"improvement" and its confidence interval. That is your noise floor.

**W2.** Measure a function's latency 10,000 times and plot the histogram. Identify the modes
and explain each.

**W3.** Demonstrate coordinated omission: a closed-loop load generator versus an open-loop
one against a service that stalls for 500 ms once per second. Report both p99s.

### Core (2.5 h)

**C1 — The study.** Complete §3, all eight steps, on a real slow thing. Deliverable: the
harness, raw data, the noise floor, the prediction/profile gap, the result with effect size
and interval, the decision against your stated criterion, and the failed attempts.

**C2 — Statistics.** Implement the bootstrap comparison of §2.5. Then take a published
Python performance claim (a blog post, a library's README) and try to reproduce it with
proper methodology. Report whether it holds, and what the original's methodological gaps
were. Be fair — often the claim holds and the methodology was merely unstated.

**C3 — Mytkowicz replication.** Reproduce the environment-size effect: run an identical
benchmark with environment variables of increasing total length, and plot the timing. Report
the magnitude. Then read the paper and write 400 words on what it implies about published
performance results generally.

**C4 — Throughput versus latency.** Take a batching parameter (batch size, flush interval)
and plot throughput and p50/p99 latency against it. Identify the knee. Then state which
setting you would choose for two different product requirements, and why.

### Challenge

**X1.** Build a reusable benchmarking framework for your study repo: environment capture,
interleaved A/B, warm-up, outlier reporting, bootstrap intervals, HDR histograms, plots, and
a markdown report generator. Use it for every measurement in PY-602 and PY-601. The framework
is the deliverable; you will use it for years.

**X2.** Read Mytkowicz et al. (2009) and Gregg's *Systems Performance* ch. 2. Write 1,200
words on measurement bias: name five sources, describe an experiment that would detect each,
and propose a methodology checklist for your team. Then apply the checklist to a benchmark
someone else published and report what it does not satisfy.

## 6. Self-check

1. Give Knuth's full claim and both halves of what it requires.
2. Give the eight steps of the methodology, and name the two most often skipped.
3. Why are throughput and latency different questions, and give two changes that improve one
   and harm the other.
4. Why must you not average percentiles, and what do you do instead?
5. Explain coordinated omission and its typical magnitude.
6. Name five sources of measurement variance, and the mitigation for each.
7. Why median and MAD rather than mean and standard deviation? When is minimum defensible?
8. Give the eight-step optimization loop and the typical payoff of each step.

## 7. Primary sources

- Knuth, "Structured Programming with go to Statements" (Computing Surveys, 1974) — §1 and
  the passage in §2.1 above.
- Mytkowicz, Diwan, Hauswirth & Sweeney, "Producing Wrong Data Without Doing Anything
  Obviously Wrong!" (ASPLOS 2009).
- Gil Tene, "How NOT to Measure Latency" (talk, ~2015) — coordinated omission.
- Gregg, *Systems Performance*, 2nd ed., chs. 1–2.
- `pyperf` documentation, especially "How to get reproducible benchmark results".

---

**Next:** [L02 — Profiling: Deterministic, Sampling, and Attribution](L02-profiling.md)
