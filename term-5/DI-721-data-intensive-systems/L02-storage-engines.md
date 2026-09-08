# DI-721 · Lesson 02 — Storage Engines: B-Trees and LSM-Trees

**Estimated study time:** 5 hours
**Prerequisites:** L01; CS-621 L01 (asymptotics), PY-602 L06 (caches and locality)

---

## 1. Orientation

There are two dominant designs for durable, ordered key-value storage, and essentially every
database you will use is one of them or a hybrid.

**B-trees** update data **in place**. A key lives at one address; changing its value means
finding that page, modifying it, and writing it back. Postgres, MySQL/InnoDB, Oracle, SQL Server,
SQLite, and most B-tree-based embedded stores work this way.

**LSM-trees** never update in place. Writes go to an in-memory structure and are periodically
flushed as **immutable sorted files**; a key's history accumulates across files, and background
**compaction** merges them, discarding superseded versions. RocksDB, LevelDB, Cassandra,
ScyllaDB, HBase, and the write path of many newer systems work this way.

The reason to study both properly is that the choice between them is the single most consequential
storage decision you can make, and it is decided by a small number of measurable quantities. This
lesson gives you those quantities, and then you write an LSM-tree yourself — because the tuning
knobs of RocksDB are unreadable until you have implemented the thing they tune.

The frame that makes the whole design space legible is the **RUM conjecture** (Athanassoulis et
al., 2016):

> You can optimise for at most two of **read** overhead, **update** overhead, and **memory
> (space)** overhead. Improving two worsens the third.

Every knob in every storage engine is a point on that surface. Once you see it, tuning stops
being folklore.

## 2. Theory

### 2.1 The three amplifications

State these precisely, because loose usage causes real confusion:

- **Write amplification** = bytes written to storage ÷ bytes of logical data written. An LSM
  rewrites data during compaction, so a 1 KB user write may become 10–30 KB of device writes. It
  matters for throughput and for SSD lifetime.
- **Read amplification** = storage reads performed ÷ logical reads requested. An LSM may consult
  several files to answer one lookup.
- **Space amplification** = bytes on storage ÷ bytes of live logical data. B-trees waste space in
  partially full pages and fragmentation; LSMs hold obsolete versions until compaction removes
  them.

Every engine trades these against each other, and *the right trade is a property of your workload,
not of the engine*. That sentence is the whole lesson in one line.

### 2.2 B-trees

A **B⁺-tree**: internal nodes hold keys and child pointers only, leaves hold all the data and are
linked for range scans. Node size equals page size, so fan-out is in the hundreds and the depth
of a tree over a billion keys is 3–4. A lookup is that many page reads, and the upper levels are
almost always resident in the buffer pool, so in practice it is one or two device reads.

**Insertion** places a key in a leaf; if the leaf is full it **splits**, propagating a key upward,
possibly recursively to the root (which is how the tree grows in height — from the top, which is
why it stays balanced). **Deletion** in theory merges underfull nodes; in practice most engines
just mark space free, because merging is expensive and the space is usually reused.

Where the real complexity lives:

- **Crash safety.** A split writes several pages, and a crash in the middle leaves a corrupt tree.
  Hence WAL (L01 §2.5), and in some designs **torn-page protection** — full-page writes to the log
  the first time a page is touched after a checkpoint (Postgres's `full_page_writes`, which is a
  significant part of its WAL volume).
- **Concurrency.** Latching a whole path kills throughput. The classic solution is **latch
  crabbing** (hold a child's latch, release the parent's once the child is known safe);
  the modern one is **B-link trees** (Lehman and Yao, 1981), which add a right-link to each node so
  a reader that arrives during a split can follow the link instead of blocking. Essentially every
  production B-tree is a B-link tree.
- **Fragmentation.** Random-order insertion leaves pages roughly 69% full on average — the
  classic B-tree occupancy result. Sequential insertion (an auto-increment key) fills pages
  completely, which is one concrete reason monotonic keys behave differently from random ones.
- **Right-edge contention.** With a monotonically increasing key, every insert hits the same
  rightmost leaf, and that page becomes the bottleneck under concurrency. This is a real,
  frequently-encountered scaling limit of auto-increment primary keys.

B-tree profile: **excellent reads** (one lookup path, predictable), **moderate space**, **poor
write amplification for random writes** (a 100-byte update writes an entire page, and possibly
writes it to the log twice).

