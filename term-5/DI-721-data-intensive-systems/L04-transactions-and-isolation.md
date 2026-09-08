# DI-721 · Lesson 04 — Transactions and Isolation Levels

**Estimated study time:** 5 hours
**Prerequisites:** L01; PY-601 L01 (correctness models), DS-701 L04 (consistency models)

---

## 1. Orientation

Isolation levels are the most widely misunderstood feature in databases, and the misunderstanding
is not the reader's fault. The ANSI SQL standard defined them badly — by enumerating three
phenomena that must not occur, in language loose enough that two engines can both claim
"REPEATABLE READ" and behave completely differently. Berenson et al.'s 1995 critique demolished
the definitions, and yet the standard's vocabulary persists in every database's documentation,
which is why you must learn both the standard's terms and what they actually mean.

The practical situation you should carry away from this lesson:

> **Most production databases run at READ COMMITTED, which permits lost updates and write skew.
> Most application code is written as though transactions were serializable. The gap between those
> two facts is where a specific and nasty class of production bug lives** — the kind that appears
> only under concurrency, corrupts data quietly, and cannot be reproduced in testing.

This lesson makes that gap explicit and gives you the tools to close it deliberately: knowing the
levels, knowing the anomalies each permits, and knowing the three ways to prevent an anomaly your
isolation level allows.

The relationship to DS-701 L04 is worth stating up front, because the vocabularies collide.
**Isolation** is about concurrent transactions on one logical copy of the data. **Consistency**
(in the distributed sense) is about the ordering of operations across replicas. Serializability is
an isolation property; linearizability is a recency property; **strict serializability** is both,
and is what Spanner provides. Databases use the word "consistency" for the C in ACID, which means
something else again — application invariants preserved — and is really the application's job. Get
these straight now and a great deal of confusing documentation becomes readable.

## 2. Theory

### 2.1 What ACID actually claims

- **Atomicity** — all or nothing. Not about concurrency; about *abortability*. The mechanism is
  the log (L01 §2.5).
- **Consistency** — the database moves from one valid state to another. This is the weakest and
  vaguest letter: the database enforces the constraints you declared, and *you* are responsible
  for the invariants you did not declare. Härder and Reuter admitted the C was included partly
  because it made the acronym work.
- **Isolation** — concurrent transactions do not interfere. This is the letter with a real
  hierarchy, and this lesson.
- **Durability** — committed data survives a crash. Weaker in practice than it sounds: durability
  is relative to a failure model, and "committed to one node's disk" survives a process crash but
  not a disk failure (DS-701 L03).

### 2.2 The phenomena

The standard defines three; the real list is longer, and the extras are the ones that hurt.

**P1 — Dirty read.** T2 reads a value written by uncommitted T1. If T1 aborts, T2 acted on data
that never existed.

**P2 — Non-repeatable read (fuzzy read).** T2 reads a row twice and gets different values because
T1 committed a change in between.

**P3 — Phantom read.** T2 runs the same *range* query twice and the second returns rows that were
not there before. The distinction from P2 matters because preventing it requires locking a
predicate or a range, not just the rows that exist — which is a categorically harder mechanism.

Beyond the standard:

**Dirty write.** T2 overwrites a value T1 wrote but has not committed. Every isolation level
prevents this, which is why it is not in the list — but it is worth naming, because it is what
row-level write locks exist for.

**Lost update.** Two transactions read-modify-write the same value; one update is silently lost.
The classic counter increment. **Permitted at READ COMMITTED**, and it is the single most common
concurrency bug in application code.

**Read skew.** T2 reads object A, then T1 commits changes to A and B, then T2 reads B — and T2
sees a state that never existed as a whole. This breaks backups, analytics, and integrity checks,
and is the main argument for snapshot isolation.

**Write skew.** Two transactions read an overlapping set, each checks an invariant that currently
holds, and each writes a *different* row based on that check. The invariant is violated even
though neither transaction wrote what the other read. This is the anomaly that **snapshot
isolation permits** and it is the reason SI is not serializable — §2.4.

### 2.3 The levels

The standard's table:

| Level | Dirty read | Non-repeatable read | Phantom |
|---|---|---|---|
| READ UNCOMMITTED | possible | possible | possible |
| READ COMMITTED | prevented | possible | possible |
| REPEATABLE READ | prevented | prevented | possible |
| SERIALIZABLE | prevented | prevented | prevented |

Berenson et al.'s objections, which you need in order to read any real documentation:

1. **The phenomena are ambiguous.** Read "loosely" (any interleaving matching the shape) they
   forbid more than intended; read "strictly" they forbid less. The paper shows the definitions
   admit histories that are clearly non-serializable at every level below SERIALIZABLE.
2. **The list is incomplete.** Lost update and write skew are not covered at all, and they are
   the ones that damage data.
3. **The levels are lock-based in origin** (they describe what two-phase locking with various lock
   durations yields) but multiversion engines achieve them differently and therefore permit
   different anomalies at the same nominal level.

And the consequence you must internalise: **what an engine calls a level tells you very little.**

- Postgres's REPEATABLE READ is **snapshot isolation** — it prevents phantoms (which the standard
  does not require at that level) but permits write skew.
- Oracle's SERIALIZABLE is **also snapshot isolation** — it permits write skew, despite the name.
- MySQL/InnoDB's REPEATABLE READ is snapshot isolation for reads *plus* next-key locking for
  locking reads, which prevents some but not all anomalies, and behaves differently for plain
  `SELECT` versus `SELECT … FOR UPDATE`.
- Postgres's SERIALIZABLE is **serializable snapshot isolation** (SSI) — genuinely serializable
  (L05).

Therefore: **do not reason from the level's name. Reason from the anomalies the engine's
documentation says it permits — and verify with a test.** Stage 2 of §3 is that test.

### 2.4 Snapshot isolation and write skew

**Snapshot isolation**: each transaction reads a consistent snapshot as of its start; writes are
buffered; at commit, the transaction aborts if another transaction has committed a write to a row
it also wrote (the **first-committer-wins** rule).

SI is excellent value: readers never block writers, writers never block readers, and read skew,
non-repeatable reads and phantoms all disappear. It is the default in Postgres's REPEATABLE READ,
Oracle, SQL Server's snapshot mode, and most MVCC systems (mechanism in L05).

But it permits **write skew**, and the canonical example is worth memorising:

> Two doctors are on call. The rule is that at least one must remain on call. Alice and Bob both
> feel ill and, simultaneously, each transaction reads "there are 2 doctors on call — fine" and
> then removes *itself*. Neither transaction wrote a row the other read. First-committer-wins does
> not fire, because they wrote different rows. Both commit. Zero doctors are on call.

The general shape: a **read of a set**, a **decision based on the whole set**, and a **write to
one member**. Once you see the shape you find it everywhere — booking systems (double-booking a
room), inventory (overselling the last unit), uniqueness enforced in application code, and
approval workflows requiring N approvers.

Note carefully that this is not the same as a lost update, and the fixes differ: a lost update is
two writes to *the same* row and is caught by first-committer-wins or by a version check; write
skew is writes to *different* rows and is caught by neither.

The **phantom** is the mechanism underneath: the transaction's decision depends on the *absence*
of rows, or on the set's membership, and there is nothing to lock, because the thing to lock does
not exist yet. This is why preventing write skew requires either materialising the conflict (§2.5)
or predicate/SSI machinery.

### 2.5 Preventing what your level permits

Four options, in ascending order of cost and descending order of how often people reach for them:

1. **Atomic operations.** `UPDATE counters SET n = n + 1 WHERE id = ?` is safe at any isolation
   level, because the read and the write happen inside the engine under a row lock. Prefer this
   whenever the operation can be expressed as one statement. Most lost updates disappear here.
