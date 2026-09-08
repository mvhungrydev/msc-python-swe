# DI-721 — Written Examination

**Time allowed: 3 hours. Closed book, no machine.**
**Answer FOUR from Section A and ONE from Section B.**
Section A: 15 marks each. Section B: 40 marks.

Where a question asks for arithmetic, show it. Where it asks for a guarantee, state what holds,
for which operations, under which conditions. Where it asks you to construct an interleaving or a
schedule, draw it explicitly — prose descriptions score at most half marks.

---

## Section A — answer FOUR

**A1.** Name the five components of a database architecture and say which one a query-planning
problem, a storage problem and an overload problem each live in. Then explain, with the memory
hierarchy ratios, why the unit of I/O is a page and what that implies for tree fan-out. *(15)*

**A2.** State the external memory model and give the cost of a B-tree search and of sorting within
it. Explain what changed relative to the RAM model and why changing the cost model changes which
algorithm wins. *(15)*

**A3.** Explain what a buffer pool does that the OS page cache cannot. Then explain why LRU is
defeated by a sequential scan, name a policy that resists it, and state the general principle about
cache policies that this illustrates. *(15)*

**A4.** Describe the path from SQL text to an executed plan. Then state the single most common
cause of catastrophically bad plans, explain why correlated predicates are systematically
mis-estimated, and say why estimation errors are worse in a deep join tree than a shallow one.
*(15)*

**A5.** State the write-ahead logging rule and explain why it is a performance win as well as a
correctness one. Then explain why `fsync`, not `write`, is the durability boundary, and give one
historical example of that being misunderstood. *(15)*

**A6.** Define write, read and space amplification precisely, with an example of each. State the
RUM conjecture and place levelled LSM, size-tiered LSM and B-trees on it. *(15)*

**A7.** Give the LSM write path and read path, and explain the three mechanisms that reduce read
amplification. Explain why bloom filters help point lookups but not range scans, and what follows
for a scan-heavy workload. *(15)*

**A8.** Compare size-tiered and levelled compaction across all three amplifications. Then define a
write stall, explain its cause in queueing terms, and give the correct operational response. *(15)*

**A9.** Explain why deletion is hard in an LSM. Describe tombstones, state the condition under which
one may safely be dropped, and explain the resurrection bug that occurs if it is dropped early.
Then explain why an insert-then-delete queue workload is the LSM's worst case. *(15)*

**A10.** Distinguish clustered from secondary indexes and say what a secondary lookup costs under
each design. Then define a covering index and explain why an index-only scan is so much faster, and
one reason it can silently stop being used. *(15)*

**A11.** State the leftmost prefix rule and justify it from the sort order. Explain the column-order
rule for a composite index, and explain precisely why a range predicate on the second column makes
the third unusable for filtering. *(15)*

**A12.** Define selectivity and state the index/scan crossover rule with its justification from the
memory hierarchy. Then explain parameter sniffing, what it looks like in a bug report, and why
skewed data makes a single cached plan wrong. *(15)*

**A13.** Give five situations in which adding an index makes a system worse, with the mechanism for
each. Include the sargability cases and be specific about implicit type coercion. *(15)*

**A14.** State what each ACID letter actually claims, identifying the one that is partly the
application's responsibility. Then give the ANSI isolation table and three specific ways real
engines deviate from it. *(15)*

**A15.** Define dirty read, non-repeatable read, phantom, lost update, read skew and write skew,
each with an explicit two-transaction interleaving. Explain what distinguishes a phantom from a
non-repeatable read and why the distinction changes the required locking mechanism. *(15)*

**A16.** Describe snapshot isolation including first-committer-wins. Then present the doctors-on-call
write skew, explain precisely why first-committer-wins does not prevent it, and state the general
shape of the anomaly. *(15)*

**A17.** Give five ways to prevent an anomaly your isolation level permits, in order of preference,
with the situation each suits. Explain what "materialising the conflict" means and when it is
necessary. *(15)*