### 2.3 LSM-trees

The write path:

1. Append to the **write-ahead log** (sequential, durable).
2. Insert into the **memtable** — an in-memory sorted structure (skip list or balanced tree).
3. When the memtable exceeds a threshold, make it immutable and flush it as an **SSTable**:
   an immutable file of sorted key-value pairs with a sparse index, and usually a bloom filter.
4. **Compaction** merges SSTables in the background, discarding superseded versions and
   tombstones.

The read path must consult the memtable, then the SSTables newest-first, stopping at the first
occurrence of the key. That is the read amplification, and three mechanisms reduce it:

- **Bloom filters** — a probabilistic set membership test with no false negatives. A negative
  answer means "definitely not in this file", so a lookup skips the file entirely. With ~10 bits
  per key the false-positive rate is about 1%, which turns "check 8 files" into "read from 1 file
  plus 0.07 expected wasted reads". Bloom filters are what make LSM point lookups viable at all,
  and note that they help point lookups but **not range scans**, which must merge across files
  regardless. That asymmetry is a real design consideration.
- **Sparse index and block cache** — one index entry per block, so a file lookup is one binary
  search in memory plus one block read.
- **Compaction itself** — fewer files means fewer probes.

**Compaction strategies** are where the RUM trade becomes a dial:

| Strategy | Write amp | Read amp | Space amp | Fits |
|---|---|---|---|---|
| **Size-tiered** | Low | High | High (~2× or worse) | Write-heavy, append-mostly |
| **Levelled** | High (~10–30×) | Low | Low (~1.1×) | Read-heavy, space-constrained |
| **Hybrid / tiered+levelled** | Middle | Middle | Middle | The usual production answer |

Levelled compaction keeps each level's files non-overlapping and each level ~10× the previous, so
a lookup checks at most one file per level; the cost is that data is rewritten roughly once per
level it descends. Size-tiered merges files of similar size, writing much less — but leaving
several overlapping files per level, so lookups probe more and obsolete data lingers.

The pathology to know by name: **write stalls**. If ingest outruns compaction, the number of L0
files grows, read amplification climbs, and eventually the engine throttles or blocks writers to
let compaction catch up. Latency goes from microseconds to seconds with no change in the
workload's *average* rate — it is a queueing failure (PY-601 L09), and it is the most common
operational problem with LSM stores.

LSM profile: **excellent write throughput** (all sequential), **tunable read cost**, **background
CPU and I/O for compaction**, **latency spikes** if compaction falls behind.

### 2.4 Deletion, and why it is genuinely hard

An LSM cannot delete in place, so a delete writes a **tombstone**: a marker that this key is
gone. The tombstone must persist until it has been compacted against *every* older SSTable that
could contain the key — otherwise the old value resurfaces. Consequences:

- Deleting data **increases** storage temporarily.
- A range scan over a heavily-deleted range reads every tombstone, so a queue table implemented
  as "insert then delete" degrades badly. Cassandra users know this as the "tombstone problem",
  and it is the canonical example of an access pattern fighting the storage engine.
- In a distributed store, tombstones interact with the CRDT-style resurrection problem from
  DS-701 L07: a tombstone garbage-collected too early lets a delayed replica reintroduce the
  deleted row.

B-trees have the mirror-image problem: deleted space is reclaimed within a page but the page is
not returned to the OS, so a table that shrank still occupies its old space until it is rewritten
(`VACUUM FULL`, `OPTIMIZE TABLE`), which requires a full rewrite and usually a lock.

### 2.5 Choosing

Reason from the workload, in this order:

1. **Write rate relative to device capability.** If random-write throughput is the binding
   constraint, LSM. If it is not, this argument does not apply.
2. **Read pattern.** Point lookups on an LSM with bloom filters are competitive. *Range scans* are
   where B-trees keep a real edge, because an LSM range scan must merge across levels and cannot
   use bloom filters.
3. **Update pattern.** Frequent updates to the same keys create garbage the LSM must compact away;
   blind writes and appends are the LSM's best case.
4. **Space budget.** Levelled LSM has the best space amplification of anything here; size-tiered
   the worst.
5. **Latency requirements.** If a p99.9 spike from a compaction stall is unacceptable and you
   cannot provision headroom for compaction, that is an argument for a B-tree.
6. **Operational familiarity.** LSMs have more tuning surface, and a badly tuned LSM is worse than
   a default B-tree.

