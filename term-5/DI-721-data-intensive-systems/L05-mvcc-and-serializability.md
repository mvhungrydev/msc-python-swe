# DI-721 · Lesson 05 — MVCC, Snapshots, and Serializability

**Estimated study time:** 5 hours
**Prerequisites:** L02, L04

---

## 1. Orientation

L04 described isolation levels as a menu of guarantees. This lesson is about the mechanism that
delivers them in every modern database, and about what it costs — because MVCC's costs are the
source of a large fraction of real operational pain in Postgres, MySQL, Oracle and SQL Server
alike.

The idea in one sentence:

> **Don't overwrite. Keep old versions, and let each transaction read the version that was current
> when its snapshot was taken.**

The consequence, which is why every serious database does this:

> **Readers never block writers, and writers never block readers.**

Under 2PL, a long analytical query holds shared locks and blocks every update to the rows it
touches. Under MVCC it reads a snapshot and blocks nothing. That single property is worth an
enormous amount, and it is why MVCC won.

But nothing is free, and MVCC's bill arrives as: storage for old versions, work to determine which
version is visible, and — the part that causes outages — **the garbage collection problem**. Dead
versions must be reclaimed, reclamation cannot proceed past the oldest running transaction, and
therefore *one idle transaction can degrade an entire database*. That failure is common enough
that recognising it is a professional skill.

The second half of the lesson is the good news: **serializable snapshot isolation** makes true
serializability affordable, and most of the folklore about serializability being unusably slow
predates it.

## 2. Theory

### 2.1 Versions and visibility

Each row version carries metadata about when it became visible and when it stopped being visible.
In Postgres's heap the version (a *tuple*) carries `xmin` (the transaction that created it) and
`xmax` (the transaction that deleted or superseded it), and both are transaction IDs.

A transaction takes a **snapshot** consisting of: the highest transaction ID assigned so far, and
the set of transaction IDs currently in progress. A version is visible to that snapshot if:

- its `xmin` committed *before* the snapshot was taken and was not in progress at that moment, and
- its `xmax` is either unset, or belongs to a transaction that had not committed by then.

That is the whole visibility rule, and it is worth writing out because everything else follows
from it. An `UPDATE` under MVCC is a `DELETE` plus an `INSERT`: it sets the old version's `xmax`
and writes a new version. **There is no in-place update of a row's data**, which explains several
otherwise-puzzling behaviours: why updates cost roughly what inserts do, why updating one column
of a wide row rewrites the whole row, and why every secondary index must gain an entry pointing at
the new version (unless HOT applies, L03 §2.4).

Two families of implementation:

- **Append-in-place (Postgres).** New versions go into the heap alongside old ones. Simple, fast
  for readers, and it makes tables grow — which is why `VACUUM` exists.
- **Undo log (InnoDB, Oracle).** The current version is updated in place and the *old* version is
  reconstructed by applying undo records backwards. Tables stay compact; reading an old snapshot
  costs more the older it is; and a long-running query can fail outright when the undo it needs
  has been purged — Oracle's famous "snapshot too old".

Neither is better in general, and knowing which one you are running on tells you which failure
mode to expect: bloat, or snapshot-too-old.

### 2.2 Snapshot isolation, precisely

**SI** = every read sees the snapshot as of transaction start; writes are private until commit; at
commit, abort if a concurrent transaction has already committed a write to a row this transaction
also wrote (**first-committer-wins**).

What SI gives you, for free, relative to READ COMMITTED: no non-repeatable reads, no read skew, no
phantoms (the snapshot simply does not contain later rows), and no lost updates *on the same row*.
What it does not give you: **write skew** (L04 §2.4), and the general absence of serialization
anomalies.

**READ COMMITTED under MVCC** is subtly different and worth stating because it is almost certainly
your default: it takes a **new snapshot for every statement**, not one per transaction. So two
identical `SELECT`s in one transaction can return different results, and — this is the part that
surprises people — an `UPDATE` that has to wait on a row lock will, when it acquires the lock,
re-evaluate its `WHERE` clause against the *newly committed* version rather than its original
snapshot. That "EvalPlanQual" behaviour prevents some lost updates that a naive reading of READ
COMMITTED would permit, and it is another reason engine behaviour cannot be inferred from the
standard.

