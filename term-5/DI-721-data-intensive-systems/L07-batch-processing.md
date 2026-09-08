# DI-721 · Lesson 07 — Batch Processing and Dataflow

**Estimated study time:** 4 hours
**Prerequisites:** L01, L06; CS-621 L02 (divide and conquer), DS-701 L01

---

## 1. Orientation

Batch processing is the oldest idea in data engineering and the one whose lessons are most often
forgotten and rediscovered. Its defining shape:

> **Read a bounded input, compute, write a new output. Do not modify the input. If it fails, run
> it again.**

That last sentence is the whole of the reliability model, and it is worth appreciating how much it
buys. Because the input is immutable and the output is written fresh, a failed job is retried by
*re-running it* — no partial state to reconcile, no compensation logic, no idempotency keys. The
functional-programming property of "no side effects on the input" is what makes fault tolerance
tractable at scale, and it is the reason batch systems were reliable long before streaming systems
were.

This lesson covers the model, the two operations that dominate it (sort and join), the failure
modes that actually happen (skew, above all), and the modern shape of the argument — because
"batch versus streaming" is largely resolved, and the resolution is more interesting than either
camp's original position.

## 2. Theory

### 2.1 The Unix heritage

`cat log | grep ERROR | awk '{print $7}' | sort | uniq -c | sort -rn | head`

Kleppmann's observation is that this pipeline exhibits every property of a modern batch system:
uniform interface (a stream of bytes), composition without coordination, immutable inputs,
separation of logic from wiring, and trivial retry. MapReduce is this pipeline distributed across
a cluster, with the sort made explicit and the intermediate results written to a distributed
filesystem instead of a pipe.

What MapReduce added was not the model but the *fault tolerance*: with thousands of commodity
machines, a job that must restart entirely on any failure will never finish. Materialising
intermediate output lets a failed task be re-run alone. What it cost was performance —
materialising everything to disk between stages is enormously wasteful when nothing fails, which
is exactly what Spark and its successors fixed with lineage-based recovery over in-memory data.

The through-line: **fault tolerance strategy is the main axis on which these systems differ, and
it is a trade against performance in the no-failure case.**

### 2.2 MapReduce, and what the shuffle really is

- **Map**: for each input record, emit zero or more key-value pairs. Embarrassingly parallel, one
  task per input split, scheduled near the data.
- **Shuffle**: partition by key, sort within partition, transfer to reducers. **This is the whole
  job.** Map and reduce are user code; the shuffle is the system, and it is where the time,
  network, disk and failures are.
- **Reduce**: for each key, process the sorted group of values.

Why sorting is central: it is how "all values for one key together" is achieved on data far larger
than memory. External merge sort (CS-621 L02's merge, plus L01's external memory model) does it in
O((n/B) log_{M/B}(n/B)) I/Os, and the sorted order also gives sort-merge joins and grouped
aggregation for free.

Two optimisations worth naming because they recur in every system since:

- **Combiners** — a partial reduce on the map side, valid when the reduce function is commutative
  and associative (that is, when it is a monoid; the same algebraic condition as DS-701 L07's CRDT
  merge, which is not a coincidence). A `SUM` combines; a `MEDIAN` does not. This distinction —
  which aggregates are algebraic and which are holistic — determines what can be pushed down, and
  it will return in L08's windowing.
- **Map-side joins** — if one input is small enough to broadcast, no shuffle is needed at all. The
  single largest available win in practice.

### 2.3 Joins, and how they fail

Four strategies, and the ability to reason about which one a planner chose is the core skill:

| Strategy | Mechanism | Cost | When |
|---|---|---|---|
| **Broadcast (map-side)** | ship the small side to every task | no shuffle of the large side | one side fits in memory |
| **Shuffle hash** | partition both by key; build a hash table per partition | shuffle both sides | both large, one partition fits in memory |
| **Sort-merge** | sort both by key; merge | shuffle and sort both | both very large, or already sorted |
| **Nested loop** | for each row, scan the other | O(n·m) | last resort; non-equi joins |

**Data skew is the dominant failure mode of distributed joins**, and it deserves its own treatment
because it is what actually goes wrong at 3 a.m.

If one key holds 40% of the rows — a "null" customer ID, a bot account, a default value — the task
that owns that key does 40% of the work. The job's runtime is the maximum over tasks, not the
average, so it is now ~400× the mean task time. The symptom is unmistakable: 999 tasks finish in
two minutes and one runs for six hours. Nothing is broken; the work is simply not divisible the
way the partitioner assumed.

The remedies:

1. **Filter the skewed key** if it is garbage (nulls, sentinels, a test account). Often it is, and
   this is a five-minute fix.
2. **Broadcast the other side** to eliminate the shuffle entirely, if it fits.
3. **Salting**: append a random suffix (1..N) to the skewed key on the large side and replicate the
   small side N times. Spreads one key over N tasks at the cost of N× replication of the small
   side. The standard general answer.
4. **Two-phase aggregation**: aggregate with a salted key, then aggregate the partial results. For
   algebraic aggregates only — the monoid condition again.