Two notes that keep the picture honest. **Modern designs blur the line**: Postgres's heap plus
WAL is not a pure B-tree story, WiredTiger offers both, and "**B**ε**-trees**" (used by
TokuDB/Percona and by some filesystems) buffer updates in internal nodes to get LSM-like write
amplification with B-tree-like reads — the RUM surface is continuous, not two points. And **the
fastest storage engine is often no storage engine**: if the working set fits in memory and
durability is provided by replication plus a log, the whole comparison changes.

## 3. Construction: write an LSM-tree

This is the largest construction in the course and the foundation of the term artifact. Build in
`mpse/di721/l02/`. Python's speed is irrelevant here — you are measuring amplification ratios and
algorithmic behaviour, both of which are language-independent.

**Stage 1 — the interface and the honest baseline.** Define `get(key)`, `put(key, value)`,
`delete(key)`, `scan(start, end)`. Implement it first as a dict serialised to a file on every
write. Benchmark it. This is your control: every later measurement is a ratio against something,
and you will need to know what "obviously bad" looks like.

**Stage 2 — memtable and WAL.** A sorted-dict memtable plus an append-only log with a length
prefix and a CRC per record. Recovery replays the log into the memtable on open. Test: write
1,000 records, `SIGKILL` the process, reopen, verify every acknowledged write is present. Then
corrupt a byte in the middle of the log and verify that recovery stops cleanly at the corruption
rather than reading garbage — a log format without checksums is not a log format.

**Stage 3 — SSTables.** Flush the memtable when it exceeds a threshold: a file of sorted
key-value pairs written in blocks, a sparse index (one entry per block) in a footer, and a
metadata header with the key range and record count. Implement `get` as: memtable, then SSTables
newest-first. Measure read latency as the number of SSTables grows from 1 to 50, and plot it.
That curve is read amplification, drawn from your own system.

**Stage 4 — bloom filters.** Implement one properly: choose m and k from the target false-positive
rate and expected key count (derive the formulas rather than copying them), use double hashing
from one 64-bit hash. Attach one per SSTable. Re-run Stage 3's measurement and overlay the curves.
Then *measure the actual false-positive rate* against the theoretical one — if they disagree, your
hash functions are correlated, which is the classic bug.

**Stage 5 — size-tiered compaction.** Merge SSTables of similar size, dropping superseded keys.
Instrument bytes written by the user versus bytes written to disk, and report write amplification.
Track space amplification over a workload of overwrites.

**Stage 6 — levelled compaction.** L0 files may overlap; L1+ are non-overlapping with a 10× size
ratio. Implement the picker (which file to compact, and which overlapping files in the next level
it must be merged with). Re-run the same workload and produce the comparison table: write amp,
read amp, space amp for both strategies. **This table is the deliverable of the whole lesson** —
it is the RUM conjecture, measured by you.

**Stage 7 — tombstones.** Deletes write tombstones; compaction drops them only when compacting
against the oldest level. Demonstrate the resurrection bug by dropping a tombstone too early, then
fix it. Then build the pathological queue workload (insert, then delete, then range-scan) and plot
scan latency as tombstones accumulate.

**Stage 8 — write stalls.** Add a compaction thread with a bounded rate. Drive ingest above what
compaction can sustain and plot write latency over time. Find the point where L0 file count
triggers your throttle. Then implement the throttle properly — gradual backpressure rather than a
cliff — and show the difference in the latency distribution. This connects directly to PY-601 L09
and DS-701 L09, and it is the same phenomenon in a third setting.

**Stage 9 — the comparison.** Benchmark your LSM against SQLite (a B-tree) on four workloads:
random-write-heavy, read-heavy point lookups, range-scan-heavy, and update-heavy on a hot key
subset. Yours will be far slower in absolute terms; report the *shapes*, not the absolute numbers,
and explain each shape from the mechanisms above. An explanation that does not predict the shape
before you measure it is a rationalisation.

## 4. Failure modes

- **A log without checksums.** Silent corruption on recovery, discovered much later.
- **Acknowledging a write before the log record is durable.** The `fsync` boundary is the
  durability boundary (L01 §2.5); if you ack before it, your durability claim is false.
- **Correlated hash functions in a bloom filter.** The measured false-positive rate is far worse
  than the theoretical one, and nothing crashes — you just do more I/O forever.
- **Compaction that cannot keep up, with no backpressure.** Latency cliffs and, eventually, an
  engine that stops accepting writes.
