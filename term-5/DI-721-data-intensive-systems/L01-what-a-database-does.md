# DI-721 · Lesson 01 — What a Database Actually Does

**Estimated study time:** 4 hours
**Prerequisites:** PY-602 L08 (I/O and syscalls), CS-621 L01 (asymptotics)

---

## 1. Orientation

Most working programmers treat a database as an oracle: you send SQL, results come back, and
the middle is somebody else's problem. That works until it doesn't — until a query that ran in
8 ms yesterday takes 40 seconds today, or a "small" schema change locks a table for an hour, or
the p99 latency triples when the dataset crosses some invisible threshold.

Every one of those surprises comes from a mechanism that is not hidden at all, just unexamined.
This lesson opens the box. Not to make you a database implementer, but because **you cannot
predict a system's behaviour from its interface** — and prediction is what distinguishes an
engineer from a user.

The organising claim for the whole course:

> A database is a stack of layered mechanisms — a parser, a planner, an execution engine, an
> access-method layer, a buffer pool, a transaction manager, a log — and almost every
> performance surprise is one of those layers behaving exactly as designed, under conditions
> you did not know it assumed.

By the end of this lesson you should be able to look at a query plan and a latency number and
form a *hypothesis* about which layer is responsible. That skill compounds for the rest of your
career.

## 2. Theory

### 2.1 The five components

Hellerstein, Stonebraker and Hamilton's architecture paper divides a database into five
components, and the division is still accurate twenty years later:

1. **Process manager** — connections, admission control, threads or processes per client. The
   layer that decides *whether* your query runs now.
2. **Query processor** — parsing, rewriting, planning, optimisation, execution.
3. **Transactional storage manager** — buffer pool, access methods, lock manager, log manager.
   This is where ACID actually happens.
4. **Shared utilities** — memory management, replication, backup, monitoring, catalog.
5. **Client communications** — the wire protocol.

Two things are worth noticing straight away. First, most performance problems live in
components 2 and 3, and they have entirely different diagnostics: a planner problem shows up as
a bad plan for a query that is fine in isolation, a storage problem shows up as a good plan that
is slow. Second, component 1 is where *overload* manifests, and it explains the common
observation that a database "falls off a cliff" — admission control is a queue, and queues have
knees (PY-601 L09).

### 2.2 The memory hierarchy is the whole story

Every design decision in a storage engine is a response to the same set of numbers. Approximate,
and the exact values matter less than the ratios:

| Operation | Latency | Relative |
|---|---|---|
| L1 cache reference | ~1 ns | 1 |
| Main memory reference | ~100 ns | 100 |
| NVMe SSD random read (4 KB) | ~50–100 µs | ~10⁵ |
| Network round trip within a datacentre | ~0.5 ms | ~10⁶ |
| Disk (rotational) seek | ~5–10 ms | ~10⁷ |

Three consequences that explain almost everything in the next five lessons:

- **The unit of I/O is a page, not a byte.** Reading 4 KB costs almost exactly what reading
  100 bytes costs, so data structures are designed around pages. This is why B-trees have
  enormous fan-out — hundreds of children per node — where an in-memory binary tree has two.
- **Sequential beats random by an order of magnitude**, even on SSDs (where the gap narrowed but
  did not close, and where writes have their own asymmetry because of erase blocks). This is the
  entire argument for the log-structured designs of L02.
- **The right measure of an algorithm's cost is I/Os, not comparisons.** CS-621's RAM model
  charges 1 for every memory access; the **external memory model** (Aggarwal–Vitter) charges 1
  per block transfer of size B between a memory of size M and unbounded disk. In that model, a
  B-tree search is O(log_B n) rather than O(log₂ n), and sorting n items is
  O((n/B) log_{M/B} (n/B)) rather than O(n log n) — which is why external sorting looks nothing
  like an in-memory sort. Changing the cost model changes which algorithm wins; that is the
  lesson, and it generalises well beyond databases.

### 2.3 Pages, records, and the buffer pool

Storage is organised into fixed-size **pages** (commonly 4–16 KB). A page holds records, usually
with a **slotted page** layout: a header, a slot array growing from the front, and record data
growing from the back, with free space in the middle. The slot array's indirection means a record
can move within its page during compaction without invalidating references to it — a small design
choice with large consequences for how updates work.

