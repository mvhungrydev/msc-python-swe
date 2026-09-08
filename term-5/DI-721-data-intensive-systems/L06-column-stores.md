# DI-721 · Lesson 06 — Column Stores and Analytical Processing

**Estimated study time:** 4.5 hours
**Prerequisites:** L01, L02; PY-602 L05 (vectorization), PY-602 L06 (caches and locality)

---

## 1. Orientation

Everything so far has assumed the OLTP shape: many small transactions, each touching a few rows,
reading whole rows by key. Analytical workloads have the opposite shape — few queries, each
scanning millions of rows but touching three columns out of eighty, aggregating rather than
retrieving.

Running the second workload on a system built for the first is one of the most common and most
expensive architectural mistakes in the industry, and the arithmetic for why is simple enough to
do in your head:

> A table of 80 columns, 100 bytes per row, 100 million rows = 10 GB. A query that reads three
> columns needs about 400 MB of data. A row store reads all 10 GB, because the unit of I/O is a
> page and every page contains all 80 columns. **A 25× penalty before any other consideration.**

Then compression multiplies the advantage (a column of one type with few distinct values
compresses 5–20×, where a row of mixed types compresses ~2×), and vectorised execution multiplies
it again. Order-of-magnitude differences between row and column stores on analytical queries are
not marketing; they are the product of three independent factors.

This lesson is about those factors, and about the honest limits: column stores are *bad* at the
things row stores are good at, and the interesting engineering is in knowing where the boundary
sits and what hybrid designs do about it.

## 2. Theory

### 2.1 The layouts

**Row-major (NSM, N-ary storage model)**: a page holds complete rows.
`[r1c1 r1c2 r1c3][r2c1 r2c2 r2c3]…`
Reading one whole row is one page read. Reading one column of a million rows touches every page.

**Column-major (DSM, decomposition storage model)**: each column is stored separately.
`[c1: r1 r2 r3 …][c2: r1 r2 r3 …]`
Reading one column is sequential and reads nothing else. Reading one whole row requires one
access per column plus a reassembly step (**tuple reconstruction**), which is the column store's
fundamental cost.

**PAX (Partition Attributes Across)**: rows are grouped into pages as in NSM, but *within* a page
the values are stored column-wise. You keep row locality at page granularity (so a row is one page
read) and get the cache and compression benefits within the page. This is the basis of Parquet and
ORC, and of most modern hybrid engines — the practical answer that gets most of both benefits.

This is not just a database concept. It is the array-of-structs versus struct-of-arrays decision
from PY-602 L04, at a different scale and with the same underlying justification. Recognising that
one idea appears at the register, cache, page and file level is worth more than either instance
alone.

### 2.2 Why columnar wins on analytics: four independent factors

**1. I/O reduction (projection).** Read only the columns the query names. The factor is
(columns read / total columns) — typically 5–30×.

**2. Compression.** A column is values of one type, often with low cardinality and local
similarity. That enables encodings that a mixed-type row cannot use:

| Encoding | Idea | Where it wins |
|---|---|---|
| **Run-length (RLE)** | store (value, count) | sorted or clustered low-cardinality columns |
| **Dictionary** | map values to small integers | strings with few distinct values — extremely common |
| **Bit-packing** | use exactly ⌈log₂(range)⌉ bits | small-range integers, and dictionary codes |
| **Frame of reference / delta** | store offsets from a base | timestamps, sorted IDs, monotonic keys |
| **Bitmap** | one bitmap per distinct value | very low cardinality, and combinable with AND/OR |

Typical results: 5–20× on real analytical data, versus ~2–3× for general-purpose compression of
rows. And compression is *not only* a storage saving — it is an I/O saving and often a CPU saving,
because less data moves through every level of the memory hierarchy.

**3. Late materialisation and operating on compressed data.** The crucial trick: with dictionary
or RLE encoding, many operations run **directly on the encoded representation**. A filter
`country = 'US'` becomes a comparison against one dictionary code; a `GROUP BY` on a
dictionary-encoded column groups integers; an RLE run of 10,000 identical values is aggregated in
one step rather than 10,000. Decompression is deferred until the last possible moment, and often
to a tiny fraction of rows. This is where a large part of the advantage actually comes from, and
it is invisible if you think of compression as purely a storage technique.

