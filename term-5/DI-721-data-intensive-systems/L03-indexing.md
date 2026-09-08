# DI-721 · Lesson 03 — Indexing, and When It Hurts

**Estimated study time:** 4.5 hours
**Prerequisites:** L01, L02

---

## 1. Orientation

"The query is slow — add an index" is the most common piece of database advice, and it is right
often enough to be dangerous. An index is not free storage magic; it is a **derived, redundant
data structure that must be kept consistent with the base data on every write**. You are trading
write throughput, storage and — importantly — *planner complexity* for read speed on a specific
access pattern.

This lesson is about making that trade deliberately. The organising claim:

> An index is a materialised answer to a question. It helps exactly the questions it materialises,
> costs on every write regardless, and its value collapses when the question changes or when the
> answer it materialises is not selective.

The three things most engineers get wrong, all covered below: they do not understand **why the
leftmost prefix rule exists** (§2.3), so they create redundant multi-column indexes; they do not
understand **selectivity** (§2.5), so they index columns where a scan would be faster; and they do
not know that indexes can make a query **slower** (§2.6), which sounds impossible until you have
seen it.

## 2. Theory

### 2.1 What an index is

An index maps values to row locations. Two structural decisions define it.

**Clustered versus secondary.** A **clustered index** *is* the table — the rows are stored in the
index's leaf pages, in key order (InnoDB's primary key, SQL Server's clustered index). A
**secondary index** stores the indexed columns plus a pointer to the row, and the pointer is
either a physical address (Postgres's `ctid`) or the clustered key (InnoDB). This distinction has
large practical consequences:

- With a clustered primary key, a secondary index lookup is *two* lookups: the secondary index to
  get the primary key, then the primary key index to get the row. A wide primary key inflates
  every secondary index.
- With physical pointers, moving a row (an update that does not fit in the page) means updating
  every index — which is why Postgres has HOT updates (§2.4) and why InnoDB chose the other
  design.
- The clustered order determines physical locality: range scans on the clustering key are
  sequential; range scans on anything else are random.

**Covering.** If an index contains every column a query needs, the query is answered from the
index alone — an **index-only scan**, with no table access at all. This is the largest single win
available from indexing, and it is why `INCLUDE`d columns (non-key payload columns) exist.

### 2.2 Index types, and the question each answers

| Type | Answers | Notes |
|---|---|---|
| **B-tree** | equality, ranges, ordering, prefix `LIKE 'abc%'` | The default, and right for most things. Supports `ORDER BY` without a sort. |
| **Hash** | equality only | No ranges, no ordering. Rarely worth it over a B-tree; occasionally wins on very large equality-only indexes. |
| **Bitmap** | low-cardinality columns, combined with AND/OR | Excellent for analytics; poor under concurrent updates. |
| **Inverted** | "which documents contain this term" | Full-text search; GIN in Postgres. Also used for array/JSON containment. |
| **GiST / SP-GiST** | geometric, range containment, nearest-neighbour | A framework rather than one structure. |
| **BRIN** | "which blocks could contain this range" | Tiny; works only when physical order correlates with the value (time-series appended in order). Huge win when it applies, useless when it does not. |
| **LSM / skip-list** | ordered access in an LSM engine | See L02. |

Two special forms worth knowing because they solve real problems cheaply:

- **Partial index**: `CREATE INDEX … WHERE status = 'pending'`. If 99% of rows are `done` and you
  only ever query `pending`, the index is 1% the size and stays hot in cache. This is often a
  better answer than a full index and is badly under-used.
- **Expression index**: `CREATE INDEX … ON t (lower(email))`. Required if the query applies a
  function to the column — because of §2.6's sargability rule.

### 2.3 Composite indexes and the leftmost prefix rule

An index on `(a, b, c)` sorts by `a`, then `b` within equal `a`, then `c`. It is a *dictionary
ordering*, and everything follows from that:

- It serves `WHERE a = ?`, `WHERE a = ? AND b = ?`, `WHERE a = ? AND b = ? AND c = ?`.
- It serves `WHERE a = ? AND b > ?` — but only as far as `b`. Once you use a range on `b`, the `c`
  portion is not usable for filtering, because within the range of `b` values, `c` is not sorted
  globally.
- It does **not** serve `WHERE b = ?` alone. There is no way to find all rows with a given `b`
  without scanning, exactly as you cannot find all words whose second letter is 'q' in a
  dictionary without reading it.

Hence the **leftmost prefix rule**: an index on `(a, b, c)` also serves as an index on `(a)` and
`(a, b)`, so creating those separately is waste. And hence the **column order rule**: equality
columns first, then the one range column, then columns needed only for output. Order matters
enormously and is the most common composite-index mistake.

An index also provides **ordering**, so `ORDER BY a, b` over an index on `(a, b)` needs no sort —
which can be the difference between a query that streams the first ten rows immediately and one
that materialises and sorts a million.

### 2.4 The cost of an index

On every insert, update of an indexed column, and delete, every affected index must be updated.
Concretely:

- **Write amplification.** A table with six indexes turns one logical write into seven structural
  writes, each with its own page reads, page writes and WAL records.
- **Random I/O.** Index updates are scattered by definition — the index is in a different order
  from the table.
- **Contention.** Under concurrency, index pages are hot spots (L02's right-edge contention).
- **Space.** Indexes routinely exceed the size of the table they index. Check; people are usually
  surprised.
- **Vacuum / maintenance load.** In an MVCC system, index entries accumulate for dead row versions
  and must be cleaned up (L05).

Engines mitigate this. Postgres's **HOT (heap-only tuple)** update avoids touching indexes when
the updated columns are not indexed *and* the new version fits in the same page — which is why
`fillfactor` is a tuning knob and why adding an index to a hot column can degrade update
throughput far more than the index's own cost suggests. InnoDB's **change buffer** defers
secondary index maintenance for pages not in the buffer pool. Both are worth knowing because both
explain otherwise-baffling performance changes.

### 2.5 Selectivity, and when a scan wins

**Selectivity** is the fraction of rows a predicate matches. The rule of thumb, which follows
directly from L01 §2.2:

> Below roughly 1–10% selectivity, an index scan wins. Above it, a sequential scan wins — because
> the index scan does one *random* I/O per matching row while the sequential scan does purely
> sequential I/O.

The crossover is a property of your hardware's random/sequential ratio (which you measured in L01
Stage 5), the row size, and the correlation between index order and physical order. It is not a
universal constant, and Postgres exposes exactly these as `random_page_cost` and
`effective_cache_size` — parameters that are frequently left at defaults tuned for rotational
disks and therefore systematically discourage index use on SSDs.

Two consequences:

- **Indexing a boolean, or a status column where 95% of rows share a value, is usually useless**
  for the common value and valuable only for the rare one — which is exactly the case for a
  *partial* index.
- **Skewed data means a single plan is wrong for some parameter values.** `WHERE status = 'rare'`
  wants the index; `WHERE status = 'common'` wants a scan. With a prepared statement and a cached
  generic plan, one of them gets the wrong plan. This is the **parameter sniffing** problem, and
  its symptoms — "the query is fast except for certain customers" — are unmistakable once you know
  it exists.

### 2.6 When indexes hurt

Five cases, each of which you should be able to recognise:

1. **The write-heavy table with speculative indexes.** Six indexes "just in case" on a table
   taking 10,000 inserts/second is a 7× write amplification for reads that may never happen.
   Measure index usage (`pg_stat_user_indexes`) and drop what is unused; most systems have
   several.
2. **The non-sargable predicate.** `WHERE YEAR(created_at) = 2024` or `WHERE email LIKE '%@x.com'`
   or `WHERE col + 0 = 5` cannot use an index on `col` — the index sorts `col`, not `f(col)`. Fix
   by rewriting to a range (`created_at >= '2024-01-01' AND < '2025-01-01'`) or by creating an
   expression index. Implicit type coercion causes this silently: comparing a `varchar` column to
   an integer parameter can disable the index with no visible sign.
3. **The redundant index.** `(a)` alongside `(a, b)` — the first is subsumed by the leftmost
   prefix rule and is pure cost.
4. **The low-selectivity index the planner uses anyway.** Bad statistics can make the planner
   choose an index scan that reads 40% of the table one random page at a time — far slower than
   the sequential scan it rejected. This is the case that surprises people, and `EXPLAIN ANALYZE`
   showing a huge row count under an index scan is the tell.
5. **The index that stops the planner from doing better.** Occasionally an index's existence
   causes the optimiser to abandon a better plan (a merge join, a bitmap combination) because the
   costed index path looks cheaper than it is. Rare, real, and the reason "just add an index" is
   not a safe default.

### 2.7 A method for indexing

1. **Start with the queries**, not the schema. List the actual access patterns, with frequencies.
2. **For each**, identify the equality predicates, the range predicate (at most one is usable),
   the ordering, and the output columns.
3. **Design the composite index** in that order: equality columns, range column, then included
   columns to make it covering if the payload is small.
4. **Look for a partial index** if a predicate is nearly always present.
5. **Check for redundancy** against existing indexes using the prefix rule.
6. **Measure** `EXPLAIN ANALYZE` before and after, on production-scale data with a production-like
   distribution. Small or uniform test data will mislead you in both directions.
7. **Re-check write throughput** after adding it, on the write path that matters.
8. **Revisit periodically.** Drop unused indexes. Access patterns change; indexes rarely get
   deleted, and the accumulated cost is invisible until you look.

## 3. Construction: index design, measured

Build in `mpse/di721/l03/`, on L01's dataset and on your own L02 engine.

**Stage 1 — the crossover point.** On a table of several million rows, run a query whose
selectivity you can vary from 0.01% to 50% in steps. Force an index scan and a sequential scan
(`SET enable_seqscan`) at each point and plot both curves. Find the crossover on your hardware.
Compare it against the ratio you measured in L01 Stage 5 and reconcile the two numbers.

**Stage 2 — composite order.** Create `(a, b)` and `(b, a)` on the same table. Run four queries —
`a=?`, `b=?`, `a=? AND b=?`, `a=? AND b>?` — against each and record which index the planner picks
and the resulting time. Produce the 4×2 table. Then add a third column and demonstrate the range
column truncating usability.

**Stage 3 — covering.** Take a query that reads three columns and filters on one. Measure it with
a plain index, then with an index that `INCLUDE`s the other two. Confirm the plan changes to an
index-only scan and record the speedup and the index size increase. Then make the table receive
concurrent updates and observe the index-only scan degrading (in Postgres, because the visibility
map is not all-visible) — this is the detail that makes index-only scans less reliable in practice
than in theory.

**Stage 4 — the write cost.** Measure insert throughput on a table with 0, 1, 3 and 6 indexes.
Plot it, and also record WAL bytes generated per insert. This chart is the one to show anyone who
proposes an index "just in case".

**Stage 5 — HOT updates.** Set up a table where the updated column is *not* indexed and
`fillfactor` leaves room; measure update throughput. Then index the updated column and measure
again. Then fill the page and measure again. Explain all three numbers.

**Stage 6 — non-sargable predicates.** Construct four non-sargable queries (function on column,
leading wildcard, arithmetic on column, implicit type coercion) and show the plan failing to use
the index. Fix each — three by rewriting, one with an expression index — and show the plan
changing. The implicit-coercion case is the one to look at hardest, because in production it
appears without anyone having written anything obviously wrong.

**Stage 7 — parameter sniffing.** With a skewed column, prepare a statement and execute it with a
rare value and then a common one. Demonstrate the generic plan being wrong for one of them.
Measure the cost, then investigate what your engine offers (`plan_cache_mode`, replanning
thresholds) and document the mitigation.

**Stage 8 — index your own engine.** Add a secondary index to your L02 LSM: a second keyspace
mapping `indexed_value → primary_key`, maintained transactionally with the base write (which is
L08's dual-write problem, in miniature, inside one system — note the connection). Measure the write
amplification you introduced and implement one query that uses it. Then handle the hard case:
updating an indexed value requires deleting the old index entry, which requires *reading the old
value first*. Measure what that read-before-write costs, and you will understand why write-heavy
systems avoid secondary indexes.

## 4. Failure modes

- **Adding an index without measuring the write path.** The read got faster; nobody checked what
  else got slower.
- **Redundant indexes from ignorance of the prefix rule.** Pure cost, very common.
- **Wrong column order in a composite index.** The index exists, looks reasonable, and is not
  used — or is used badly.
- **Indexing a low-cardinality column** where a partial index or nothing is correct.
- **Non-sargable predicates**, especially from implicit type coercion, which produces no warning.
- **Benchmarking on uniform data.** Real distributions are skewed; skew is what breaks plans.
- **Never dropping indexes.** Every system accumulates unused ones; the cost is continuous and
  silent.
- **Trusting `random_page_cost` defaults on SSDs.** The default discourages index use on hardware
  where random reads are far cheaper than the default assumes.
- **Creating an index on a large table without `CONCURRENTLY`** (or the equivalent), taking a
  write lock for the duration. An avoidable outage.

## 5. Exercises

### Warm-up (30 min)

1. State the leftmost prefix rule and explain it with the dictionary analogy. Then say which of
   these an index on `(a, b, c)` serves: `b = ?`; `a = ? AND c = ?`; `a = ? AND b > ? AND c = ?`;
   `ORDER BY a, b`.
2. Define selectivity and state the rule of thumb for the index/scan crossover, with its
   justification from the memory hierarchy.
3. Give four non-sargable predicates and the fix for each.

### Core (3 h)

4. Complete Stages 1–3. Deliver the crossover plot, the 4×2 composite table, and the covering
   measurement.
5. Complete Stages 4–5 and deliver both charts with an interpretation of the HOT-update numbers.
6. Complete Stages 6–7.
7. Take a real schema you work with. List its indexes, identify redundancy by the prefix rule,
   identify unused indexes from the engine's statistics, and propose a change set with the
   expected effect on both reads and writes. Then state how you would validate it safely.

### Challenge

8. Complete Stage 8, including the read-before-write measurement and a written account of what it
   implies for secondary indexes in write-optimised stores.
9. Write an **index advisor**: given a workload (a list of queries with frequencies) and a schema,
   enumerate candidate indexes, estimate the benefit using the engine's own hypothetical-index
   facility (`HypoPG` in Postgres) and the cost from your Stage 4 measurements, and output a
   recommended set under a storage budget. Then evaluate it: run the workload with your
   recommendations and with a hand-designed set, and report honestly where the advisor was worse.
   The interesting part of this exercise is where it fails — advisors systematically over-index,
   and understanding why is the lesson.

## 6. Self-check

1. Distinguish clustered from secondary indexes, and say what a secondary index lookup costs under
   each design.
2. What is a covering index and why is an index-only scan so much faster?
3. State the leftmost prefix rule and the column-order rule, and justify both from the sort order.
4. Why does a range predicate on `b` make `c` unusable in an index on `(a, b, c)`?
5. Give five costs of an index.
6. What is a HOT update and what disables it?
7. Explain the selectivity crossover and what hardware parameter moves it.
8. What is parameter sniffing and what does it look like in a bug report?
9. Give five situations in which an index makes things worse.
10. Give the eight-step method for designing an index.

## 7. Primary sources

- **Winand, *SQL Performance Explained*** (and use-the-index-luke.com) — the clearest treatment of
  composite indexes and sargability available anywhere; free online.
- Graefe, "Modern B-Tree Techniques" (FnTDB, 2011).
- Selinger et al., "Access Path Selection…" (SIGMOD 1979) — where selectivity estimation began.
- Leis et al., "How Good Are Query Optimizers, Really?" (VLDB 2015).
- Chaudhuri & Narasayya, "An Efficient Cost-Driven Index Selection Tool for Microsoft SQL Server"
  (VLDB 1997) — the origin of index advisors, and honest about their limits.
- The PostgreSQL documentation on index types, partial indexes and index-only scans — unusually
  good, and worth reading start to finish.
- Kleppmann, *Designing Data-Intensive Applications*, ch. 3 (secondary indexes section).

---

**Previous:** [L02](L02-storage-engines.md) · **Next:**
[L04 — Transactions and Isolation Levels](L04-transactions-and-isolation.md)
