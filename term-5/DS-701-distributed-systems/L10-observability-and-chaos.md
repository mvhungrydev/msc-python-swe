# DS-701 · Lesson 10 — Observability, Tracing, and Chaos Engineering

**Estimated study time:** 4 hours
**Prerequisites:** L01, L02, L09; PY-602 L01 (measurement statistics)

---

## 1. Orientation

Every previous lesson in this course described a mechanism. This one describes how you find out
what the mechanisms are actually doing in a system you did not fully build, cannot fully hold in
your head, and cannot stop.

The distinction that organises the lesson:

> **Monitoring** answers questions you thought to ask in advance. **Observability** is the
> property of a system that lets you answer questions you did *not* think to ask in advance —
> without shipping new code.

Monitoring is a dashboard with the four graphs someone drew in 2019. Observability is being able
to ask "show me the p99 latency for requests from tenant 4471, on build 2f9c, that touched the
read replica" at 3 a.m., having never anticipated that question. The difference is not a product
you buy; it is a property of the *data you emit* — high-cardinality, high-dimensional,
event-shaped, and joinable.

The second half of the lesson is the natural consequence of L01's honesty about failure: if you
accept that partial failure is the normal state, then the only way to know how your system
behaves under partial failure is to **cause partial failure on purpose, in a controlled way,
before it happens to you.** That is chaos engineering, and it is an experimental science, not
vandalism.

## 2. Theory

### 2.1 The three signals, and why the framing is slightly wrong

The conventional taxonomy:

- **Metrics** — numeric time series, aggregated. Cheap, constant-cost per time bucket, excellent
  for alerting and trends. Fatally limited by **cardinality**: a metric labelled by user ID or
  request ID explodes the storage and usually the bill.
- **Logs** — discrete, timestamped records. Arbitrarily high cardinality, expensive at volume,
  historically unstructured and therefore hard to query.
- **Traces** — the causal path of a single request across services, as a tree of spans.

The framing is slightly wrong because it presents three separate pillars when what you actually
want is **one wide, structured event per unit of work, emitted once, from which all three views
are derived.** The event carries every dimension you know at the time — request ID, trace ID,
user, tenant, build SHA, region, instance, cache hit/miss, downstream latencies, error class,
queue wait, retry count. Metrics are aggregations over those events; traces are events joined by
trace ID; logs are the events themselves.

Emitting one wide event per request rather than fifteen scattered log lines is the single highest
leverage change most systems can make, and the reason is joinability: dimensions in the same
event can be correlated, dimensions in different events cannot.

**Cardinality is the crux.** The questions that actually resolve incidents are almost always
high-cardinality ones ("which tenant?", "which build?", "which instance?"). Metrics systems
cannot answer them by construction. This is why "we have dashboards" and "we have observability"
are different claims.

### 2.2 Distributed tracing

Dapper's model, now standardised as OpenTelemetry:

- A **trace** is one logical operation end to end, identified by a `trace_id`.
- A **span** is one unit of work within it: an ID, a parent ID, a start and end time, a status,
  and arbitrary key-value **attributes** plus timestamped **events**.
- **Context propagation** carries `trace_id` and the current `span_id` across process boundaries,
  in headers (the W3C `traceparent` header is the interoperable standard). *This* is the part that
  matters and the part that breaks: a trace that stops at a queue boundary, a thread pool, or a
  library that does not forward headers is a trace with a hole in it.

**Sampling.** Tracing every request at scale is prohibitive, so you sample:

- **Head-based**: decide at the start of the trace, usually a fixed probability. Simple, and
  propagates cleanly — but you decide before you know whether the request was interesting, so you
  will drop the slow and failed ones at the same rate as the boring ones.
- **Tail-based**: buffer spans and decide after the trace completes, keeping all errors, all slow
  traces, and a small sample of normal ones. Vastly better data; requires a collector that can
  buffer complete traces, which is real infrastructure.
- **Dynamic / weighted**: sample rare keys (an unusual endpoint, a specific tenant) at a higher
  rate and record the sampling weight so aggregates remain unbiased. This is the technique that
  makes tracing affordable *and* useful, and the weight is the part people forget.

**Clocks.** Span timings come from different machines' clocks (L02), so a child span can appear to
start before its parent, or a duration can be negative. Trace visualisations that look
paradoxical are usually clock skew, not causality violations — and this is exactly why the causal
relationships must be carried by the parent-child structure rather than inferred from timestamps.