**A18.** State the MVCC visibility rule using `xmin`, `xmax` and a snapshot, and use it to explain
why an `UPDATE` is a delete plus an insert. Then compare append-in-place and undo-log MVCC, naming
the characteristic operational failure of each. *(15)*

**A19.** Define the three dependency edge types and state the conflict-serializability theorem.
Then state what structure every non-serializable snapshot-isolation execution contains, and explain
how SSI uses it. *(15)*

**A20.** Explain how one idle transaction can degrade an entire database, following the chain from
snapshot to horizon to bloat. Then give the five operational practices that prevent it, and explain
what transaction ID wraparound protects against. *(15)*

**A21.** Give the arithmetic for columnar's projection advantage on a wide table. Then name all four
factors behind the analytical advantage, and explain the one that is invisible if you think of
compression purely as a storage saving. *(15)*

**A22.** Name five columnar encodings and the data shape each suits. Explain what "operating on
compressed data" means with a concrete example for two of them, and why it saves CPU as well as
I/O. *(15)*

**A23.** Give four things column stores are bad at, with the mechanism for each. Then explain PAX,
why Parquet uses it, and what a table format adds over a directory of Parquet files. *(15)*

**A24.** Explain zone maps and state the single property that determines whether they help. Then
explain the small-files problem, why it is specific to object storage, and the standard remedy.
*(15)*

**A25.** State batch processing's reliability model in one sentence, and name three things that
break it. Then explain why the shuffle is the whole job and what sorting achieves within it. *(15)*

**A26.** Define a combiner and the algebraic condition it requires. Classify `SUM`, `AVG`, `MAX`,
`MEDIAN` and `COUNT DISTINCT` by it, and say what you do about the ones that fail. *(15)*

**A27.** Describe data skew in a distributed join: its cause, its unmistakable symptom, and five
remedies with the cost of each. Then state the general principle about partitioning that it
illustrates. *(15)*

**A28.** Distinguish event time from processing time and explain why the choice determines
reproducibility. Define a watermark, state that it is a heuristic, and give the failure in each
direction. *(15)*

**A29.** Give the three accumulation modes and the sink semantics each requires. Then describe the
failure that occurs when accumulating panes are emitted to an adding consumer, and say why nothing
raises an error. *(15)*

**A30.** State precisely what "exactly-once" means in a stream processor, describe the mechanism
(replayable source, checkpointed state, transactional or idempotent sink), and identify exactly
where the guarantee stops. *(15)*

**A31.** Give three CDC implementations and state all three failure modes of the polling approach,
being precise about the commit-order case. Then explain how log-based CDC can take down a production
database. *(15)*

**A32.** Compare log-based CDC with the transactional outbox across event shape, coupling and schema
migration. State the rule of thumb for choosing, and justify it in terms of information hiding.
*(15)*

**A33.** Define event sourcing and distinguish it from CQRS. Give six costs, and state the honest
test for whether a domain warrants it. *(15)*

**A34.** State the stream–table duality and use it to explain log compaction, database replication
and cache invalidation as one idea. Then state what changes about the failure mode when a cache
becomes a materialised view. *(15)*

**A35.** Define backward, forward and full compatibility, saying who can be upgraded first under
each. Give five schema changes classified under Avro, Protobuf and unvalidated JSON, and explain why
the first two can evolve where the third cannot. *(15)*

**A36.** Give the six steps of expand/migrate/contract and explain why each is individually
reversible. Then explain why event logs cannot be migrated this way and give the three alternatives.
*(15)*

**A37.** Distinguish Type 1, 2 and 3 slowly changing dimensions. Give the failure of Type 1 with a
concrete example, and connect it to the temporal-correctness problem in stream–table joins. Then
define grain and say what mixing it produces. *(15)*

---

## Section B — answer ONE

**B1. Choose the engine, and prove it.** A system ingests 200,000 events per second, each a small
record keyed by device ID. The two query patterns are: a point lookup of a device's latest state
(99% of queries), and a scan of one device's history over a time range (1%). Retention is 90 days;
the dataset is roughly 40× available RAM. Space is budgeted and tight.