2. **Explicit locking.** `SELECT … FOR UPDATE` takes a write lock on the rows read, converting a
   read-modify-write into something the engine can serialise. It works for lost updates and, when
   you can lock the *right* rows, for write skew — but the doctors example shows the trap: you
   must lock the rows whose *absence or presence* matters, which sometimes means locking a parent
   row (a materialised conflict, below). Note also the deadlock risk: consistent lock ordering
   is your responsibility (PY-601 L03's lesson, at a different scale).
3. **Optimistic concurrency (compare-and-set).** `UPDATE … WHERE id = ? AND version = ?`, checking
   that one row was affected and retrying otherwise. No locks, excellent under low contention,
   requires the retry loop to be correct and bounded. This is the same mechanism as DS-701 L08's
   conditional write.
4. **Materialising the conflict.** When there is no row to lock, create one. A `bookings` table
   with a row per (room, time slot) gives you something to lock for a booking system; a
   `on_call_constraint` row per shift gives the doctors something to contend on. It is inelegant —
   you are adding a row purely as a lock target — and it is sometimes the only practical answer
   short of full serializability.
5. **Use a genuinely serializable isolation level** and handle serialization failures with retries.
   This is often the right answer and is under-used out of a vague fear of cost; L05 covers what
   it actually costs.

The discipline: **for each transaction in your system, name the anomalies your isolation level
permits and state, per transaction, why they are harmless or how you prevented them.** That
document is short, and almost nobody writes it. Writing it finds bugs.

### 2.6 Two-phase locking, briefly

The classical route to serializability. **2PL**: a growing phase in which locks are acquired and a
shrinking phase in which they are released; **strict 2PL** holds all locks until commit, which
also gives recoverability. Shared locks for reads, exclusive for writes.

Serializability requires handling phantoms, which needs **predicate locks** (lock the *condition*,
not the rows) — too expensive in general, so real systems use **index-range locks** (next-key
locking in InnoDB): lock the index range covering the predicate. This works when there is an
index; without one it degenerates to locking the whole table.

2PL's properties: correct, and famous for being slow. Locks serialise access, deadlocks require
detection and victim selection, and one long-running transaction can block many others. Throughput
under contention degrades badly, and tail latency degrades worse. That reputation is why the
industry moved to MVCC, and why "serializable is too slow" became folklore — a belief formed
against 2PL and inherited by SSI, which behaves quite differently (L05).

### 2.7 Long transactions

Not an isolation level, but the operational issue that isolation creates, and it belongs here
because it causes real outages:

- **In a locking system**, a long transaction holds locks and blocks everything behind it.
- **In an MVCC system**, a long transaction holds back the horizon of versions that can be cleaned
  up, so dead tuples accumulate across the *entire* database, tables bloat, and vacuum cannot
  reclaim. A single forgotten `BEGIN` in an idle session can bloat a production database over
  hours (L05 §2.5).
- **Either way**, an application that opens a transaction and then makes a network call to a third
  party has coupled its database's health to that third party's latency. Never hold a transaction
  open across an external call.

Keep transactions short, do not hold them across user think-time or network calls, and monitor the
oldest running transaction as a first-class metric.

## 3. Construction: an anomaly laboratory

Build in `mpse/di721/l04/`. The deliverable is a test suite that *demonstrates* each anomaly, and
its value is permanent: you will re-run it against every database you ever adopt.

**Stage 1 — the harness.** A framework for running two (or three) transactions with controlled
interleaving: `T1.begin()`, `T1.read(x)`, `T2.begin()`, `T2.write(x)`, `T1.read(x)`, and so on,
with explicit synchronisation so the interleaving is deterministic rather than hoped-for. This
harness is what makes concurrency bugs reproducible, and it is the reason this lesson has a
construction section at all.

**Stage 2 — the anomaly catalogue.** For each of: dirty read, dirty write, non-repeatable read,
phantom, lost update, read skew, write skew — write the minimal interleaving that exhibits it.
Run each against READ UNCOMMITTED, READ COMMITTED, REPEATABLE READ and SERIALIZABLE, and build the
7×4 result matrix for your engine. Then compare your matrix with the ANSI table and write up every
cell where they differ, with the explanation.

**Stage 3 — a second engine.** Run the same suite against MySQL/InnoDB (or SQLite, or your own
engine from L02 once it has transactions). Produce the side-by-side matrix. The differences at the
same nominal level are the point of the exercise, and seeing them yourself is worth more than
reading this lesson twice.

**Stage 4 — lost updates, four ways.** Take a counter increment. Implement it (a) as a naive
read-modify-write, (b) as an atomic `UPDATE … SET n = n + 1`, (c) with `SELECT … FOR UPDATE`,
(d) with a version-based CAS and a retry loop. Run 100 concurrent clients doing 100 increments
each and check the final value. Record correctness *and* throughput for each. One of them will be
wrong; the other three will differ in speed by more than you expect.

**Stage 5 — write skew.** Implement the doctors-on-call scenario and demonstrate the violation
under snapshot isolation. Then fix it four ways: `SELECT … FOR UPDATE` on the right rows,
materialising the conflict, a database constraint, and SERIALIZABLE with retry. Measure each and
write up which you would ship and why.

**Stage 6 — deadlocks.** Construct one deliberately with inconsistent lock ordering. Observe the
engine's detection and victim selection, and the error your application receives. Then write the
retry wrapper that handles it correctly — including the part everyone gets wrong: the retry must
re-execute the *whole transaction*, re-reading everything, because the state has changed.

**Stage 7 — the long transaction.** Open a transaction, read one row, and leave it idle. Then run
a heavy update workload and measure table bloat and vacuum behaviour over ten minutes. Plot it.
Then write the monitoring query that would have caught it, and the alert threshold you would set.

**Stage 8 — transactions in your own engine.** Add transactions to your L02 LSM: a write set
buffered in memory, atomic commit via a single log record containing all writes, and READ
COMMITTED isolation from the fact that uncommitted writes are not yet in the memtable. Then run
your Stage 2 catalogue against it and document exactly which anomalies it permits. Do not fix
them yet — L05 adds MVCC.

## 4. Failure modes

- **Trusting the level's name.** Oracle's SERIALIZABLE is snapshot isolation. Postgres's
  REPEATABLE READ is stronger than the standard requires. Verify.
- **Read-modify-write in application code** without a lock, a CAS, or an atomic statement. The
  lost update, and the most common concurrency bug in production systems.
- **Assuming a `SELECT` before an `INSERT` prevents duplicates.** It does not, at any level below
  serializable, and it does not even then unless the read is protected. Use a unique constraint —
  which is the database materialising the conflict for you.
- **Not knowing your default level.** It is usually READ COMMITTED, and it permits more than most
  developers assume.
- **Holding a transaction across a network call.** Couples your database to someone else's uptime.
- **Retrying only the failed statement** rather than the whole transaction after a serialization
  failure or deadlock. The retried statement acts on stale reads.
- **Ignoring serialization failures.** At SERIALIZABLE they are normal and expected; an
  application that treats them as errors rather than retrying is not using the level correctly.
- **Testing concurrency with a single client.** Every anomaly here is invisible without controlled
  interleaving — hence Stage 1.

## 5. Exercises

### Warm-up (30 min)

1. Define dirty read, non-repeatable read, phantom, lost update, read skew and write skew, each
   with a two-transaction interleaving.
2. Explain what distinguishes a phantom from a non-repeatable read, and why the distinction
   changes the required locking mechanism.
3. Explain why write skew is not prevented by first-committer-wins, being precise about which rows
   each transaction wrote.

### Core (3.5 h)

4. Complete Stages 1–3 and deliver both engines' 7×4 matrices with the differences explained.
5. Complete Stage 4 and deliver the correctness/throughput table.
6. Complete Stage 5 with all four fixes and the recommendation.
7. For a system you work with: determine its isolation level, list its three most concurrency-
   sensitive transactions, and for each name the anomalies the level permits and state why they
   are harmless or how they are prevented. This is the document from §2.5, and writing it for a
   real system is the most valuable exercise in the lesson.

### Challenge

8. Complete Stages 6–8, including the bloat plot and the monitoring query.
9. Implement a **general write-skew detector** for a workload: given transaction read sets and
   write sets logged from a run, build the serialization graph (with rw-antidependency edges) and
   detect cycles, reporting which transaction pairs could have produced an anomaly. Validate it
   against your Stage 2 catalogue — it must find the write skew and must not flag the safe cases.
   You are building, in miniature, the mechanism L05's SSI uses at runtime, and doing it offline
   first makes L05 much easier.

## 6. Self-check

1. What does each ACID letter actually claim, and which is partly the application's job?
2. Give the ANSI table, then give three ways real engines deviate from it.
3. What is snapshot isolation, and what is the first-committer-wins rule?
4. Describe write skew with the doctors example and state its general shape.
5. Why does first-committer-wins not prevent write skew?
6. Give five ways to prevent an anomaly your isolation level permits, in order of preference.
7. What is materialising the conflict, and when is it necessary?
8. What is strict 2PL and why did the industry move away from it?
9. Why do phantoms require predicate or index-range locks?
10. Give three distinct harms caused by a long-running transaction.

## 7. Primary sources

- **Berenson, Bernstein, Gray, Melton, O'Neil & O'Neil, "A Critique of ANSI SQL Isolation Levels"
  (SIGMOD 1995)** — read this before trusting any isolation documentation, including this lesson's.
- **Adya, Liskov & O'Neil, "Generalized Isolation Level Definitions" (ICDE 2000)** — the
  implementation-independent definitions that fix the standard's problems.
- Gray & Reuter, *Transaction Processing*, chapters 7–8.
- Bernstein, Hadzilacos & Goodman, *Concurrency Control and Recovery in Database Systems* (1987) —
  free online.
- Bailis et al., "Highly Available Transactions: Virtues and Limitations" (VLDB 2014) — which
  isolation levels are achievable without coordination.
- Kleppmann, *Designing Data-Intensive Applications*, ch. 7 — the best modern treatment of write
  skew for practitioners.
- Your engine's isolation documentation, read *after* running Stage 2 against it. It will read
  differently.

---

**Previous:** [L03](L03-indexing.md) · **Next:**
[L05 — MVCC, Snapshots, and Serializability](L05-mvcc-and-serializability.md)