### 2.3 What to alert on

The dominant failure mode of alerting is not missing alerts; it is **too many alerts**, which
produces fatigue, which produces missed alerts. The two frameworks worth knowing:

- **The four golden signals** (Google SRE): latency, traffic, errors, saturation. Note that
  *latency* means the distribution of successful requests separated from failed ones, because
  fast failures otherwise flatter your numbers.
- **USE** (Brendan Gregg) for resources: utilisation, saturation, errors. Applied per resource:
  CPU, memory, disk, network, connection pool, thread pool.

The organising discipline is **symptom-based alerting against an SLO**:

> Alert on things that are *user-visible and require human action*. Everything else is a
> dashboard or a ticket.

An **SLI** is a measured ratio (good events / valid events). An **SLO** is a target for it over a
window. The **error budget** is `1 − SLO` — the amount of failure you have explicitly agreed to
allow, and its power is organisational as much as technical: it converts "is reliability good
enough?" from an argument into a measurement, and it makes the decision to slow down and fix
things a *consequence* rather than a negotiation.

**Alert on burn rate, not on threshold crossings.** A 99.9% availability SLO over 30 days permits
about 43 minutes of errors. Alerting when the error rate exceeds some fixed number is either
noisy or slow. Alerting when the *budget burn rate* is high enough to exhaust the budget in a
short time — with a multi-window, multi-burn-rate policy (for example, a fast page at 14.4×
sustained over 5 minutes and 1 hour; a slower ticket at 6× over 6 hours) — gives you fast
detection of severe events and no pages for slow ones. This is the single most useful concrete
technique in this section; the SRE Workbook chapter 5 has the arithmetic.

**A percentile is not a number, it is a summary.** Averaging percentiles across instances or
across time is arithmetically meaningless. Aggregate with histograms (or t-digests), not with
pre-computed percentiles. And beware **coordinated omission** (PY-602 L01): if your measurement
harness stops sending requests while the system is stalled, the stall is invisible in your
latency data.

### 2.4 Debugging you cannot do

Some honesty about what does not transfer from single-process debugging. You cannot attach a
debugger to "the system". You cannot reproduce the state, because the state is spread across
machines and includes in-flight messages. You cannot step through, because stepping changes the
timing and the timing is the bug. And you cannot trust timestamps to establish ordering (L02) —
which is why causality must be *carried* (trace context, causal metadata) rather than
reconstructed.

What replaces it:

- **Correlation IDs everywhere**, propagated through every hop including queues and background
  jobs. Non-negotiable.
- **Structured events**, so that "find all requests where X" is a query rather than a regex.
- **Exemplars**: attach a trace ID to a metric bucket so that "the p99 got worse" leads directly
  to an example trace of a slow request rather than to a hunt.
- **Continuous profiling** in production (PY-602 L02) — sampled CPU and allocation profiles with
  the same labels as your traces, so you can go from a slow span to the actual stack.
- **Deterministic replay for the algorithmic core.** Your L05 Raft implementation and your L09
  resilience layer should be testable over a seeded, simulated network, so consensus and
  resilience bugs are reproducible even though production is not. FoundationDB's deterministic
  simulation is the reference example, and it is the reason they can claim the correctness they
  do.

### 2.5 Chaos engineering

> **Chaos engineering** is the discipline of experimenting on a system in order to build
> confidence in its capability to withstand turbulent conditions in production.

The word doing the work is **experimenting**. The method has a definite shape:

1. **Define steady state** as a measurable business or system metric — orders per minute,
   successful streams started. Not "CPU is fine".
2. **Hypothesise** that steady state continues under a specific injected condition. Write it down
   *before* running.
3. **Introduce real-world events**: instance termination, latency injection, packet loss,
   partition, dependency failure, clock skew, disk filling, region loss.
4. **Try to disprove the hypothesis** — the goal is to find the difference between the steady
   state you expected and the one you got.

The discipline around it:

- **Minimise the blast radius.** Start in a test environment, then production with a small
  percentage of traffic, with an abort button that works, during business hours with people
  watching. An experiment you cannot stop is not an experiment.
- **Every hypothesis is written before the run**, or you are not experimenting, you are breaking
  things and rationalising afterwards.