**4. Vectorised execution and SIMD.** A column is a contiguous array of one type, which is exactly
what a CPU wants: no per-row function call, predictable branches, cache lines fully utilised,
SIMD-friendly. The Volcano iterator's per-row overhead (L01 §2.4) disappears when `next()` returns
10,000 values.

The factors are roughly multiplicative, which is why the end-to-end difference is so large.

### 2.3 What column stores are bad at

Stated plainly, because the marketing does not:

- **Point lookups by key.** One row means one access per column plus reconstruction. A row store
  does one page read. Column stores can be 10–100× worse here.
- **Single-row inserts.** An insert touches every column's storage. Column stores therefore batch
  writes — into a write-optimised row-store delta that is periodically merged into columnar
  format (Vertica's WOS/ROS split, and essentially the same idea as L02's memtable and SSTables).
- **Updates and deletes.** Usually implemented as delete-vectors plus reinsert, with periodic
  rewriting. Frequent single-row updates are the worst case.
- **Transactions across many rows.** Possible, but the machinery is heavier and most analytical
  systems offer weaker guarantees than an OLTP engine would.
- **`SELECT *`.** Reads every column and pays reconstruction for all of them, giving up the entire
  advantage. Analytics engineers learn quickly not to do this; it is worth knowing *why*.

Hence the standard architecture: **OLTP on a row store, replicated into a column store for
analytics** — which is exactly what L09's change-data-capture pipeline is for, and the reason this
lesson sits where it does in the course.

### 2.4 Making scans skip work

A column store still has to read the column. Three mechanisms reduce even that.

- **Zone maps / min-max statistics.** Store per block the min and max of each column. A predicate
  `WHERE date >= '2024-06-01'` skips every block whose max is below that. This is enormously
  effective *when the data is clustered on the predicate column*, and nearly useless otherwise —
  which makes **data layout a first-class tuning decision**. Sorting or clustering on the common
  filter column can be worth a 10× difference by itself, and it is the single highest-leverage
  physical design choice in an analytical system. (Postgres's BRIN index, L03 §2.2, is the same
  idea.)
- **Partition pruning.** Physically partition by a column (usually date) so the planner eliminates
  whole files or directories. Coarser than zone maps and requires no reading at all.
- **Bloom filters** per block for high-cardinality equality predicates (L02 §2.3, applied again).

Modern lakehouse formats add **multi-dimensional clustering** (Z-ordering, Hilbert curves) to make
zone maps effective for more than one column at a time — with the honest caveat that clustering on
several dimensions is worse for each of them than clustering on one, so this is a compromise, not
a free lunch.

### 2.5 Formats, and the storage/compute split

**Parquet** and **ORC** are PAX-layout files: data in row groups, columns as chunks within each
group, per-chunk encodings and statistics, and a footer holding the schema and metadata. They are
the de facto interchange format for analytical data, and they made the **decoupling of storage
from compute** practical: files live in object storage, engines (Spark, Trino, DuckDB, Snowflake,
BigQuery) read them, and no engine owns the data.

The consequences are architectural:

- **Multiple engines over one copy of the data**, with no export step.
- **Storage and compute scale independently**, which is most of the cost argument for cloud
  warehouses.
- **You lose transactions and consistency**, because a directory of files has no commit protocol.
  This is what the **table formats** — Iceberg, Delta Lake, Hudi — add: an atomically-swapped
  metadata pointer giving snapshot isolation, schema evolution, and time travel over immutable
  files in object storage. Iceberg's commit is a compare-and-swap on a metadata pointer, which is
  DS-701 L08's conditional-write idempotence pattern applied to a table.

Two practical facts about object storage that shape everything here: first-byte latency is tens of
milliseconds (so many small files are disastrous — the "small files problem" is the dominant
performance issue in real lakehouses), and there is no rename or append, so all mutation is
write-new-file plus metadata swap.

### 2.6 The HTAP question

Can one system serve both? Approaches in use:

