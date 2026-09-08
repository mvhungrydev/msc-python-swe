# PY-602 · Lesson 09 — Capacity, Queueing, and Performance in Production

**Estimated study time:** 3.5 hours
**Prerequisites:** L01, L02; PY-601 L09

---

## 1. Orientation

Everything so far has been about making code faster. This lesson is about the question that
actually gets asked: **how many machines do we need, and what happens when we get it wrong?**

The gap between a benchmark number and a capacity number is large and it is where most
performance work fails to land. A function that is 3× faster in a microbenchmark may change
nothing about how many instances you run — because the instance count was set by memory, or
by a connection limit, or by the p99 requirement rather than the mean.

## 2. Theory

### 2.1 The capacity model

The minimal model, which you should be able to write for any service:

```
capacity_per_instance = min(
    cpu_bound_limit,       # cores / cpu_seconds_per_request
    memory_bound_limit,    # memory / memory_per_concurrent_request
    concurrency_limit,     # max in-flight (connections, threads, pool size)
    downstream_limit,      # what dependencies will accept
)
instances = ceil(peak_rps / (capacity_per_instance * target_utilization)) + headroom
```

Two things fall out immediately:

- **One of those four is the binding constraint, and it is often not CPU.** Optimizing CPU
  when memory binds changes nothing. Identify the binding constraint *first*; this is the
  single most valuable step in the lesson.
- **`target_utilization` is not 95%.** §2.3.

Then the harder questions the model forces: what is peak versus average (usually 3–10× for a
consumer service, and the ratio is a fact about your traffic that you should know); what is
the growth rate; what is the failure domain (if one of three AZs is lost, the other two must
carry everything, so you are provisioning for N+1 or N+2); and how fast can you scale
(if scaling takes five minutes and your spike takes thirty seconds, autoscaling does not
help).

### 2.2 The Universal Scalability Law

Amdahl says speedup plateaus. Reality is worse: throughput often *decreases* past some
concurrency.

```
C(N) = N / (1 + α(N−1) + βN(N−1))
```

- **α — contention.** Serialized work: a lock, a single-threaded stage, the GIL.
- **β — coherency.** The cost of keeping N workers consistent: cache-line ping-pong,
  distributed coordination, lock handoff. Grows as N², which is why the curve turns *down*.

Fit it to measured data (throughput at increasing concurrency) with least squares. The
outputs are actionable:

- **α tells you the serial fraction** — where to look for a lock or a single-threaded stage.
- **β tells you whether adding capacity will hurt.** If β > 0 and you are near the peak,
  more workers make things worse.
- **The peak N** tells you the maximum useful concurrency, which is a number you can set as a
  configuration limit.

Every engineer who has added workers and watched throughput drop has met β. Measuring it
converts that experience into a number you can act on.

### 2.3 Utilization and latency

From PY-601 L09 §2.6, and it belongs here too because it is the number capacity plans get
wrong:

For an M/M/1 queue, mean wait ∝ ρ/(1−ρ):

| Utilization | Relative wait |
|---|---|
| 50% | 1× |
| 70% | 2.3× |
| 80% | 4× |
| 90% | 9× |
| 95% | 19× |
| 99% | 99× |

**Plan for 60–70%.** The last 30% of capacity costs an order of magnitude in tail latency,
and it removes your ability to absorb a burst or lose an instance.

The Kingman approximation adds: waiting time scales with the *variability* of arrivals and
of service times, so a service with a p99 50× its median queues far worse at the same
utilization than one with a tight distribution. **Reducing variance is often worth more than
reducing the mean**, and it is almost never what people optimize.

### 2.4 Benchmarking a service properly

Microbenchmarks (L01) do not tell you capacity. A load test does, if it is designed right:

**Open-loop, not closed-loop.** A closed-loop generator (send, wait for response, send next)
cannot exceed the system's own rate, so it never shows overload behaviour and it exhibits
coordinated omission (L01 §2.3). Use an open-loop generator that sends at a fixed rate
regardless of responses, and record latency from the *intended* send time. `wrk2`, `k6`,
`vegeta`, and `locust` (in the right mode) can do this; naive scripts almost never do.

**Ramp, do not step.** Increase the rate gradually and record the full curve: throughput,
goodput (successful responses within the SLO), latency percentiles, error rate, CPU, memory,
and queue depths. The interesting region is around the knee, and a single-point test misses
it entirely.

**Realistic workload mix.** Production traffic is not one endpoint. Cache hit ratios,
payload sizes, and key distributions must match, or you are measuring a different system.
A cache with a 99% hit rate in the test and 70% in production is not the same service.

**Realistic data volume.** A 10,000-row table and a 100-million-row table have different
query plans (DI-721 L02). Benchmarks on small data are routinely misleading in exactly this
way.

**Warm up.** JIT, caches, connection pools, page cache. Then discard the warm-up.

**Test the failure modes**, not just the happy path: a slow dependency, a dependency
returning errors, an instance lost mid-test, a cache flushed.