The **buffer pool** is the database's cache of pages in memory, and it is not the OS page cache:
databases manage their own because they know things the OS does not — that this page is a leaf
being scanned once and should not evict the root, that this page must not reach disk before its
log record does. Key mechanisms:

- **Pinning**: a page in use cannot be evicted.
- **Dirty pages**: modified in memory, not yet written. The *only* reason correctness survives a
  crash with dirty pages in flight is the log (§2.5).
- **Eviction policy**: LRU is the naive choice and is defeated by a single large sequential scan,
  which evicts everything useful in favour of pages that will never be read again. Real systems
  use LRU-K, CLOCK, or scan-resistant variants like 2Q. This is worth internalising as a general
  principle: *a cache policy is an assumption about the access pattern, and a workload that
  violates the assumption turns the cache into overhead.*

**The five-minute rule** (Gray and Putzolu, 1987, restated periodically as hardware changes)
gives the economics: a page should be kept in memory rather than re-read from disk if it is
accessed more often than roughly every five minutes, where the interval is derived from the price
ratio of memory to storage and the cost per I/O. What is striking is that the rule's *number* has
stayed near five minutes across four decades of hardware change, because the prices moved
together. It is a good example of a result that survives because it is stated as a ratio.

### 2.4 From SQL to a plan

A query goes through: **parse** (syntax → AST), **bind** (resolve names against the catalog),
**rewrite** (normalise, unnest subqueries, push down predicates), **plan** (choose among
equivalent physical implementations), **execute**.

Planning is the interesting part. A declarative query does not say *how*, so the optimiser must
choose: which index (or none), which join algorithm (nested loop, hash, merge), which join order,
whether to sort or hash for aggregation. Join ordering alone is exponential in the number of
relations, so optimisers use dynamic programming (System R's algorithm, CS-621 L04's technique in
its most commercially important application) with heuristics and a cutoff.

The choice is driven by a **cost model** fed by **statistics** — table cardinalities, column
histograms, distinct-value counts. And here is the single most useful thing to know about query
performance:

> **Most catastrophically bad plans are caused by bad cardinality estimates, not by a bad cost
> model.** The optimiser chose correctly given what it believed; it believed the join would
> produce 40 rows and it produced 4 million.

Estimation errors compound multiplicatively through a join tree, so a small error at the leaves
becomes an enormous one at the root. That is why: stale statistics cause sudden plan regressions;
correlated predicates (`city = 'Dallas' AND state = 'TX'`) are systematically underestimated by
optimisers that assume independence; and `EXPLAIN ANALYZE`, which shows estimated *and* actual
row counts side by side, is the diagnostic that matters — not `EXPLAIN`, which shows only what
the optimiser believed.

**Execution** is usually one of three models: the **iterator/Volcano** model (each operator
exposes `next()`, pulling one row at a time — simple, and dominated by per-row function call
overhead), **vectorised** execution (each `next()` returns a batch of ~1,000 values, amortising
the overhead and enabling SIMD — this is PY-602 L05's vectorisation argument in a different
domain), and **compilation** (generate and JIT machine code for the specific query). Modern
analytical systems are vectorised, compiled, or both.

### 2.5 The log, and why it comes first

Durability with acceptable performance requires one idea: **write-ahead logging**.

> **The WAL rule.** The log record describing a change must reach stable storage *before* the
> data page containing that change does.