- **Dropping tombstones too early.** Deleted data returns. This is a correctness bug that testing
  rarely finds by accident, so test for it deliberately.
- **A queue implemented as an LSM table.** Insert-then-delete against a range scan is the
  storage engine's worst case; use a queue.
- **Monotonic keys into a B-tree under high concurrency.** Right-edge contention.
- **Benchmarking with a uniform random key distribution** when production is Zipfian. Cache hit
  rates, compaction behaviour and bloom filter effectiveness all change with the distribution, so
  the benchmark measures a system you do not have.
- **Tuning RocksDB by copying a configuration.** Each knob is a RUM trade; if you cannot say which
  two you chose, you did not choose.

## 5. Exercises

### Warm-up (30 min)

1. Define the three amplifications precisely and state the RUM conjecture. For each of levelled
   LSM, size-tiered LSM and B-tree, say which two of the three are being favoured.
2. Derive the bloom filter's optimal number of hash functions k for given m and n, and compute the
   bits per key needed for a 1% false-positive rate.
3. Explain why bloom filters help point lookups but not range scans, and what follows for a
   scan-heavy workload on an LSM.

### Core (3.5 h)

4. Complete Stages 1–4. Deliver the read-amplification curve with and without bloom filters, and
   the measured-versus-theoretical false-positive rate.
5. Complete Stages 5–6 and deliver the comparison table. Write 500 words interpreting it in RUM
   terms.
6. Complete Stage 7, including the deliberately-introduced resurrection bug, its fix, and the
   tombstone scan-degradation plot.
7. For a system you work with, determine which storage engine it uses and write 400 words on
   whether the workload matches the engine's profile — with at least one number you measured
   rather than assumed.

### Challenge

8. Complete Stages 8–9 with the full four-workload comparison and the shape explanations
   (predictions written *before* the measurements).
9. Implement **partitioned bloom filters or a prefix bloom filter** to accelerate a common range
   pattern, and quantify the improvement. Then implement a **Bε-tree-style buffered B-tree** —
   a B-tree whose internal nodes buffer pending updates, flushing them down in batches — and add
   it to the Stage 9 comparison as a third engine. Report where it lands on the RUM surface
   relative to your other two, and whether it delivered what the literature claims.

## 6. Self-check

1. Define write, read and space amplification, and give a concrete example of each.
2. State the RUM conjecture and place three engines on it.
3. Why is B-tree fan-out in the hundreds, and what is the depth over 10⁹ keys?
4. What is a B-link tree and what problem does the right-link solve?
5. Give the LSM write path in four steps and the read path with its three mitigations.
6. Compare size-tiered and levelled compaction across all three amplifications.
7. What is a write stall, what causes it, and what is the correct response?
8. Why can a tombstone not be dropped as soon as the delete is compacted once?
9. Why does an insert-then-delete queue workload behave badly on an LSM?
10. Give six questions that decide between a B-tree and an LSM for a given workload.

## 7. Primary sources

- **O'Neil, Cheng, Gawlick & O'Neil, "The Log-Structured Merge-Tree" (Acta Informatica, 1996)** —
  the original.
- **Athanassoulis et al., "Designing Access Methods: The RUM Conjecture" (EDBT 2016)** — short,
  and the best organising frame available.
- Lehman & Yao, "Efficient Locking for Concurrent Operations on B-Trees" (TODS 1981) — B-link
  trees.
- Graefe, "Modern B-Tree Techniques" (FnTDB, 2011) — everything B-trees learned after 1990.
- Dong et al., "Optimizing Space Amplification in RocksDB" (CIDR 2017) — the practitioners'
  account of the trade-offs, with production numbers.
- Dayan, Athanassoulis & Idreos, "Monkey: Optimal Navigable Key-Value Store" (SIGMOD 2017) —
  optimal bloom filter allocation across levels; a beautiful result.
- Bender et al., "An Introduction to Bε-trees and Write-Optimization" (;login:, 2015).
- Petrov, *Database Internals*, Part I.
- Kleppmann, *Designing Data-Intensive Applications*, ch. 3.
- The RocksDB wiki's compaction and tuning pages — read them *after* Stage 6, when they will
  finally make sense.

---

**Previous:** [L01](L01-what-a-database-does.md) · **Next:**
[L03 — Indexing, and When It Hurts](L03-indexing.md)