5. **Adaptive execution**: modern engines (Spark AQE) detect skewed partitions at runtime from
   shuffle statistics and split them automatically. Use it, and still know what it is doing,
   because it only fires when the statistics reveal the skew.

The general lesson generalises beyond joins: **a partitioning scheme is a bet that the key
distribution is roughly uniform, and real distributions are Zipfian.** DS-701 L06's hot-partition
problem is the same statement about a different system.

### 2.4 Dataflow beyond MapReduce

Spark, Flink, Tez and their kin generalise the model to an arbitrary **DAG of operators** rather
than a fixed map-shuffle-reduce. What that changes:

- **Multi-stage jobs without materialising every intermediate.** A chain of maps stays in memory.
- **Pipelined execution**, so downstream work starts before upstream finishes where possible.
- **Lineage-based fault tolerance**: instead of materialising output for recovery, record how each
  partition was *derived*, and recompute a lost one from its inputs. This is the central idea of
  RDDs, and the trade is explicit — cheap when nothing fails, potentially expensive recomputation
  when something does, which is why checkpointing exists as a middle ground.
- **Query optimisation over the DAG**: predicate pushdown, projection pushdown, join reordering,
  adaptive re-planning from runtime statistics. The database ideas of L01 §2.4 arriving in the
  batch world about thirty years later.

The convergence worth noticing: batch engines acquired optimisers and columnar storage from
databases; databases acquired distributed execution from batch engines. The distinction is now
largely about interface and workload rather than about mechanism.

### 2.5 The output side

The rule that makes batch reliable is that **output is written to a new location and swapped in
atomically**. Concretely: write to a temporary path, then rename or update a metadata pointer.
Never mutate the previous output in place.

That gives: retryability (a partial output is discarded, not merged), atomicity from the consumer's
point of view, and a trivially available rollback — the previous output still exists.

Two complications that bite in practice:

- **Object storage has no atomic rename.** S3's "rename" is copy-then-delete, and is neither atomic
  nor cheap. This is precisely why table formats (L06 §2.5) exist: the atomic operation becomes a
  compare-and-swap on a small metadata pointer rather than a directory rename.
- **Downstream consumers must observe the swap, not the intermediate.** Publish a manifest or a
  success marker, and have consumers read the pointer rather than listing a directory.

### 2.6 Batch versus streaming, resolved

The historical position — batch for correctness and throughput, streaming for latency, with a
**Lambda architecture** running both and reconciling — is now mostly obsolete, and it is worth
knowing why so you do not build it.

The Lambda architecture's problems were structural: the same logic implemented twice in two
systems, which then drift; reconciliation logic that is itself complex and untested; and double
the operational surface. The **Kappa** alternative — one streaming system, with reprocessing done
by replaying the log from the beginning — became viable once stream processors gained exactly-once
state handling, event-time correctness and the ability to replay (L08, L09).

The modern synthesis is that **batch is a special case of streaming: a bounded stream.** Flink and
Beam make this literal — the same program runs over a bounded or unbounded source with the same
semantics — and the practical consequence is that you choose based on latency requirements and
operational preference rather than on correctness.

What still favours batch: reprocessing large historical datasets efficiently; workloads where the
computation genuinely requires the whole dataset (training a model, a global sort); simplicity,
which is a real engineering value; and cost, since batch can use cheap preemptible capacity in a
way a low-latency stream cannot.

What still favours streaming: latency requirements measured in seconds; unbounded sources where
"the whole dataset" is not a meaningful concept; and incremental computation where recomputing
from scratch is wasteful.

## 3. Construction: a batch framework, and the failures it teaches

Build in `mpse/di721/l07/`. Use multiprocessing on one machine — the distribution is simulated but
every failure mode below is real and reproducible locally.

**Stage 1 — MapReduce in miniature.** A framework with pluggable map and reduce functions, input
splitting, a shuffle that partitions by hash and sorts, and a reduce phase. Run word count on a
few GB of text. Instrument every phase.

**Stage 2 — where the time goes.** Break the runtime down by phase: map, shuffle-write, transfer,
sort, shuffle-read, reduce. Produce the chart. The shuffle will dominate; that is the lesson, and
having measured it yourself is different from having read it.

**Stage 3 — combiners.** Add combiner support and re-run word count. Measure the reduction in
shuffled bytes and in runtime. Then implement an aggregate that is *not* combinable (an exact
median) and one that is approximately combinable (a t-digest or your HLL from CS-621 L07) and
compare all three. Write the paragraph on the algebraic condition and what it costs to give it up.

**Stage 4 — external sort.** Implement a proper external merge sort for the shuffle, with a bounded
memory budget and k-way merging. Measure I/O volume against the theoretical bound from the external
memory model, and vary the memory budget to show the effect on the number of merge passes.

**Stage 5 — joins.** Implement broadcast, shuffle hash and sort-merge joins. Benchmark all three
across relative table sizes from 1:1 to 1:10,000 and produce the crossover chart. Then write the
rule you would give a planner.

**Stage 6 — skew.** Generate a Zipfian key distribution and run a shuffle join. Plot per-task
runtime and observe the straggler. Then implement salting and re-measure; then implement adaptive
splitting (detect an oversized partition from the shuffle statistics and split it) and re-measure.
Report all three, and state the cost salting imposed on the small side.