### 2.3 The formal picture: serialization graphs

To understand SSI you need the standard theory, and it is worth having anyway.

Build a graph with a node per transaction and an edge T1 → T2 whenever T2 depends on T1:

- **wr** (read dependency): T1 writes x, T2 reads T1's version.
- **ww** (write dependency): T1 writes x, T2 overwrites it.
- **rw** (anti-dependency): T1 reads x, T2 then writes a new version of x. T1 must be ordered
  *before* T2, because T1 did not see T2's write.

> **A schedule is conflict-serializable iff this graph is acyclic.**

The key theorem behind SSI (Adya; then Fekete et al.): **every non-serializable execution under
snapshot isolation contains a cycle with at least two consecutive rw-antidependency edges**, and
those two edges are "in" and "out" of a single transaction — a **pivot**. In the doctors example,
each transaction reads what the other is about to write: two rw edges, forming exactly this
structure.

That theorem is what makes efficient detection possible. You do not need to build and check the
whole graph; you need only detect a transaction with both an incoming and an outgoing
rw-antidependency.

### 2.4 Serializable Snapshot Isolation

**SSI** (Cahill, Röhm and Fekete, 2008; shipped in PostgreSQL 9.1) runs snapshot isolation and
adds runtime tracking:

- Reads acquire **SIREAD locks** — not real locks; they block nothing and are purely a record of
  "this transaction read this data".
- When a write conflicts with an SIREAD, an rw-antidependency edge is recorded.
- A transaction with both an incoming and an outgoing rw edge is a **pivot** and is a candidate for
  abort. When the conditions for a dangerous structure are met, one transaction in it is aborted
  with a **serialization failure**.

The properties that matter in practice:

- **Genuinely serializable.** Write skew is prevented, and so is everything else.
- **Optimistic**: nothing blocks. Throughput under low contention is close to plain SI.
- **False positives are possible** — some aborted transactions would in fact have been safe. The
  detection is conservative because being exact is too expensive.
- **The application must retry.** A `SERIALIZABLE` application without a retry loop is broken by
  construction. The retry must re-run the whole transaction.
- **Predicate reads need index support.** SIREAD locks are taken at the granularity available;
  without a useful index, a range predicate escalates to a page or relation-level SIREAD, which
  inflates false positives dramatically. **Serializable performance therefore depends on your
  indexes**, which is a genuinely surprising connection and a practical one.

The correction to the folklore: serializability's bad reputation was earned by 2PL. SSI is a
different mechanism with a different cost curve — the cost appears as *aborts under contention*
rather than as *blocking*, and for many OLTP workloads the abort rate is low enough that the
simplicity of not having to reason about anomalies is worth it. Measure before assuming otherwise
(Stage 5).

### 2.5 The garbage collection problem

This is the section with the most operational value in the lesson.

Old versions must be reclaimed. A version can be reclaimed only when **no running transaction
could still need to see it** — that is, when it is older than the oldest snapshot currently held.
So:

> **The oldest running transaction pins the reclamation horizon for the entire database.**

An idle-in-transaction session that ran one `SELECT` and then went to lunch prevents cleanup of
dead versions in *every* table, including ones it never touched. The consequences compound:

- **Table bloat.** Tables and indexes grow; sequential scans read dead rows; the buffer pool caches
  dead rows; performance degrades everywhere.
- **Index bloat**, which is worse, because index pages do not readily return space.
- **Vacuum falling behind**, so the problem accelerates once it starts.
- **In Postgres specifically, transaction ID wraparound.** Transaction IDs are 32-bit and compared
  modulo 2³¹. If vacuum cannot freeze old tuples before the ID space wraps, the database *shuts
  down* to protect data. Postgres emits increasingly urgent warnings first, and a shutdown of this
  kind is always the end of a long chain of ignored ones.

The operational discipline follows directly:

1. Monitor the **oldest running transaction** and the **oldest `idle in transaction` session** as
   first-class metrics with alerts. This one metric predicts most MVCC incidents.
2. Set `idle_in_transaction_session_timeout` (and its equivalents) so a forgotten transaction dies.
3. Monitor **dead tuple counts** and vacuum progress; tune autovacuum to be more aggressive than
   the defaults on large, hot tables — the defaults are conservative and were chosen for smaller
   machines than yours.
