# DI-721 — Problem Sets

The three problem sets build the data layer of the Term 5 artifact: a storage engine you wrote, a
transactional layer whose isolation you can state, and a pipeline that moves data out of it without
losing its meaning. Do them in order and keep the code — Problem Set 3 depends on the engine from
Problem Set 1.

**A note on measurement.** This course's standard of evidence is a number you produced, with the
method stated. "LSM-trees have higher write amplification" is a sentence from a book; "my levelled
compaction wrote 14.2× the logical bytes on this workload, versus 3.1× for size-tiered, and here is
the workload" is an answer. Every part below that asks you to *measure* means: state the workload,
state the method, report the number, and say what would change it.

**A note on predictions.** Several parts ask you to predict a result before measuring it. Do this
in writing, and keep the wrong predictions in your submission. The gap between what you expected
and what happened is the most valuable thing this course produces, and erasing it wastes the
lesson.

---

## Problem Set 1 — A Storage Engine, Benchmarked Honestly
**Covers L01–L03 · Budget: 22–26 hours**

**Part A — Reading a database (L01).** Stages 1–3 of L01 §3: the dataset with correlated and
skewed columns, the ten-query plan table with estimated versus actual rows, and three queries where
the estimate is off by more than 10× with the explanation for each. Then fix one with extended
statistics and re-measure.

**Part B — The hardware constants (L01).** Stages 4–5: the buffer-pool measurements across three
`shared_buffers` settings, the cache-pollution demonstration, and your machine's
sequential-versus-random ratio. Record that ratio prominently; it is referenced in Parts E and H.

**Part C — Durability, measured (L01).** Stage 6: throughput under three `synchronous_commit`
settings, the `pg_waldump` inspection of a simple update, and 400 words on what
`synchronous_commit = off` actually risks — precisely what can be lost and what cannot.

**Part D — The LSM (L02).** Stages 1–4: the naive baseline, memtable and WAL with crash-recovery and
corruption tests, SSTables with the read-amplification curve, and bloom filters with the
theoretical-versus-measured false-positive rate. If those two rates disagree, find out why before
proceeding.

**Part E — Compaction (L02).** Stages 5–6: size-tiered and levelled compaction, each instrumented
for all three amplifications, and the comparison table. Then 500 words interpreting the table in
RUM terms, and a statement of which strategy you would choose for three named workloads.

**Part F — Deletion and stalls (L02).** Stages 7–8: the deliberately-introduced tombstone
resurrection bug and its fix; the tombstone scan-degradation plot; the write-stall reproduction; and
the gradual-backpressure implementation with its effect on the latency distribution.

**Part G — Against a real engine (L02).** Stage 9: your LSM versus SQLite across four workloads.
Predictions written before measurement. Report shapes, explain each from the mechanisms, and
account for every prediction you got wrong.

**Part H — Indexing (L03).** Stages 1–4: the selectivity crossover plot reconciled against your
Part B ratio, the composite-order 4×2 table, the covering-index measurement including the
concurrent-update degradation, and the write-cost chart across 0/1/3/6 indexes.

**Part I — Indexes that hurt (L03).** Stages 5–7: HOT updates in three configurations with all
three numbers explained; four non-sargable predicates with fixes (including the implicit-coercion
case); and the parameter-sniffing demonstration with its mitigation.

**Design note (1,500–2,000 words).** Where your intuition about storage was wrong, specifically.
The RUM position you would choose for a workload you actually have, with the numbers that justify
it. What the bloom filter false-positive discrepancy (if you had one) taught you about trusting
theoretical formulas over measurement. And the one thing you now know about your production
database that you did not know before Part A.

---

## Problem Set 2 — Transactions, Anomalies, and a Column Store
**Covers L04–L06 · Budget: 22–26 hours**

