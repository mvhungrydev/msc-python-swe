# DI-721 — Data-Intensive Systems

**Term:** 5 · **Credits:** 15 · **Nominal hours:** 140
**Prerequisites:** PY-602, SE-521, CS-621
**Co-requisite:** DS-701

---

## Driving question

> How does data keep its meaning as it moves?

Every interesting system moves data: from a request into a table, from a table into an index,
from a database into a warehouse, from a stream into a model, from one team's schema into
another team's assumptions. At each hop, meaning can be lost — silently, and usually much
earlier than anyone notices.

This course is about the machinery that moves data and the guarantees each piece of it actually
provides. It has two halves that look separate and are not:

- **Storage** — what a database does on disk, and why. B-trees and LSM-trees, indexes,
  transactions and isolation, MVCC, column layouts. The theme is that every storage design is a
  bet about the access pattern, and knowing the bet is what lets you predict the failure.
- **Movement** — batch and stream processing, delivery semantics, event sourcing, change data
  capture, schema evolution. The theme is that a pipeline's correctness is mostly a property of
  its *contracts*, not its code.

DS-701 asks what you can guarantee when parts of your system are failing. DI-721 asks what a
piece of data means once it has passed through four systems written by three teams, and what it
costs to keep that answer stable. The two courses share a build artifact for exactly this
reason.

The unifying discipline: **a data system's honest description is the set of guarantees it makes
about the data leaving it, not the set of features it offers.** Most data incidents are a
mismatch between an assumed guarantee and an actual one.

## Learning outcomes

On completion you will be able to:

1. **Explain** how B-tree and LSM-tree storage engines work, and predict which is right for a
   given read/write mix — with the write and read amplification arithmetic to back it.
2. **Design** indexes for a workload and account for their cost, including the cases where an
   index makes things worse.
3. **State** the isolation levels precisely, name the anomaly each permits, and construct a
   demonstration of each.
4. **Explain** MVCC and snapshot isolation, including write skew and why serializable snapshot
   isolation is different.
5. **Distinguish** row and column storage and explain, quantitatively, why analytical queries
   need the latter.
6. **Compare** batch and stream processing models, and reason about event time, watermarks and
   windowing.
7. **Reason** correctly about delivery semantics, and build effectively-once pipelines from
   at-least-once parts.
8. **Design** event-sourced systems and change-data-capture pipelines, including the outbox and
   its failure modes.
9. **Evolve** a schema without breaking consumers, and state a compatibility contract.
10. **Model** data for analytical use, and defend the modelling choice against the alternatives.

## Lessons

| # | Title | Hours |
|---|---|---|
| L01 | What a Database Actually Does | 4 |
| L02 | Storage Engines: B-Trees and LSM-Trees | 5 |
| L03 | Indexing, and When It Hurts | 4.5 |
| L04 | Transactions and Isolation Levels | 5 |
| L05 | MVCC, Snapshots, and Serializability | 5 |
| L06 | Column Stores and Analytical Processing | 4.5 |
| L07 | Batch Processing and Dataflow | 4 |
| L08 | Stream Processing, Event Time, and Windows | 5 |
| L09 | Event Sourcing, CDC, and the Log as Truth | 4.5 |
| L10 | Schema Evolution, Contracts, and Data Modelling | 4.5 |

## Assessment

| Component | Weight |
|---|---|
| Problem set 1 (L01–L03): a storage engine you wrote, benchmarked honestly | 20% |
| Problem set 2 (L04–L06): transactions, anomalies demonstrated, and a column store | 20% |
| Problem set 3 (L07–L10): a pipeline that is correct under failure and evolvable | 20% |
| Written exam | 25% |
| Course position paper (1,500 words) | 15% |

## Term 5 build artifact (with DS-701)

DS-701's artifact is a replicated key-value store. DI-721 supplies **what is underneath and
what flows out of it**:

- A **storage engine you wrote** — an LSM-tree with a memtable, SSTables, a write-ahead log,
  compaction and a bloom filter — used as the store's local persistence, with a benchmark that
  reports write amplification, read amplification and space amplification honestly.
- A **transactional layer** with a stated isolation level, and a test suite that demonstrates
  which anomalies it permits and which it prevents.
- A **change-data-capture pipeline** out of the store: an outbox or log tail feeding a stream
  processor that maintains a derived view.
- A **stream processor** with event-time windowing and watermarks, producing correct results
  under out-of-order and late-arriving data.
- A **schema registry and compatibility check** in CI, such that an incompatible change to the
  event schema fails the build.
- An **analytical view** of the same data in columnar form, with a measured comparison against
  the row store for two representative queries.

Together with DS-701's replication, consensus and Jepsen-style testing, this is a complete
data system with defensible guarantees at every layer. That is the term's real deliverable: not
a program, but a set of claims you can support.

## Required reading

- **Kleppmann, *Designing Data-Intensive Applications*, chs. 3, 4, 7, 10, 11, 12.** (Chs. 5, 8,
  9 belong to DS-701.) This book is the spine of the course.
- **Hellerstein, Stonebraker & Hamilton, "Architecture of a Database System" (FnTDB, 2007)** —
  the best single overview of what is inside a database.
- **O'Neil, Cheng, Gawlick & O'Neil, "The Log-Structured Merge-Tree" (Acta Informatica, 1996).**
- **Berenson, Bernstein, Gray, Melton, O'Neil & O'Neil, "A Critique of ANSI SQL Isolation
  Levels" (SIGMOD 1995)** — read this before you trust any isolation-level documentation.
- Fekete, Liarokapis, O'Neil, O'Neil & Shasha, "Making Snapshot Isolation Serializable"
  (TODS 2005).
- Abadi, Boncz & Harizopoulos, "The Design and Implementation of Modern Column-Oriented
  Database Systems" (FnTDB, 2013).
- **Akidau et al., "The Dataflow Model" (VLDB 2015)** — the paper that settled how to think
  about event time.
- Kleppmann, "Turning the Database Inside-Out" and Kreps, "The Log: What Every Software
  Engineer Should Know About Real-Time Data's Unifying Abstraction".

## Recommended

- Petrov, *Database Internals* — the storage-engine detail this course compresses.
- Graefe, "Modern B-Tree Techniques" (FnTDB, 2011).
- Dong et al., "Optimizing Space Amplification in RocksDB" (CIDR 2017).
- Akidau, Chernyak & Lax, *Streaming Systems*.
- Kimball & Ross, *The Data Warehouse Toolkit* — dated in its infrastructure, still correct in
  its modelling.
- Armbrust et al., "Lakehouse: A New Generation of Open Platforms…" (CIDR 2021).
- Stonebraker & Çetintemel, "'One Size Fits All': An Idea Whose Time Has Come and Gone"
  (ICDE 2005).

## A note on scope

This course teaches you to *build* small versions of these systems, not to operate large ones.
You will write a storage engine that is a thousand times slower than RocksDB. That is the point:
having written one, you will read RocksDB's tuning parameters and know what each of them is
trading against what, which is a thing you cannot learn from the documentation alone.