Design the storage layer. Your answer must: choose between B-tree and LSM with the amplification
arithmetic; choose a compaction strategy and defend it in RUM terms; design the key so that both
query patterns are served, explaining what the key ordering buys; state your bloom filter
configuration and what it costs in memory; explain what happens when ingest exceeds compaction
throughput and what you do about it; explain how the 90-day retention is implemented and why the
naive delete-based implementation is wrong; and state the one workload change that would invalidate
your design. *(40)*

**B2. The concurrency bug.** A payments system running at READ COMMITTED processes refunds. The
logic reads the order's total refunded amount, checks that a new refund would not exceed the order
total, and inserts a refund row. In production, some orders have been refunded for more than their
total. Under test, this has never reproduced.

Write the analysis and the fix. Your answer must: name the anomaly precisely and construct the
exact interleaving that produces it, drawn as a timeline; explain why it does not reproduce under
single-threaded testing and what test harness would catch it; explain whether raising to REPEATABLE
READ / snapshot isolation would fix it, with reasoning about which rows each transaction reads and
writes; give three fixes at different levels — atomic statement, explicit locking, and isolation
level — with the trade-offs of each; explain what the application must do if you choose
SERIALIZABLE; and state what monitoring would have detected the problem before customers did. *(40)*

**B3. The pipeline.** An OLTP system on Postgres must feed: a search index (freshness within
seconds), an analytical warehouse (freshness within an hour), a cache (freshness within seconds),
and a partner's system (a published contract, freshness within a day).

Design the data movement. Your answer must: choose CDC, outbox, or both, per consumer, and justify
each choice — being explicit about which consumers should couple to your schema and which must not;
explain the dual-write problem and where it would otherwise appear; describe the bootstrap of a new
consumer including the snapshot-to-stream handoff; state the delivery guarantee at each hop and how
effectively-once is achieved at each sink; explain what happens when the warehouse consumer is down
for six hours, including the effect on the source database; describe the schema-change process for
the partner contract, including compatibility mode and deprecation; and identify the two metrics you
would alert on. *(40)*

**B4. Correct results from late data.** A billing system computes hourly usage charges per customer
from a stream of metering events. Events are typically seconds late, occasionally hours late, and a
small fraction arrive days late after a device reconnects. Bills are issued monthly and must be
correct; customers see a running estimate in a dashboard that must update within a minute.

Design it. Your answer must: answer the Dataflow model's four questions explicitly; choose event
time or processing time for each consumer and justify each; specify the watermark policy and the
allowed lateness, with the reasoning about what data that discards and why it is acceptable —
noting that "acceptable" differs between the dashboard and the bill; specify the accumulation mode
for each sink and what each sink must support; explain what happens to data that arrives after the
allowed lateness, and why silently dropping it is not an option in a billing system; describe the
state size implications and how state is bounded; and describe how you would reprocess a month of
history after finding a bug in the pricing logic, including how you would verify the new results
before switching customers onto them. *(40)*

**B5. The breaking change.** A team owns a `customers` table consumed by fourteen downstream
artifacts: dashboards, ML feature pipelines, a partner export and several services. They need to
split `name` into `given_name` and `family_name`, change `balance` from dollars to cents, and drop
a column they believe is unused.

Write the plan. Your answer must: treat the three changes separately, because they are not the same
kind of change — say why for each; give the expand/migrate/contract sequence for the split with the
reversibility of each step; explain what makes the units change categorically more dangerous than
the split, and how you would prevent silent misinterpretation (the answer is not "tell people");
describe how you would establish whether the third column is genuinely unused, and the class of
dependency you cannot detect technically; state what should have existed beforehand to make all
three changes routine — schema registry, contracts, lineage, ownership — and what each would have
contributed; and give the deprecation policy you would adopt going forward, with the enforcement
point that makes it real rather than aspirational. *(40)*

---

*Marks in Section A are awarded for precision and for arithmetic where arithmetic is asked for. A
mechanism described correctly but without its conditions scores partial marks. In Section B, an
answer that does not identify what it traded away, and a condition under which its design fails,
cannot reach the upper band, however sound the design.*