- **A failed experiment is a success.** It found a defect before a customer did. The output is a
  fix and a regression test, not a blameless shrug.
- **Automate it and run it continuously.** A one-off game day proves the system was resilient
  once; continuous experiments catch the regression introduced next quarter.

**Game days** are the human-side counterpart: exercise the *response*, not just the system. Most
incident cost is in detection and coordination, and those are the parts that are never tested
until they are needed. A game day tests whether the runbook is accurate, whether the on-call
engineer can find the dashboard, and whether the escalation path works at 3 a.m.

The prerequisite most teams skip: **do not run chaos experiments on a system you cannot observe.**
Without §2.1–2.3 in place, an experiment tells you only that something broke, which you could have
guessed. Observability first, chaos second.

## 3. Construction: instrumenting and then breaking your own system

Build in `mpse/ds701/l10/`, applied to the L05 Raft implementation and the L09 resilience layer,
so that you are instrumenting a real distributed system you wrote yourself.

**Stage 1 — wide events.** Define one structured event per unit of work, emitted once at
completion, carrying at minimum: `trace_id`, `span_id`, `parent_id`, service, operation, duration,
status, error class, and every dimension you have (node, term, role, retry count, queue wait).
Emit as JSON lines. Write a small query script that answers "p99 duration grouped by node and
role" from the raw events. Notice you did not have to decide in advance that you would want that
grouping — that is the property you are building.

**Stage 2 — OpenTelemetry tracing.** Instrument the Raft RPCs with real OTel spans and W3C
`traceparent` propagation. Run a Jaeger container and look at a real trace of an election.
Deliberately break propagation across one boundary and observe what a hole in a trace looks like,
so you recognise it later.

**Stage 3 — the clock skew artefact.** Inject 200 ms of skew on one node (you have this from L02)
and view the trace. Document the paradoxical rendering, then write the paragraph explaining why
parent-child structure, not timestamps, is what carries causality.

**Stage 4 — sampling.** Implement head-based sampling at 1%, then tail-based sampling that keeps
100% of errors and traces over the p99, plus 1% of the rest. Run a workload with a 0.5% error rate
and compare what each strategy retained. Then add weighted sampling and verify that your
aggregates over the sampled data match the aggregates over the full data within the expected error.

**Stage 5 — SLOs and burn-rate alerts.** Define an SLI for your system, set a 99.9% SLO over 30
days, compute the error budget, and implement multi-window multi-burn-rate alerting (14.4× over
5 m and 1 h; 6× over 6 h; 1× over 3 d). Then replay three incident shapes through it — a short
total outage, a long low-grade degradation, and a brief spike — and record which alerts fire and
how quickly. Compare against a naive "error rate > 1%" rule.

**Stage 6 — exemplars and profiling.** Attach trace IDs to your latency histogram buckets so a
slow bucket links to a real trace. Add a sampling profiler with the same labels, and demonstrate
going from "p99 regressed" to a stack in three steps.

**Stage 7 — the chaos harness.** Extend the L01 fault injector into an experiment runner: a
declarative experiment (steady-state metric, threshold, injected fault, duration, abort condition)
that runs, measures, and reports pass/fail — with the abort condition genuinely enforced.

**Stage 8 — the experiments.** Write and run at least six against your own cluster: kill the
leader; partition a minority; partition the leader into the minority; add 500 ms of latency on one
link; drop 10% of packets; skew a clock by 5 seconds. For each, write the hypothesis first. Then —
and this is the part that matters — **write up the one that surprised you**, including what you
had believed, what actually happened, and what you changed. If nothing surprised you, your
experiments were too gentle.

## 4. Failure modes

- **Dashboards mistaken for observability.** If you can only answer questions you predicted, you
  have monitoring.
- **Low-cardinality-only telemetry.** The dimension that identifies the problem — tenant, build,
  instance — is exactly the one metrics cannot hold.
- **Averaging percentiles.** Arithmetically meaningless; aggregate histograms instead.
- **Alerting on causes rather than symptoms.** Produces fatigue and misses the symptom nobody
  predicted a cause for.
- **Threshold alerts instead of burn rates.** Either noisy or slow; usually both, at different
  times of day.
- **Broken context propagation.** One library that drops headers silently amputates every trace
  downstream of it, and the traces still look plausible.
- **Head-based sampling only.** You throw away the errors and slow requests at the same rate as
  everything else — that is, you throw away precisely the data you needed.
