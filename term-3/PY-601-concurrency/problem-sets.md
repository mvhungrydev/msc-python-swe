# PY-601 — Problem Sets

**A note on evidence.** A concurrency claim requires a schedule, not an assurance. "This is
thread-safe" is a hope; "here is the invariant, here is the window in which it is violated, here is
the lock that covers exactly that window, and here is the stress test that fails when I narrow it"
is an answer.

**A note on the constructions.** Each set builds on the lessons' §3 constructions. Where a part
names lesson stages, do those first — several later parts assume the harness they produce.

**A note on what testing establishes.** Every part that reports a passing stress test must also
state what the pass does *not* establish. A run that found no violation is evidence proportional to
the schedules it explored, and saying so precisely is the assessed skill.

---

## Problem Set 1 — A Thread-Safe Component with a Correctness Argument
**Covers L01–L04 · Budget: 14–18 hours**
*Builds on: L01 §3 stages 1–5, L02 §3 (all stages), L03 §3 (all stages), L04 §3 stages 1–7*

**Part A — The model comparison (L01 §3).** One workload, five implementations
(sequential, threads, asyncio, processes, hybrid), measured across a range of per-item CPU
cost. Plots for wall clock and CPU time. The crossover point, explained. The serial fraction
inferred from the measured speedup curve, compared with your prediction from reading the
code. The USL curve for the process version, with the turn-down point identified and its
physical cause named.

**Part B — The GIL experiments (L02 §3).** All five, including the switch-interval latency
curve (experiment 3) — which almost nobody runs and which explains a class of production
tail-latency mystery. Include experiment 5 on a free-threaded build if you can get one, or
via Docker.

**Part C — The cache (L03 §3).** All four versions, the stampede test (50 threads, one cold
key, assert exactly one computation), the remaining-defects list from version 3 with a fix
for each, and throughput and p99 latency for all four at a realistic hit ratio.

**Part D — The metrics collector (L04 §3).** All four versions with measurements, and a
precise statement of the consistency your chosen version provides plus one way a caller could
misuse it.

**Part E — The correctness argument.** For your chosen cache implementation, write the
argument that it is correct: state the invariants, classify each as safety or liveness, say
what protects each, and identify the interleaving points. Then state the correctness
condition you are claiming (linearizable? conditionally thread-safe? — L01 §2.4, L04 §2.1)
and defend it.

**Design note (1,200–1,600 words).** Which measurement most changed your mental model. The
serial fraction discrepancy in Part A and where it actually came from. Where you moved up
the design hierarchy (L04 §2.7) and what it cost.

### Marking emphasis

The correctness argument. The implementation is the easy part. The marks are in stating the
invariant, identifying the window in which it is violated, and showing the synchronisation covers
exactly that window — no more and no less.

---

## Problem Set 2 — An Event Loop, Then a Service with Correct Cancellation
**Covers L05–L07 · Budget: 16–20 hours**
*Builds on: L05 §3 (all stages), L06 §3 stages 1–7, L07 §3 (all stages)*

**Part A — Build the event loop (L05 §3).** Stages 1–6: cooperative scheduling, timers,
real I/O with `selectors`, tasks and futures, cancellation, and the slow-callback
diagnostic. An echo server tested with 100 concurrent clients. Roughly 150–250 lines.

**Part B — Benchmark it.** Against `asyncio` and `asyncio` + `uvloop`, at 10/100/1000
concurrent connections. Profile the gap and attribute it to specific causes.

**Part C — The service (L06 §3).** Both the naive and the structured version. The four
demonstrated problems from stage 2 *with measurements*: the loop stalled by a blocking call,
`gather` leaving siblings running, the lost fire-and-forget exception, and in-flight work
killed at shutdown. Then the `TaskGroup` version, bulkheads, and the latency comparison
under a degraded upstream — that comparison is the entire argument for the structure.

**Part D — Cancellation safety (L07 §3).** The worker, all six stages: the
interruption-point table, the correctness argument (redelivery + idempotence), graceful
shutdown, the five hostile tests, and deadline propagation.