Given that rule, a crash is recoverable: replay the log forward to redo committed changes that
had not reached the data pages, and undo uncommitted ones. **ARIES** is the canonical algorithm
(analysis, redo, undo, with log sequence numbers, compensation log records for undo, and
fuzzy checkpointing so checkpoints don't stop the world).

Why this is a good trade: log writes are **sequential** and data-page writes are **random**
(§2.2). Turning random durable writes into sequential ones is the trick, and the data pages can
then be written lazily, in batches, in a better order.

Two things follow that people get wrong in practice. First, **`fsync` is the actual durability
boundary, not `write`** — data in the OS page cache is not durable, and the historical record of
databases losing data to misunderstood `fsync` semantics (including PostgreSQL's "fsyncgate" in
2018, where a failed `fsync` could be reported once and then forgotten by the kernel) should make
you humble about this. Second, **the log is not merely a recovery mechanism; it is a stream of
every change in commit order**, which makes it the natural source for replication (DS-701 L03) and
for change data capture (L09). "The log is the truth and the tables are a cache of it" is the
central idea of the second half of this course, and it starts here.

### 2.6 The honest question: which database?

The engineering answer is not a product name, it is a set of questions:

- **What is the access pattern?** Point lookups by key, small range scans, large aggregations, or
  full scans? Read-heavy or write-heavy? Storage engines are bets on this (L02).
- **What is the working set relative to memory?** A dataset that fits in RAM makes most of this
  lesson moot; one that is 100× RAM makes it everything.
- **What consistency and isolation do you actually need**, per operation (L04, and DS-701 L04)?
- **What is the write pattern?** Appends, in-place updates, deletes, or a mix? Deletes and
  updates are where storage engines differ most.
- **How will the schema change**, and who consumes the data downstream (L10)?
- **What must the system do when a node fails** (DS-701)?

Stonebraker's "one size fits all" argument is that these requirements are different enough that
specialised engines beat general ones by one to two orders of magnitude on their own workloads,
which is why the landscape fragmented into OLTP stores, analytical stores, time-series stores,
search engines and streaming systems. The counter-argument — that operational complexity of five
systems exceeds the performance benefit, and that a good general-purpose database is within a
factor of a few on most workloads — is also often correct. The judgement is about *your*
workload's distance from the general case, and neither slogan substitutes for measuring it.

## 3. Construction: instrumenting a real database

Build in `mpse/di721/l01/`. Use PostgreSQL in Docker (the lab setup has it); the exercises
transfer to any engine but the specific commands are Postgres.

**Stage 1 — a dataset with structure.** Generate a synthetic dataset large enough to exceed
`shared_buffers` — an orders/customers/line-items schema of a few million rows. Include a
*correlated* pair of columns (city and state) and a *skewed* column (a status where 95% of rows
share one value). Those two properties are what make the later stages interesting.

**Stage 2 — the plan.** Run `EXPLAIN (ANALYZE, BUFFERS)` on ten queries of varying shape. For
each, record: estimated rows, actual rows, the ratio, the chosen plan, buffer hits and reads, and
the time. Build a table. Do not optimise anything yet — this stage is about learning to read.

**Stage 3 — break the estimator.** Find three queries where the estimate is off by more than 10×.
The correlated columns and the skewed column will get you two; find a third. For each, explain
*why* the estimator was wrong in terms of the independence assumption or the histogram's
resolution. Then fix one with extended statistics (`CREATE STATISTICS`) and re-measure.

**Stage 4 — the buffer pool.** With `pg_buffercache`, observe which pages are resident. Run a
large sequential scan and measure the effect on the hit rate of a subsequent point-lookup
workload. Then vary `shared_buffers` across three settings and plot throughput. You are
reproducing the cache-pollution problem of §2.3 with your own hands.

**Stage 5 — sequential versus random.** Write a small benchmark *outside* the database: read
1 GB sequentially, then read the same 1 GB in random 4 KB blocks (use `O_DIRECT` or at least drop
caches between runs). Compute the ratio on your hardware. Keep this number; it is the constant
that justifies L02's entire design space, and it is different on your machine than in any book.

**Stage 6 — the WAL.** Measure transaction throughput with `synchronous_commit` on and off, and
with `fsync` off (in a throwaway container only — this configuration can corrupt data, which is
itself the point). Then, with `pg_waldump`, look at the actual log records for a simple update.
Write a paragraph on what you saw and what each setting is trading.

**Stage 7 — the cliff.** Increase concurrent client count in steps and plot throughput and p99
latency. Find the knee. Then add a connection pooler (PgBouncer) and repeat. Explain the shape
using Little's Law and the admission-control framing of §2.1. This is the same curve as PY-601
L09's and DS-701 L09's; seeing it in a third context is the point.

## 4. Failure modes

- **Treating the database as an oracle.** If you cannot form a hypothesis about *why* a query is
  slow before you look, you are guessing.
- **Reading `EXPLAIN` instead of `EXPLAIN ANALYZE`.** The former shows beliefs; the discrepancy
  between belief and reality is the diagnosis.
- **Assuming statistics are current.** After a bulk load or a large delete, they are not, and the
  plan regression that follows will look mysterious.
- **Ignoring correlation.** Optimisers assume independence by default; real data is correlated,
  and the estimate is wrong by the product of the correlations.
- **Benchmarking with a dataset that fits in memory** and deploying against one that does not.
  The two regimes have different bottlenecks and the benchmark tells you nothing about the
  target.
- **Confusing `write` with durability.** Only `fsync` (and only when it succeeds, and only when
  the drive is not lying about its write cache) is durable.
- **Tuning parameters found on the internet.** Every parameter is a trade; if you cannot say what
  it trades against what, you are performing a ritual.
- **Adding connections to fix throughput.** Past the knee, more concurrency reduces throughput.
  Pool instead.

## 5. Exercises

### Warm-up (30 min)

1. Give the memory hierarchy latencies as ratios and derive from them (a) why B-trees have high
   fan-out and (b) why write-ahead logging is faster than writing data pages synchronously.
2. In the external memory model, state the cost of a B-tree search and of sorting, and explain
   what changed relative to the RAM model.
3. Explain why LRU is defeated by a sequential scan and name one policy that resists it.

### Core (3 h)

4. Complete Stages 1–3. Deliver the table from Stage 2 and the three estimator failures with
   explanations.
5. Complete Stages 4–5. Deliver the sequential-versus-random ratio for your hardware and the
   buffer-pool plot.
6. Complete Stage 6, and write 400 words on what `synchronous_commit = off` actually risks —
   being precise about *what* can be lost and what cannot.
7. Take a slow query from a system you work with (or construct one). Write the diagnosis: which
   of §2.1's components is responsible, what evidence supports that, and what you would change.
   If you have no such system, use a query from Stage 2 and make it 100× slower by a schema
   change, then diagnose it as if you had not caused it.

### Challenge

8. Complete Stage 7, including the Little's Law analysis with your measured numbers.
9. Implement a **cardinality estimator** for a single table: equi-depth histograms per column,
   with the independence assumption for conjunctions. Measure its error on your Stage 1 dataset
   across 100 random predicates, then implement a two-column joint histogram for the correlated
   pair and quantify the improvement. Write up what this tells you about why real optimisers do
   not simply keep joint histograms for everything.

## 6. Self-check

1. Name the five components of a database and say which layer a plan problem versus a storage
   problem lives in.
2. Why is the page, not the byte, the unit of I/O, and what does that imply for tree fan-out?
3. State the external memory model and the B-tree search cost within it.
4. What does the buffer pool do that the OS page cache cannot?
5. What is the five-minute rule and why has its number been stable for decades?
6. What is the single most common cause of catastrophically bad query plans?
7. Why are correlated predicates systematically mis-estimated?
8. State the WAL rule and explain why it is a performance win as well as a correctness one.
9. Distinguish iterator, vectorised and compiled execution.
10. Give five questions you would ask before choosing a database, and say what each one's answer
    changes.

## 7. Primary sources

- **Hellerstein, Stonebraker & Hamilton, "Architecture of a Database System" (Foundations and
  Trends in Databases, 2007)** — read the whole thing; it is the map for this course.
- **Gray & Reuter, *Transaction Processing: Concepts and Techniques*** — the reference, dense and
  worth owning.
- Mohan et al., "ARIES: A Transaction Recovery Method…" (TODS 1992).
- Aggarwal & Vitter, "The Input/Output Complexity of Sorting and Related Problems" (CACM 1988) —
  the external memory model.
- Gray & Putzolu, "The 5 Minute Rule for Trading Memory for Disc Accesses" (1987), and the
  periodic restatements.
- Selinger et al., "Access Path Selection in a Relational Database Management System"
  (SIGMOD 1979) — the System R optimiser, and still the shape of every optimiser since.
- Leis et al., "How Good Are Query Optimizers, Really?" (VLDB 2015) — the empirical answer, and
  the source of the cardinality-estimation claim in §2.4.
- Stonebraker & Çetintemel, "'One Size Fits All': An Idea Whose Time Has Come and Gone"
  (ICDE 2005).
- Kleppmann, *Designing Data-Intensive Applications*, ch. 3.

---

**Next:** [L02 — Storage Engines: B-Trees and LSM-Trees](L02-storage-engines.md)