**Part A — The anomaly laboratory (L04).** Stages 1–3: the deterministic interleaving harness, the
7×4 anomaly matrix for your primary engine, and the same matrix for a second engine. Every cell
where your matrix differs from the ANSI table must be explained. This harness is a permanent asset;
build it well.

**Part B — Lost updates, four ways (L04).** Stage 4: naive read-modify-write, atomic `UPDATE`,
`SELECT … FOR UPDATE`, and version-based CAS, each under 100 concurrent clients, with correctness
*and* throughput reported.

**Part C — Write skew (L04).** Stage 5: the doctors-on-call violation demonstrated under snapshot
isolation, then four fixes with measurements and a defended recommendation. Then find and document
one instance of the write-skew *shape* in a system you actually work with.

**Part D — Deadlocks and long transactions (L04).** Stages 6–7: a deliberate deadlock with the
engine's detection observed, a correct retry wrapper (re-executing the whole transaction), and the
bloat experiment with its plot, its monitoring query and your chosen alert threshold.

**Part E — MVCC implemented (L05).** Stages 1–3: versioned storage, the visibility function with
its property test, snapshot isolation with first-committer-wins, and the L04 catalogue re-run
against your own engine — showing write skew surviving.

**Part F — Serializability (L05).** Stages 4–5: per-statement snapshots and the
lock-wait-re-evaluation case; then SSI with rw-antidependency tracking and pivot detection, the
doctors scenario aborting, and the SI-versus-SSI throughput and abort-rate chart across contention
levels.

**Part G — The cost of MVCC (L05).** Stage 7: garbage collection with a horizon, the
long-transaction
pathology reproduced with its storage and latency plot, the recovery, and the production monitoring
query. Then Stage 8: your engine's abort-rate curve compared against Postgres's, with the
differences explained in terms of read-set granularity.

**Part H — Columnar (L06).** Stages 1–5: the row baseline, the columnar layout, the encodings,
operating on encoded data, and vectorised execution — each measured *separately*, culminating in the
factor-decomposition table with the gap between the product and the measured end-to-end speedup
explained.

**Part I — Layout and reality (L06).** Stages 6–7: zone maps on sorted versus shuffled data, and
your implementation against DuckDB on Parquet with the three biggest reasons you lost, plus the
small-files measurement.

**Design note (1,500–2,000 words).** The isolation policy you would write for a real OLTP
application: which level per transaction and why, with the anomalies each permits named and
addressed. What implementing SSI taught you that reading about it did not. And the honest answer to
"is serializable too slow?", supported by your Part F chart rather than by folklore.

---

## Problem Set 3 — A Pipeline That Is Correct Under Failure and Evolvable
**Covers L07–L10 · Budget: 24–28 hours**

**Part A — Batch (L07).** Stages 1–3: the MapReduce framework, the phase-breakdown chart, and the
three-aggregate comparison (combinable, approximately combinable, holistic) with the algebraic
condition explained.

**Part B — Sorting and joins (L07).** Stages 4–5: external merge sort measured against the external
memory model's bound across memory budgets, and the three join strategies benchmarked across
relative sizes from 1:1 to 1:10,000 with the crossover chart and your planner rule.

**Part C — Skew (L07).** Stage 6: the Zipfian straggler, salting, and adaptive splitting, all three
measured, with the cost salting imposed on the small side stated explicitly.

**Part D — Fault tolerance and output (L07).** Stages 7–8: materialisation versus lineage recovery,
compared on both no-failure runtime and recovery time; then the atomic output protocol, including
the object-storage version where no atomic rename exists.

**Part E — Event time (L08).** Stages 1–4: the skewed event generator, the processing-time
irreproducibility demonstration, event-time windows with watermarks, and the
completeness-versus-latency curve across four watermark delays, with allowed lateness and a
late-data side output.

**Part F — Triggers, sessions, state (L08).** Stages 5–7: all three accumulation modes with correct
consumers for each *and* the deliberate mismatch producing a silently wrong result; session windows
including the merge case; and barrier-based checkpointing with recovery verified and checkpoint
duration measured against state size.