The deliverable is the **characteristic curve**: throughput and latency percentiles against
offered load, up to and past saturation. That single chart answers most capacity questions
and almost no team has one.

### 2.5 What to measure in production

The **USE method** (Gregg) for resources — for every resource, track:

- **Utilization** — percentage of time busy.
- **Saturation** — queued work waiting.
- **Errors** — error events.

Applied to CPU, memory, disk, network, and each connection pool. It is a checklist, and its
value is that it is exhaustive: a resource with no saturation metric is a resource you cannot
diagnose.

The **RED method** for services — Rate, Errors, Duration (as a histogram). Per endpoint, per
dependency.

The **four golden signals** (SRE book) — latency, traffic, errors, saturation.

What is most often missing, in my experience of reading dashboards:

- **Saturation metrics.** Queue depths, pool waits, thread-pool queue length. Utilization
  without saturation cannot distinguish "busy" from "falling behind".
- **Latency histograms rather than averages.** And mergeable ones (t-digest, HDR) so
  aggregation is valid (L01 §2.3).
- **Per-dependency breakdown.** "The service is slow" versus "the service is slow because
  dependency X's p99 tripled".
- **Goodput**, not just throughput. Responses that arrived after the client gave up are not
  successes.
- **Saturation of the *thread pool*** in an async service — the single most common blind
  spot (PY-601 L06 §2.6).

### 2.6 Cost as a performance metric

For a cloud service the honest performance metric is often **cost per unit of work**:
dollars per million requests, per GB processed, per model inference.

It composes the things that matter and it is the number that gets optimization work funded:

```
cost_per_million_requests
    = (instance_cost_per_hour / requests_per_hour_per_instance) × 10⁶
```

Track it over time. A 20% CPU improvement that does not reduce instance count is worth
nothing on this metric, which is a useful corrective; conversely, a change that lets you run
on smaller instances shows up immediately.

The related lever people forget: **an algorithmic or representational change that halves
memory** may let you use an instance type with half the memory and the same CPU, which can
be a 40% cost reduction with no latency change at all.

### 2.7 Performance regressions

Performance decays the way architecture does (SE-521 L08): gradually, from individually
reasonable changes.

The defences, in order of value per unit effort:

1. **Continuous profiling** (L02 §2.6) with cross-deploy diffs. Catches regressions that no
   benchmark covers, because it profiles the real workload.
2. **A load test in a pre-production environment**, run on a schedule, with the
   characteristic curve compared against the previous run.
3. **A CI performance tripwire** — a benchmark with a generous threshold (say 2×) to catch
   catastrophic regressions. Not a precise gate; CI machines are too noisy (L01 §2.4), and a
   tight threshold produces flaky failures that get the check disabled.
4. **SLO-based alerting with error budgets.** The production version of a fitness function
   (SE-521 L08 §2.2).
5. **Import-time and startup-time budgets** for CLIs and serverless.

And the cultural one: **a performance review step for changes to known-hot paths.** A
CODEOWNERS entry on the ten hottest modules costs nothing and catches the change that adds a
database call inside a loop.

### 2.8 Knowing when to stop

Performance work has no natural end, so define one:

- **Against an SLO.** "p99 < 200 ms at 2× current peak." When you meet it, stop.
- **Against cost.** "Below $X per million requests."
- **Against the next bottleneck.** When the profile is flat — no single item above ~5% —
  further work has poor returns and you should stop or restructure.
- **Against effort.** When the next 10% costs more than it returns, in engineering time or
  in complexity.

Write the criterion down before starting (L01 §2.1). Without it, performance work expands to
fill the available time, and it produces complexity that someone maintains forever.

## 3. Construction: a capacity model and a load test

Take a real service.

**Step 1 — the four limits.** Measure each: CPU seconds per request (from
`process_time` under load), memory per concurrent request (RSS delta divided by concurrency),
the configured concurrency limits (threads, connections, pool sizes), and each downstream's
capacity. **Identify the binding constraint.** Most people have never done this and are
surprised by the answer.

**Step 2 — the load test.** Build an open-loop test with a realistic mix and realistic data
volume. Verify it is open-loop by checking that offered rate is achieved regardless of
response latency.

**Step 3 — the characteristic curve.** Ramp from 10% to 200% of current peak. Record
throughput, goodput, p50/p99/p999, errors, CPU, memory, and every queue depth. Plot. Identify
the knee, and identify *which* of the four limits binds at the knee — confirming or refuting
step 1.

**Step 4 — fit the USL.** Throughput against concurrency. Extract α and β. Identify the
physical cause of each: what is serialized (α), and what coordinates (β). Report the peak
concurrency.

**Step 5 — the capacity model.** Write it out, with the numbers, the peak-to-average ratio,
the failure-domain requirement, and the utilization target with its justification. Then
compute the instance count and compare with what you are actually running. Explain the
difference; it is usually informative in both directions.