4. Run long analytical queries against a replica, where they pin that replica's horizon rather than
   the primary's — noting that with hot standby feedback enabled, they pin the *primary's* horizon
   too, which is a trap worth knowing.
5. Never hold a transaction open across an external call (L04 §2.7).

Undo-log systems trade this for the reciprocal problem: the undo segment grows, purge falls behind,
reads of old snapshots get slower as they walk longer undo chains, and eventually a long query
fails with "snapshot too old". Same underlying cause — a long-running reader — different symptom.

### 2.6 Choosing an isolation level, concretely

A workable default policy:

- **READ COMMITTED** for the bulk of simple OLTP where each transaction is a single statement or
  where read-modify-write is expressed atomically. Cheapest, no retries.
- **SNAPSHOT / REPEATABLE READ** for read-only transactions that must see a consistent view across
  several statements — reports, exports, integrity checks, backups. This is the one people most
  often fail to use, and read skew in a nightly report is a real and under-diagnosed bug.
- **SERIALIZABLE** for the small number of transactions with cross-row invariants — anything with
  the write-skew shape from L04 §2.4. Mixing levels per transaction is fine and normal; note that
  in Postgres SSI's guarantees only hold among transactions that are *all* serializable, so a
  serializable transaction is not protected from a concurrent read-committed one.
- Above all: **write down, per transaction, which level it uses and why.**

## 3. Construction: implement MVCC

Build in `mpse/di721/l05/`, extending the L02 storage engine and the L04 harness. This is the
construction where the theory becomes concrete, and doing it makes the rest of the course easier.

**Stage 1 — versioned storage.** Change your engine's value format to a chain of versions, each
with `xmin`, `xmax` and the payload. Implement a transaction ID counter and a commit log recording
each transaction's status (in-progress / committed / aborted).

**Stage 2 — snapshots and visibility.** Implement snapshot acquisition (highest ID, plus the set of
in-progress IDs) and the visibility rule from §2.1 as a single well-tested function. Property-test
it: for any sequence of transactions and any snapshot, exactly one version of each key is visible,
and the visible version never changes for a fixed snapshot. That invariant is the whole point of
MVCC and it should be an executable assertion.

**Stage 3 — snapshot isolation.** Transaction-start snapshots, buffered writes, and
first-committer-wins at commit. Run your L04 anomaly catalogue against it: dirty reads,
non-repeatable reads, phantoms and read skew should all disappear; **write skew should remain**.
Demonstrate the doctors scenario failing on your own engine. That demonstration is the deliverable.

**Stage 4 — read committed.** Add per-statement snapshots. Show the difference from SI with a
transaction that reads the same key twice around a concurrent commit. Then implement the
lock-wait-and-re-evaluate behaviour of §2.2 and construct the case that distinguishes it from a
naive implementation.

**Stage 5 — SSI.** Track read sets (your SIREAD equivalent), detect rw-antidependencies, identify
pivots with both an incoming and an outgoing rw edge, and abort. Verify that the doctors scenario
now aborts one transaction. Then measure: run a mixed workload at SI and at SSI across contention
levels from low to high, and plot throughput and abort rate for both. This chart is what lets you
form your own opinion about serializability's cost instead of inheriting one.

**Stage 6 — the false positive.** Construct a workload where SSI aborts a transaction that was
actually safe. Explain why the detection is conservative and what an exact detector would cost.
Then demonstrate the index effect: run a range-predicate workload with and without a supporting
index and show the abort rate changing because of read-set granularity.

**Stage 7 — garbage collection.** Implement version cleanup with a horizon computed from the oldest
active snapshot. Then reproduce the pathology: start a long-lived transaction, run a heavy update
workload, and plot storage growth and read latency over time. Kill the long transaction and show
the recovery. Write the monitoring query you would use in production and the alert threshold.

**Stage 8 — against a real database.** Run your L04 harness against Postgres at REPEATABLE READ and
SERIALIZABLE, measure the abort rate under contention, and compare the shape against your own
implementation's. Where they differ, find out why — the differences are usually about read-set
granularity and are instructive.

## 4. Failure modes