**Part G — Exactly-once and joins (L08).** Stages 8–9: the idempotent sink surviving three induced
failures, the non-idempotent sink producing duplicates, the paragraph naming exactly where the
guarantee ends; then the stream–stream join with its state measurement and the stream–table temporal
correctness demonstration with your defended choice.

**Part H — CDC and the log (L09).** Stages 1–4: polling CDC with all three failures demonstrated
(including the commit-order case), log-based CDC with the WAL-retention alert, snapshot-plus-stream
bootstrapping verified under concurrent writes, and — the required deliverable — the
CDC-versus-outbox
refactor comparison.

**Part I — Event sourcing (L09).** Stages 5–8: the event store with optimistic concurrency,
rehydration timings with and without snapshots, two projections with rebuild timings at 10⁵ and 10⁶
events, the read-your-writes anomaly with two fixes compared, and all three hard cases —
compensating event, crypto-shredding, poison message.

**Part J — Contracts and evolution (L10).** Stages 1–5: schemas in Avro and Protobuf with a
registry, the compatibility matrix with actual read behaviour verified, CI enforcement including the
non-transitive trap, a real data contract with continuously-running quality checks alerting the
producer, and the zero-downtime column split with reversibility demonstrated at each step.

**Part K — Migration and modelling (L10).** Stages 6–8: the naive-versus-batched migration
comparison, event-log upcasting with the archived-fixture replay test, and the star schema with
Type 1 versus Type 2 dimensions producing different answers to the same question — with a statement
of which is correct and under what condition the other would be.

**Design note (2,000–2,500 words).** Trace one field of data from the moment the application writes
it, through your storage engine, your CDC pipeline, your stream processor and your analytical
model, and state at each hop what guarantee holds and what could silently corrupt its meaning.
Then: the one place in your pipeline where you know the guarantee is weaker than you would like,
and what it would cost to fix. And the answer to the course's driving question, for the system you
built.

---

## Course position paper (1,500 words)

Choose one:

1. **"Event sourcing is adopted for the wrong reasons more often than any other pattern in modern
   data architecture."** Defend or refute, using your Problem Set 3 experience and at least two
   primary sources.
2. **"The unbundling of the database into a log plus derived views is a genuine architectural
   advance, not a fashion."** Argue it, addressing the operational cost honestly.
3. **"'One size fits all' is gone for good — but most organisations run too many specialised data
   systems, not too few."** Use Stonebraker's argument and the counter-argument, and your own
   measurements from L06 and L09.
4. **"Serializable isolation should be the default, and the folklore against it is thirty years out
   of date."** Argue from your Problem Set 2 Part F measurements, and address the cases where you
   are wrong.
5. **"Most data quality problems are contract problems, and most contract problems are ownership
   problems."** Defend or refute with reference to L10 §2.6 and a real organisation you know.

The structure is the one from `00-program/assessment-and-rubrics.md`: claim, grounds, the strongest
rebuttal you can construct, and the limits of your position. A paper that does not name a condition
under which its claim fails has not made a claim.

---

## Submission checklist (per set)

- [ ] Code in `courses/di721/psN/`, runnable from a clean clone via the README.
- [ ] Tests passing, with a stated coverage figure *and* a sentence on what coverage does not tell
      you here.
- [ ] `mypy --strict` and `ruff` clean, or every exception documented with a reason.
- [ ] All measurements reproducible: every measurement states the workload, the method, and what would change the number.
- [ ] Charts and tables as files, not as descriptions — a plot referred to but not produced does not
      count.
- [ ] `NOTE.md` — the design note.
- [ ] `MARK.md` — self-marked against the five-criterion rubric with one line of justification per
      criterion.
- [ ] `log/failures.md` updated with everything you got wrong on the way, including the predictions
      that were incorrect.