**Step 6 — cost per unit of work.** Compute it. Then compute what it would be if you fixed
the binding constraint from step 1, and what that would be worth annually.

**Step 7 — degraded-mode tests.** Re-run the ramp with: a downstream at 10× its normal p99,
a downstream returning 50% errors, and one instance killed mid-test. Record the curve for
each. These tell you what happens on the worst day, and they are the tests nobody runs.

**Step 8 — the observability gap.** For every number you had to measure specially for this
exercise, ask why it is not already a dashboard. Add the missing saturation metrics.

**Step 9 — the report.** Capacity model, characteristic curves (normal and degraded), the
USL fit with its physical explanation, the cost figure, the recommendation, and the stopping
criterion for further work.

## 4. Failure modes

- **Optimizing the non-binding constraint.** CPU work on a memory-bound service.
- **Closed-loop load testing.** Cannot show overload; coordinated omission.
- **Single-point load tests.** No curve, no knee, no information about the interesting
  region.
- **Unrealistic data volume or workload mix.**
- **Planning for 95% utilization.**
- **No failure-domain headroom.** Losing one AZ takes the service down.
- **Autoscaling slower than the spike.**
- **Averages instead of histograms**; averaging percentiles across instances.
- **Utilization without saturation metrics.**
- **Throughput without goodput.**
- **No cost metric**, so nobody can evaluate whether an optimization was worth it.
- **No stopping criterion.** Performance work without end.
- **A CI performance gate with a tight threshold.** Flaky, then disabled.
- **Never testing degraded modes.** The worst day is the untested case.

## 5. Exercises

### Warm-up (25 min)

**W1.** For a service you know, estimate all four limits from §2.1 and name the binding one.
Then measure and check.

**W2.** Show that a closed-loop load generator cannot exceed the service's own rate, and that
an open-loop one can. Report the p99 each measures against a service that stalls periodically.

**W3.** Compute the utilization/latency table for your own service's measured service-time
distribution, and mark where you are running.

### Core (2.5 h)

**C1 — The capacity study.** Complete §3, all nine steps. Deliverable: the four limits with
the binding one identified, the characteristic curve, the USL fit with physical causes, the
capacity model with the instance count compared against reality, the cost figure, the three
degraded-mode curves, and the added observability.

**C2 — USL in practice.** Measure throughput against worker count for a real system, well
past the peak. Fit α and β. Then *find* the physical causes: for α, profile and locate the
serialized section; for β, look for a shared lock, a shared cache line, or a coordinating
dependency. Fix one and re-measure the coefficients. Report the change.

**C3 — Variance reduction.** Identify the largest source of service-time variance in a
service (a cache miss path, a slow query, a GC pause, a retry). Reduce it without changing
the mean. Measure p99 and the queueing behaviour at 70% utilization before and after. Report
the improvement and connect it to Kingman.

**C4 — Cost model.** Build a cost-per-million-requests model for a service, decomposed by
resource. Then evaluate three proposed optimizations against it, including one that improves
CPU with no instance-count change (and therefore no cost change). Present it as you would to
a manager deciding what to fund.

### Challenge

**X1.** Build a complete performance-regression system: continuous profiling with deploy
diffs, a scheduled load test producing the characteristic curve, a CI tripwire with a
justified threshold, and SLO alerting with an error budget. Run it for a month. Report what
it caught, what it missed, and its false-positive rate.

**X2.** Take a service and produce a full capacity plan for 10× growth: the binding
constraint at each stage of growth (it will change), the architectural changes required at
each stage, the cost curve, and the lead time for each change. Present it as a document a
technical leadership group could act on. This is the deliverable that turns performance
engineering into a business input, and it is the skill this course exists to build.

## 6. Self-check

1. Give the four limits in a capacity model and say why identifying the binding one comes
   first.
2. State the USL and what α and β mean physically.
3. What utilization should you plan for, and why not 95%?
4. Why does variance matter as much as the mean for queueing?
5. What is wrong with closed-loop load testing?
6. Give the USE and RED methods, and name the metric most often missing.
7. Why is cost per unit of work a good performance metric?
8. Give four stopping criteria for performance work.

## 7. Primary sources

- Gregg, *Systems Performance*, 2nd ed., ch. 2 (USE method), ch. 12 (benchmarking). Chapter
  12's "benchmarking sins" section should be read by everyone who runs a benchmark.
- Gunther, *Guerrilla Capacity Planning* — the USL.
- Beyer et al., *Site Reliability Engineering*, chs. 4 (SLOs), 21 (overload), 22 (cascading
  failures).
- Gil Tene, "How NOT to Measure Latency".
- Little (1961); Kingman (1961) for the variability result.

---

**Previous:** [L08](L08-io-and-syscalls.md) ·
**Course complete.** Next: [problem sets](problem-sets.md), [exam](exam.md), and
[Term 4](../../term-4/CS-621-algorithms-complexity/syllabus.md).
