# CAP-799 — Project Catalogue

Twenty capstone projects, each with the four things a proposal needs: **the question**, **the
minimum system** that could answer it, **the evaluation**, and **the principal risk**.

These are starting points. A project arising from a problem you actually have at work is better than
any of them, because you will care about the answer and you will have a real workload to measure
against. Use these as templates for the *shape* of a suitable question.

Each is tagged with the courses it draws on, and marked for difficulty: ◆ substantial, ◆◆ hard,
◆◆◆ ambitious (consider narrowing).

---

## Storage and data systems

### 1. The compaction strategy crossover ◆
*DI-721, PY-602*

**Question.** For a given read/write mix and key distribution, where exactly is the crossover
between levelled and size-tiered compaction, and do the published rules of thumb hold?

**Minimum system.** Your DI-721 L02 LSM with both strategies, instrumented for all three
amplifications, plus a workload generator with configurable read/write ratio and Zipfian skew.

**Evaluation.** A grid over read ratio, skew parameter and dataset-to-memory ratio. Report the
crossover surface. Validate two points against RocksDB configured equivalently.

**Risk.** Your engine's constant factors differ so much from RocksDB's that the crossover does not
transfer. Mitigate by reporting *ratios* and by validating the two points.

### 2. Bloom filter allocation across levels ◆◆
*DI-721, CS-621*

**Question.** Does Monkey's optimal bloom filter allocation (more bits at shallower levels) deliver
its predicted improvement on a real workload, and how sensitive is the gain to workload assumptions?

**Minimum system.** Your LSM with per-level configurable bloom bits-per-key; the Monkey allocation
implemented alongside the uniform one.

**Evaluation.** Measure read amplification and memory for both, across workloads that satisfy and
violate the paper's assumptions. Report where the theory holds and where it does not.

**Risk.** The improvement is real but small relative to your measurement noise. Establish the noise
floor in week 2.

### 3. Point-in-time correctness at scale ◆◆
*DI-721, ML-741*

**Question.** What does point-in-time-correct feature computation actually cost, in engineering
complexity and in compute, versus the naive approach — and how large is the leakage it prevents on
realistic data?

**Minimum system.** Two feature pipelines over the same event log: naive and as-of-join, plus a
model trained on each.

**Evaluation.** The offline/online metric gap for both. Compute cost per training set. Then vary
the degree of temporal correlation in the synthetic data and report how the leakage scales.

**Risk.** Synthetic data makes leakage trivially large or trivially small. Use a real dataset with a
genuine temporal structure if you can obtain one.

---

## Distributed systems

### 4. Consensus versus CRDTs, measured ◆◆
*DS-701, FM-751*

**Question.** For a shopping-cart-style workload with a real invariant, what does the CRDT design
actually buy in availability, and what does it cost in anomalies, compared with a Raft-replicated
design under an identical fault schedule?

**Minimum system.** Both implementations behind one interface, plus the DS-701 fault injector and a
linearizability checker.

**Evaluation.** Availability, latency and anomaly count under a fixed fault schedule; the invariant
violation rate for the CRDT design; and the escrow variant's numbers. Specify the invariant in TLA+
and check that the escrow scheme preserves it.

**Risk.** The comparison is unfair because one implementation is more mature. Mitigate by building
both from the same primitives and reporting engineering effort for each.

### 5. Deterministic simulation testing ◆◆◆
*DS-701, FM-751*

**Question.** How much of FoundationDB-style deterministic simulation testing can a small team build,
and what class of bug does it find that randomised fault injection does not?

**Minimum system.** A seeded scheduler controlling all non-determinism — message order, delays,
drops, crashes, timers — for your Raft implementation, such that a seed reproduces a run exactly.

**Evaluation.** Run thousands of seeds with a linearizability checker. Compare bug-finding rate
against unseeded chaos testing at equal wall-clock. Categorise the bugs each found.

**Risk.** Achieving true determinism is harder than it looks; budget for it and scope down the
system if necessary. This is the project's real content.

### 6. The metastable failure boundary ◆◆
*DS-701, CA-731, PY-601*

**Question.** For a realistic service, where exactly is the boundary between a load spike that the
system recovers from and one that produces a self-sustaining metastable failure — and which
mitigation moves it furthest?

**Minimum system.** A service with a configurable dependency, retry logic, and pluggable mitigations
(jitter, retry budget, circuit breaker, concurrency limit, load shedding).

**Evaluation.** Map the boundary in (spike magnitude, spike duration) space for each mitigation
configuration. Report which mitigation moves the boundary most per unit of complexity.

**Risk.** The boundary is sharp and noisy. Use many trials per point and report confidence.