- **`idle in transaction` sessions.** The single most common MVCC operational problem. Set the
  timeout.
- **Long analytics on the primary.** Pins the horizon; use a replica, and understand what hot
  standby feedback does to that plan.
- **Autovacuum left at defaults on a large hot table.** Falls behind, and the deficit compounds.
- **Ignoring wraparound warnings.** They escalate to a shutdown, and by then the recovery is long.
- **SERIALIZABLE without a retry loop.** Not using the level; just failing at it.
- **Retrying only the failed statement.** The transaction's earlier reads are stale.
- **Assuming SERIALIZABLE protects you from non-serializable transactions.** In Postgres, SSI's
  guarantee holds among serializable transactions only.
- **Assuming SI is serializable because it prevents phantoms.** It prevents phantoms and still
  permits write skew; these are independent.
- **Blaming disk for bloat-driven slowdowns.** The scans got slower because they are reading dead
  tuples, not because the disk changed.

## 5. Exercises

### Warm-up (30 min)

1. State the visibility rule using `xmin`, `xmax` and a snapshot, and use it to explain why an
   `UPDATE` is a delete plus an insert.
2. Compare append-in-place and undo-log MVCC, naming the characteristic failure of each.
3. Define the three dependency edge types and state the conflict-serializability theorem.

### Core (3.5 h)

4. Complete Stages 1–3. Deliver the property test from Stage 2 and the write-skew demonstration on
   your own engine.
5. Complete Stages 4–5 and deliver the SI-versus-SSI throughput and abort-rate chart.
6. Complete Stage 7 and deliver the bloat plot, the recovery, and the monitoring query.
7. For a system you work with: find the oldest running transaction right now, find the tables with
   the highest dead-tuple ratio, and determine whether autovacuum is keeping up. Write 400 words
   on what you found and what you would change.

### Challenge

8. Complete Stages 6 and 8, including the false-positive construction and the index-granularity
   demonstration.
9. Implement **snapshot export / time travel**: allow a client to open a transaction at a
   *specified* past snapshot, so historical queries are possible. Then confront the two problems
   this creates — the garbage collector must not reclaim versions any exported snapshot might need,
   and there must be a retention bound so the horizon cannot be pinned forever. Design and
   implement a retention policy, then write up how your design relates to the tension in §2.5 and
   to what "snapshot too old" is really protecting.

## 6. Self-check

1. State the MVCC visibility rule precisely.
2. Why do readers not block writers under MVCC, and why does that matter?
3. Give the two implementation families and each one's characteristic failure.
4. How does READ COMMITTED under MVCC differ from SI, in two respects?
5. What structure does every non-serializable SI execution contain?
6. What is an SIREAD lock, and in what sense is it not a lock?
7. Why does SSI produce false positives, and what makes them more frequent?
8. Why does serializable performance depend on indexes?
9. Explain how one idle transaction can degrade an entire database.
10. Give a per-transaction isolation policy for a typical OLTP application, with justification for
    each level.

## 7. Primary sources

- **Cahill, Röhm & Fekete, "Serializable Isolation for Snapshot Databases" (SIGMOD 2008)** — SSI,
  and the basis of PostgreSQL's implementation.
- **Fekete, Liarokapis, O'Neil, O'Neil & Shasha, "Making Snapshot Isolation Serializable"
  (TODS 2005)** — the dangerous-structure theorem.
- Adya, Liskov & O'Neil, "Generalized Isolation Level Definitions" (ICDE 2000).
- Ports & Grittner, "Serializable Snapshot Isolation in PostgreSQL" (VLDB 2012) — the
  implementation report; unusually candid about the trade-offs.
- Wu et al., "An Empirical Evaluation of In-Memory Multi-Version Concurrency Control" (VLDB 2017)
  — a systematic comparison of MVCC design choices.
- Bernstein, Hadzilacos & Goodman, *Concurrency Control and Recovery*, chapters 1–5.
- The PostgreSQL documentation on MVCC, vacuum and transaction ID wraparound; and the InnoDB
  documentation on undo logs and purge. Read whichever matches your production system.

---

**Previous:** [L04](L04-transactions-and-isolation.md) · **Next:**
[L06 — Column Stores and Analytical Processing](L06-column-stores.md)