- **Dual storage in one engine**: row store for the OLTP path, column store kept in sync for the
  analytical path (SQL Server's columnstore indexes, Oracle's In-Memory column store, TiDB's
  TiFlash, SingleStore). Convenient; the sync has a cost and a lag.
- **Separate systems with a pipeline** (L09): the mainstream answer. More moving parts, clean
  separation, and the pipeline is where the correctness problems live.
- **One system, honestly hybrid** (DuckDB for single-node analytics; various distributed
  attempts). Excellent within its scale envelope.

The honest assessment is that HTAP is real but narrower than vendors claim: the workloads want
different physical designs, different isolation, different resource management, and different
failure behaviour. What has genuinely changed is that the *analytical* side got so much faster and
cheaper that a modest analytical workload can now run on the transactional system without a
separate warehouse — which for many organisations removes the pipeline entirely, and removing a
pipeline is worth a great deal.

## 3. Construction: build a column store and measure it

Build in `mpse/di721/l06/`. You will implement enough of a columnar engine to measure each of
§2.2's four factors *separately*, which is the point — the aggregate speedup teaches you nothing,
the decomposition teaches you everything.

**Stage 1 — the dataset and the row baseline.** Generate a wide analytical table: 20+ columns,
several million rows, with a realistic mix — a few low-cardinality strings (country, status), a
timestamp, several integers with small ranges, a couple of high-cardinality columns. Store it
row-wise in a simple binary format and implement three queries: a filtered aggregate, a group-by,
and a point lookup by ID. Measure each, and measure bytes read.

**Stage 2 — columnar layout.** Store each column in its own file, uncompressed. Re-run all three
queries. Report the speedup for the two analytical queries and the *slowdown* for the point
lookup, with bytes read for each. **Factor 1, isolated.**

**Stage 3 — encodings.** Implement RLE, dictionary and bit-packing, choosing per column based on
measured statistics. Report compression ratio per column and overall, and re-run the queries.
**Factor 2, isolated** — and note how much of the improvement is I/O and how much is CPU.

**Stage 4 — operating on encoded data.** Implement the filter and the group-by so that they run
against dictionary codes and RLE runs *without decoding*. Measure against the decode-then-operate
version. **Factor 3, isolated**, and it should be the one that surprises you most.

**Stage 5 — vectorised execution.** Rewrite the execution path to process batches of ~1,000 values
with NumPy rather than row-at-a-time Python. Measure. **Factor 4, isolated.** Then produce the
decomposition table: the contribution of each factor, and their product against the measured
end-to-end speedup. Explain any gap — there will be one, and explaining it is the exercise.

**Stage 6 — zone maps.** Add per-block min/max statistics and predicate-based block skipping.
Measure on data sorted by the predicate column and on the same data shuffled. The difference
between those two numbers is the argument for physical clustering, in your own measurements.

**Stage 7 — Parquet, and a real engine.** Write the same data with `pyarrow` to Parquet and query
it with DuckDB. Compare against your implementation on all three queries. You will lose by a large
factor; identify the three biggest reasons, using DuckDB's `EXPLAIN ANALYZE`. Then vary the row
group size and file count and measure — reproducing the small-files problem deliberately.

**Stage 8 — the pipeline.** Take the row store from L02/L05 and write a batch export into columnar
form, with a `updated_at` watermark so it is incremental. Measure the export cost and the freshness
lag. Then answer, in writing, the question this raises and that L09 will resolve: what happens to
a row deleted from the source, and what happens if the export crashes halfway?

## 4. Failure modes

- **Running analytics on the OLTP database.** The scans evict the buffer pool (L01 §2.3), the long
  transactions pin the MVCC horizon (L05 §2.5), and the query is slow anyway.
- **`SELECT *` against a column store.** Discards the entire advantage and adds reconstruction
  cost.
- **Single-row inserts into a column store.** Batch, or use the write-optimised path the engine
  provides.
- **Ignoring data layout.** Zone maps and partition pruning are worth an order of magnitude and do
  nothing if the data is not clustered on the predicate.
- **The small files problem.** Thousands of tiny Parquet files in object storage turn a scan into
  a latency-bound operation. Compact them; it is routine maintenance, not an optimisation.