### 7. Trace validation in production shape ◆◆
*FM-751, DS-701*

**Question.** What does it cost to keep a TLA+ specification connected to a running implementation
via trace validation, and what fraction of injected implementation bugs does it catch?

**Minimum system.** Your L06 specification, instrumentation emitting state traces, and a validator.

**Evaluation.** Inject twenty implementation bugs of varying subtlety. Report the catch rate,
compared against property-based testing derived from the same specification. Report instrumentation
overhead and the false-divergence rate from specification/implementation abstraction differences.

**Risk.** The abstraction gap produces so many false divergences that the technique is unusable.
That is itself a publishable finding — report it rigorously.

---

## Platform and cloud

### 8. The price of a nine, measured ◆◆
*CA-731, DS-701*

**Question.** For a concrete service, what does each additional nine of availability actually cost,
and does the theoretical availability arithmetic predict the measured availability under realistic
failure injection?

**Minimum system.** One service deployed in four configurations of increasing redundancy, with a
failure injector modelling realistic correlated failures.

**Evaluation.** Measured availability per configuration against the arithmetic's prediction; cost
per configuration; and an analysis of where and why the arithmetic overstates.

**Risk.** Achieving statistically meaningful availability measurements requires long runs. Use
accelerated failure rates and state the extrapolation assumption.

### 9. Shuffle sharding, empirically ◆
*CA-731*

**Question.** Does shuffle sharding's combinatorial promise hold under realistic tenant behaviour,
including correlated tenant activity and client-side retries?

**Minimum system.** A multi-tenant service with configurable sharding (simple, shuffle, with and
without retry), and a tenant workload generator with configurable correlation.

**Evaluation.** Fraction of tenants affected by one bad tenant, measured against the analytical
prediction, across correlation levels and shard parameters. Identify where theory and measurement
diverge and why.

**Risk.** None serious; this is a well-scoped ◆ project and a good choice if time is tight.

### 10. Static stability, costed ◆◆
*CA-731*

**Question.** At what utilisation does static stability stop being worth its cost, given realistic
control-plane failure probabilities and correlated recovery demand?

**Minimum system.** A simulator of a multi-AZ service with autoscaling, a modelled control plane
with configurable latency and failure probability, and a contended capacity pool.

**Evaluation.** Availability and cost for statically stable versus autoscaled designs across
utilisation levels and control-plane reliability. Identify the crossover surface.

**Risk.** The result depends heavily on the control-plane model; ground it in published incident
data and report the sensitivity.

### 11. An SMT-based platform policy analyser ◆◆
*FM-751, CA-731*

**Question.** Can a small team build a sound blast-radius analyser for a real IAM or RBAC
configuration, and what does it find that graph-based analysis misses?

**Minimum system.** An SMT encoding of a real policy configuration including conditions, boundaries
and role-assumption chains, plus the graph-based analyser for comparison.

**Evaluation.** Findings from each on a real configuration; the soundness argument for the encoding;
and a deliberate injection of ten escalation paths with the detection rate for each technique.

**Risk.** Encoding fidelity. The soundness section is the hard and valuable part — budget for it.

---

## Language and runtime

### 12. Free-threaded Python, measured ◆◆
*PY-601, PY-602*

**Question.** For which workload shapes does free-threaded CPython actually outperform
multiprocessing and the GIL-bound baseline, and where is the crossover?

**Minimum system.** A benchmark suite spanning CPU-bound, memory-bound, I/O-bound and mixed
workloads, with shared-state contention as a parameter.

**Evaluation.** Throughput and latency across all three execution models, at varying core counts and
contention levels. Report the crossover surface and the per-object-locking overhead.

**Risk.** Build availability and version churn. Pin a build, record it, and note that the result is
dated.

### 13. Where does the abstraction cost go? ◆
*PY-501, PY-502, PY-602*

**Question.** For each of Python's abstraction mechanisms — descriptors, properties, `__getattr__`,
dataclasses, ABCs, protocols — what is the measured runtime cost, and how much of it does the
specialising interpreter recover?

**Minimum system.** A micro-benchmark suite with equivalent implementations at each abstraction
level, plus bytecode and specialisation inspection.

**Evaluation.** Cost per mechanism, across Python versions, with the specialisation behaviour
explained. Then a realistic composite benchmark showing whether the micro-benchmark differences
survive.

**Risk.** Micro-benchmarks that measure nothing real. The composite benchmark is what makes this a
capstone rather than an exercise.

### 14. A gradual type system for a real codebase ◆◆◆
*CS-641, SE-511*

**Question.** What fraction of a real untyped codebase's bugs would a gradual type system have
caught, and what is the annotation effort per bug caught?