**Stage 7 — fault tolerance.** Add task-level retry with materialised intermediates. Kill a random
task mid-run and show recovery. Then implement lineage-based recovery instead, kill a task, and
compare recovery time and no-failure runtime for the two strategies. This is the RDD trade-off,
measured by you, and it is the most instructive single experiment in the lesson.

**Stage 8 — atomic output.** Implement the write-temp-then-swap protocol with a manifest, and a
consumer that reads via the pointer. Kill the job during the write and show that the consumer never
observes a partial output. Then implement it against a local object-storage emulator (MinIO) and
confront the missing atomic rename directly — solve it with a metadata pointer, which is a
miniature of what Iceberg does.

## 4. Failure modes

- **Skew, unrecognised.** The job "hangs" at 99%. It is not hung; one task has 40% of the data.
- **Mutating the input, or the previous output.** Destroys retryability, which was the entire
  reliability model.
- **Non-deterministic map functions.** A retried task produces different output, so recomputation
  is no longer safe. Random numbers without a seed, `now()`, and iteration over an unordered
  collection are the usual culprits.
- **Non-idempotent side effects in a task.** Sending an email from a map function means sending it
  again on retry (DS-701 L08).
- **Small files.** A job producing a hundred thousand tiny output files creates a problem for every
  downstream consumer (L06 §2.5). Control the output partition count.
- **Ignoring the shuffle.** Optimising map-side code when 80% of the time is in the shuffle.
- **A join with no size estimate.** The planner picks sort-merge where broadcast would have been
  100× faster, because it does not know one side is 3 MB.
- **`GROUP BY` on a high-cardinality key with a holistic aggregate.** No combiner is possible and
  the shuffle carries everything.
- **Lambda architecture by default.** Two implementations of one logic; they will drift, and the
  reconciliation will be where the bugs live.

## 5. Exercises

### Warm-up (30 min)

1. Explain why immutable input plus fresh output makes batch fault tolerance simple, and name three
   things that break the property.
2. Give the four join strategies with their costs and the condition that selects each.
3. Explain the algebraic condition a combiner requires, classify `SUM`, `COUNT`, `AVG`, `MAX`,
   `MEDIAN` and `COUNT DISTINCT` by it, and say what you do about the ones that fail.

### Core (3 h)

4. Complete Stages 1–3 and deliver the phase-breakdown chart and the three-aggregate comparison.
5. Complete Stages 4–5 and deliver the join crossover chart with your planner rule.
6. Complete Stage 6 and deliver all three skew measurements with the salting cost stated.
7. Take a batch job you know (or one from Stage 5). Identify its shuffle volume, its join
   strategies and any skew, and propose one change with a predicted effect. Then measure.

### Challenge

8. Complete Stages 7–8, including the materialisation-versus-lineage comparison and the
   object-storage atomic-swap implementation.
9. Implement a **cost-based optimiser** for your framework: collect statistics (row counts,
   distinct values, size), and for a multi-join query enumerate plans and choose by estimated cost,
   including the broadcast/shuffle/sort-merge decision and join order. Compare its choices against
   exhaustive search on a five-table query, and against actual measured runtimes. Report where the
   estimates were wrong and connect it back to L01 §2.4 — you should find that your errors are
   dominated by cardinality estimation, exactly as the literature says.

## 6. Self-check

1. State batch processing's reliability model in one sentence and say what each part buys.
2. Why is the shuffle the whole job, and what does sorting achieve?
3. What is a combiner, what condition does it require, and which common aggregates fail it?
4. Describe skew, its symptom, and five remedies.
5. What is lineage-based fault tolerance and what does it trade against materialisation?
6. Why must batch output be written to a new location and swapped?
7. Why does object storage complicate that, and what solves it?
8. What was the Lambda architecture and why is it now usually the wrong choice?
9. In what sense is batch a special case of streaming?
10. Give three things that still genuinely favour batch.

## 7. Primary sources

- **Dean & Ghemawat, "MapReduce: Simplified Data Processing on Large Clusters" (OSDI 2004).**
- **Zaharia et al., "Resilient Distributed Datasets" (NSDI 2012)** — lineage-based fault tolerance.
- Kleppmann, *Designing Data-Intensive Applications*, ch. 10 — the Unix-heritage argument in full.
- Isard et al., "Dryad" (EuroSys 2007) — the general DAG model.
- Armbrust et al., "Spark SQL" (SIGMOD 2015) — the optimiser, and the convergence with databases.
- Chambers et al., "FlumeJava" (PLDI 2010) — the pipeline abstraction that became Beam.
- Graefe, "Query Evaluation Techniques for Large Databases" (ACM Computing Surveys, 1993) — joins
  and sorting, comprehensively; still the best reference.
- Blanas et al., "A Comparison of Join Algorithms for Log Processing in MapReduce" (SIGMOD 2010).

---

**Previous:** [L06](L06-column-stores.md) · **Next:**
[L08 — Stream Processing, Event Time, and Windows](L08-stream-processing.md)