**Part E — TaskGroup semantics (L06 C2).** Tests establishing the observed behaviour for
six cases, and the comparison with `anyio`'s task group.

**Design note (1,500–2,000 words).** What building the loop changed about how you read
`asyncio` code. The interruption-point table and why the answer turned out to be idempotence
rather than shutdown code. Your position on whether `asyncio` should deprecate bare
`create_task`.

### Marking emphasis

Cancellation. Most submissions handle the happy path and the timeout. The assessed cases are
cancellation during cleanup, cancellation of a task that is itself cancelling, and shutdown with
work in flight.

---

## Problem Set 3 — A Parallel Pipeline with Flow Control, Tested Adversarially
**Covers L08–L10 · Budget: 14–18 hours**
*Builds on: L08 §3 (all stages), L09 §3 stages 1–7, L10 §3 (all stages)*

**Part A — The parallel pipeline (L08 §3).** All seven stages. The per-change measurement
table from stage 3 (this is the assessed centrepiece — it shows where the time actually
went), the shared-memory result, the scalability curve with the turn-down explained, the
crash-recovery implementation, and the seven-way comparison including the vectorized
sequential version.

**Part B — Flow control (L09 §3).** All seven stages. The failure-mode plots for the
unbounded and bounded versions, the goodput comparison with and without deadline checking,
the shedding policy with metrics, the utilization/latency curve with a defended capacity
target, and rate-limiter verification under burst and 429s.

**Part C — Testing (L10 §3).** All seven levels on one component: stress at two switch
intervals, deterministic simulation with 10,000 seeds and a reproducible failing seed, the
property assertions, a linearizability checker with a demonstrated catch, fault injection
including cancellation at every `await`, a model check in PlusCal, and the level-7 table
filled in for your component.

**Part D — Production readiness (L10 C3).** Named tasks, `faulthandler` on a signal, a task
dump, `slow_callback_duration` logging, and a hang-diagnosis runbook. Then the blind
diagnosis exercise, timed.

**Design note (1,500–2,000 words).** Where the parallel pipeline's time actually went versus
where you expected. What deadline checking did to goodput under overload. What each testing
level established and — more importantly — what it did not. The bug that only the
deterministic simulator found.

### Marking emphasis

What the testing does not establish. Every passing stress test must be accompanied by a statement
of the schedules it did not explore. A confident report of a clean run scores below a hedged one.

---

## Term 3 build artifact (shared with PY-602)

A high-throughput service:

- An async I/O-bound front end with structured concurrency and correct cancellation.
- A parallel CPU-bound stage (processes or a GIL-releasing extension).
- Bounded queues with backpressure propagating end to end, and a deliberate shedding policy.
- Graceful shutdown with no lost or duplicated effects, verified by test.
- A deterministic simulation harness for the concurrent logic.
- Production instrumentation: named tasks, stack dumps on signal, slow-callback logging,
  drop and rejection metrics.
- A measured before/after performance report with full methodology (PY-602 L01) — machine,
  OS, Python build, medians and spread, and the statistical treatment.

---

## Course position paper (1,500 words)

**Driving question: what does it mean for a concurrent program to be correct?**

Claim, grounds, rebuttal, limits. Grounds from your own work: the correctness argument in
PS1 Part E, the interruption-point analysis in PS2 Part D, and the level-7 table in PS3
Part C. The rebuttal must state fairly the position that formal correctness conditions are
largely irrelevant to practice — that real systems are made reliable by idempotence,
retries, monitoring, and restarts rather than by linearizability arguments — and then answer
it.

---

## Submission checklist (per set)

- [ ] Code in `courses/py601/psN/`, runnable from a clean clone.
- [ ] Every measurement reproducible: the script and `scripts/envinfo.py` output committed.
- [ ] Medians and spread, never a single run. Machine, OS, Python build recorded.
- [ ] Concurrency tests marked so they can be run 1,000× in a nightly job.
- [ ] `NOTE.md` — the design note.
- [ ] `MARK.md` — self-marked, one line of justification per criterion.
- [ ] `log/failures.md` updated — this course should produce a long entry.