**Minimum system.** A real codebase with a git history containing fixed bugs; a typing effort applied
to it; and a classification of each historical bug as type-preventable or not.

**Evaluation.** Bugs preventable per hour of annotation. Then the more interesting question: which
*kinds* of bug are type-preventable and which are not, with the taxonomy.

**Risk.** Selection bias in which bugs are in the history. State it as a threat to validity.

---

## Machine learning systems

### 15. The offline/online gap, quantified ◆◆
*ML-741*

**Question.** For a recommendation or ranking task, how well does offline metric improvement predict
online improvement, and does the correlation depend on which offline metric is used?

**Minimum system.** A simulator with known ground truth (ML-741 L08 Stage 1), several models, and
both offline and online evaluation.

**Evaluation.** Correlation between offline improvement and true online improvement, across several
offline metrics and several degrees of feedback-loop strength. Report where the correlation breaks
down.

**Risk.** The simulator's structure determines the answer. Vary the simulator's assumptions and
report the sensitivity — that variation is the finding.

### 16. Exploration's real cost ◆◆
*ML-741*

**Question.** For a system with a feedback loop, what exploration rate maximises long-run true
performance, and how much long-run performance does a team lose by choosing the rate that maximises
short-run metrics?

**Minimum system.** The ML-741 L08 loop simulator with configurable exploration and a known ground
truth.

**Evaluation.** Long-run true satisfaction against exploration rate, for several loop strengths and
time horizons. Report the gap between the myopically-optimal and long-run-optimal rates.

**Risk.** None serious; well-scoped, and the result is directly useful.

### 17. LLM system evaluation methodology ◆◆
*ML-741, FM-751*

**Question.** For a retrieval-augmented question-answering system, how well does LLM-as-judge agree
with human judgement, and what rubric properties predict agreement?

**Minimum system.** A RAG system, a golden set of at least 200 items, a judge with several rubric
variants, and human grading of a substantial sample.

**Evaluation.** Judge-human agreement per rubric variant, with the biases measured. Then the useful
part: what distinguishes rubrics with high agreement from those with low, stated as guidance.

**Risk.** Human grading is expensive and slow. Budget half your evaluation time for it, and recruit
a second grader for inter-annotator agreement.

---

## Verification

### 18. Specifying something that has never been specified ◆◆
*FM-751*

**Question.** Does the design of [a protocol from a recent paper, an internal system, or an RFC
draft] satisfy the properties its authors claim?

**Minimum system.** A faithful TLA+ specification, the properties as stated by the authors, and a
model check.

**Evaluation.** The check result, with bounds. Then, whatever the outcome: the list of ambiguities
in the source document that you had to resolve to specify it, and how you resolved each.

**Risk.** The protocol is correct and you find nothing. The ambiguity list is then the deliverable —
and it usually has more entries than the authors would like. This is how Zave's Chord work started.

### 19. Property-based testing versus model checking, measured ◆◆
*FM-751, SE-511*

**Question.** For a class of concurrent data structures, which bugs does model checking find that
property-based testing does not, and vice versa — and what is the cost per bug for each?

**Minimum system.** Three concurrent structures, each with a TLA+ specification and a property-based
test suite, plus a corpus of twenty injected bugs of varying subtlety.

**Evaluation.** Detection rate per technique per bug class; time-to-detection; engineering hours to
build each apparatus. Then the categorisation: what kind of bug does each technique structurally
miss?

**Risk.** The injected bug corpus is unrepresentative. Draw the bugs from real CVEs and issue
trackers where possible, and say where you did not.

### 20. Verified configuration for a real platform ◆◆◆
*FM-751, CA-731*

**Question.** Can the safety-relevant configuration of a real platform — network reachability, IAM,
Kubernetes admission policy — be verified end to end, and what does the verification miss?

**Minimum system.** SMT encodings of all three layers, composed, with a set of end-to-end properties
("nothing on the internet can reach the database, through any path, under any policy").

**Evaluation.** Properties verified with the assumptions stated; injected misconfigurations with the
detection rate; and the unsoundness/incompleteness analysis — which is the project's real
contribution.

**Risk.** Ambitious. Narrow to two layers if week 6 arrives without a working composition.

---

## Choosing

If none of these fits, the test for your own idea is §2 of the handbook: a question rather than a
system, difficulty in the right place, something measurable, achievable in 200 hours, and defensible
for an hour.

And the practical advice: **choose the one you would still be interested in at week 9**, when the
building is done, the first results are ambiguous, and the writing has not started. That is the week
that determines whether a capstone finishes, and interest is the only thing that gets you through
it.