- **Logging in the hot path, synchronously.** Observability that changes the performance it is
  observing. Buffer and emit asynchronously, and measure the overhead.
- **Chaos before observability.** You learn that something broke and nothing about why.
- **Chaos with no blast-radius control or abort.** That is an outage you scheduled.
- **Game days that only test the system.** Detection and coordination are usually the slow parts,
  and they are the parts that only a game day exercises.
- **PII in wide events.** High-cardinality dimensions are exactly where personal data leaks. Decide
  what may be recorded, and enforce it at the emission layer rather than by policy.

## 5. Exercises

### Warm-up (30 min)

1. Give a question that metrics cannot answer, one that logs answer badly, and one that only a
   trace answers. Say why in each case.
2. Explain cardinality and why it is the dividing line between monitoring and observability.
3. A 99.95% SLO over 28 days: compute the error budget in minutes, and the burn rate that would
   exhaust it in one hour.

### Core (3 h)

4. Complete Stages 1–3. Deliver the trace screenshot of an election and the clock-skew write-up.
5. Complete Stage 4 and deliver the comparison table of what each sampling strategy retained.
6. Complete Stage 5 and deliver the table of which alerts fired for which incident shape, with a
   paragraph on why the naive rule is worse.
7. Write an **observability review** of a service you work with: list the questions you could not
   answer during a real incident, and specify the exact instrumentation change that would have
   answered each. Be concrete — name the dimension.

### Challenge

8. Complete Stages 7–8 with all six experiments and the surprise write-up.
9. Build a **deterministic simulation harness** for your Raft implementation: a seeded scheduler
   controlling message delivery order, delays, drops and node crashes, such that a given seed
   reproduces a run exactly. Run it for thousands of seeds with a linearizability checker
   (Jepsen's Knossos or Porcupine, or your own from L04) asserting the invariant. Report any seed
   that fails, and show that re-running that seed reproduces the failure exactly. This exercise is
   the closest thing in the course to how correctness is actually established in industrial
   distributed systems.

## 6. Self-check

1. State the difference between monitoring and observability in one sentence.
2. Why is one wide event per unit of work better than fifteen log lines?
3. What is context propagation, and what is the most common way it breaks?
4. Compare head-based and tail-based sampling; what does weighted sampling add?
5. Why can a child span appear to start before its parent?
6. Name the four golden signals and the USE method's three.
7. Define SLI, SLO and error budget, and explain what the error budget is *for* organisationally.
8. Why alert on burn rate, and what does multi-window buy you?
9. Give the four steps of a chaos experiment.
10. Why must observability precede chaos engineering?

## 7. Primary sources

- **Sigelman et al., "Dapper, a Large-Scale Distributed Systems Tracing Infrastructure"
  (Google, 2010)** — the origin of the span model.
- **Basiri et al., "Chaos Engineering" (IEEE Software, 2016)** and the *Principles of Chaos
  Engineering* statement — the experimental method, stated properly.
- Google, *The Site Reliability Engineering Workbook*, chapters 2 and 5 — SLOs and, in particular,
  multi-window multi-burn-rate alerting.
- Google, *Site Reliability Engineering*, chapter 6 ("Monitoring Distributed Systems").
- Majors, Fong-Jones & Miranda, *Observability Engineering* — the wide-event, high-cardinality
  argument in full.
- The OpenTelemetry specification, and the W3C Trace Context recommendation.
- Zhao et al., "lprof" / Beschastnikh et al., "Debugging Distributed Systems" (CACM 2016) — what
  automated analysis of distributed logs can and cannot do.
- Kingsbury, the Jepsen reports — read three; they are the best available demonstration of how
  distributed systems actually fail, and of the standard of evidence you should hold yourself to.
- Zhou et al., "FoundationDB: A Distributed Unbundled Transactional Key Value Store" (SIGMOD 2021)
  — the deterministic simulation testing section.

---

**Previous:** [L09](L09-failure-detection-and-resilience.md) ·
**Next:** [Problem sets](problem-sets.md) · [Exam](exam.md)

*This completes the DS-701 lesson sequence. Before the problem sets, re-read the L01 orientation:
the whole course is an elaboration of the claim that partial failure is the defining
characteristic of a distributed system, and every lesson since has been either a way to tolerate
it, a proof about the limits of tolerating it, or a way to see it happening.*