- **Over-partitioning.** Partitioning by day when queries span years produces the same problem from
  the other direction.
- **A directory of Parquet files treated as a table.** No atomicity, no schema enforcement, no
  concurrent-write safety. Use a table format.
- **Believing the HTAP pitch without measuring your own workload's mix.**
- **Benchmarking with uniform random data.** Compression ratios and zone-map effectiveness both
  depend entirely on the real distribution, so uniform data makes columnar look worse than it is
  and makes clustering look useless.

## 5. Exercises

### Warm-up (30 min)

1. Do the arithmetic: 80 columns, 120 bytes/row, 50 M rows, a query reading four columns. Bytes
   read row-wise versus column-wise, and the ratio. Then estimate the effect of 8× compression.
2. Name the four factors behind columnar's analytical advantage and say which is invisible if you
   think of compression only as storage saving.
3. Explain PAX and why it is the basis of Parquet rather than pure DSM.

### Core (3.5 h)

4. Complete Stages 1–5 and deliver the factor-decomposition table with the gap explained.
5. Complete Stage 6 and deliver the sorted-versus-shuffled comparison.
6. Complete Stage 7, including the three reasons DuckDB beats your implementation and the
   small-files measurement.
7. For an analytical workload you know: determine the storage layout in use, the clustering, and
   the partitioning; then identify one physical design change and predict its effect before
   measuring it.

### Challenge

8. Complete Stage 8 and write the answers to its two questions, then check them against L09 after
   you have read it.
9. Implement **late materialisation** properly: run the entire filter and aggregation pipeline on
   position lists (selection vectors) and dictionary codes, materialising actual values only for
   the final output. Compare against early materialisation across a range of selectivities and plot
   both curves. Identify the selectivity at which late materialisation stops paying, and explain
   the crossover — this is the same shape of argument as L03's index/scan crossover, and noticing
   that is part of the exercise.

## 6. Self-check

1. Give the arithmetic for the projection advantage and the typical factor.
2. Name five columnar encodings and the data shape each suits.
3. What does "operating on compressed data" mean and why is it a large part of the win?
4. Why is a column a good fit for vectorised and SIMD execution?
5. Give four things column stores are bad at, with the mechanism for each.
6. What is PAX and which formats use it?
7. What are zone maps and what single property determines whether they help?
8. What do table formats add over a directory of Parquet files, and what mechanism do they use to
   commit?
9. Why is the small files problem specifically an object-storage problem?
10. State the honest case for and against HTAP.

## 7. Primary sources

- **Abadi, Boncz & Harizopoulos, "The Design and Implementation of Modern Column-Oriented Database
  Systems" (Foundations and Trends in Databases, 2013)** — the definitive survey; read the whole
  thing.
- **Stonebraker et al., "C-Store: A Column-oriented DBMS" (VLDB 2005)** and Lamb et al., "The
  Vertica Analytic Database" (VLDB 2012) — the research system and its production successor.
- Abadi, Madden & Ferreira, "Integrating Compression and Execution in Column-Oriented Database
  Systems" (SIGMOD 2006) — operating on compressed data.
- Boncz, Zukowski & Nes, "MonetDB/X100: Hyper-Pipelining Query Execution" (CIDR 2005) — where
  vectorised execution comes from.
- Ailamaki et al., "Weaving Relations for Cache Performance" (VLDB 2001) — PAX.
- Melnik et al., "Dremel: Interactive Analysis of Web-Scale Datasets" (VLDB 2010) — nested
  columnar storage; the ancestor of Parquet's repetition/definition levels.
- Armbrust et al., "Delta Lake" (VLDB 2020) and "Lakehouse" (CIDR 2021); the Apache Iceberg
  specification.
- Raasveldt & Mühleisen, "DuckDB: an Embeddable Analytical Database" (SIGMOD 2019).
- Kleppmann, *Designing Data-Intensive Applications*, ch. 3 (analytical section).

---

**Previous:** [L05](L05-mvcc-and-serializability.md) · **Next:**
[L07 — Batch Processing and Dataflow](L07-batch-processing.md)
