# DI-721 · Lesson 09 — Event Sourcing, CDC, and the Log as Truth

**Estimated study time:** 4.5 hours
**Prerequisites:** L01, L05, L08; DS-701 L08 (the outbox and idempotence)

---

## 1. Orientation

Two observations from earlier lessons meet here.

From L01: **the database already has a log** — the write-ahead log — and it contains every change
in commit order. From L08: **a stream processor is a machine for turning a log into a derived
view.** Put them together and you get the idea that reorganises the second half of this course:

> **The log is the system of record. Every table, index, cache, search index and analytical view is
> a materialised view over it** — derived, disposable, and rebuildable by replay.

Kleppmann called this "turning the database inside-out": the components a database keeps private —
the log, the indexes, the materialised views, the replication stream — are unbundled into
composable pieces you assemble yourself.

The practical value is not philosophical. It resolves specific, recurring problems:

- **The dual-write problem** (DS-701 L08): with the log as the source, there is nothing to
  dual-write.
- **"Can you add a new view of this data?"**: replay the log, no migration.
- **"What did this look like last Tuesday?"**: replay to that offset.
- **"Why is this record wrong?"**: read the events that produced it.
- **"The search index is out of sync"**: rebuild it from the log.

And the costs are equally specific, which is why the second half of this lesson is about when *not*
to do it. Event sourcing is a genuinely powerful pattern with a genuinely high complexity floor,
and it is adopted for the wrong reasons more often than any other pattern in this course.

## 2. Theory

### 2.1 Change data capture

**CDC** turns an existing database into an event stream without changing the application. Three
implementations, in ascending order of quality:

1. **Query-based polling**: `SELECT * WHERE updated_at > :last`. Simple, works anywhere, and wrong
   in three ways — it misses deletes entirely, it misses intermediate values between polls, and it
   can miss rows whose transaction committed after a later transaction's (a row can become visible
   with an `updated_at` earlier than the last watermark). That third failure is subtle, real, and
   silently loses data.
2. **Trigger-based**: database triggers write to a changelog table. Captures everything including
   deletes, at the cost of write amplification and application-visible latency on every write.
3. **Log-based**: read the replication log directly (Postgres logical decoding, MySQL binlog,
   MongoDB oplog), typically via Debezium. **This is the right answer.** It captures every change
   including deletes, in exact commit order, with no load on the write path, and it reuses the
   mechanism the database already maintains for replication (DS-701 L03).

Log-based CDC's practical concerns:

- **Snapshot plus stream.** The log does not go back to the beginning of time, so bootstrapping
  requires a consistent snapshot of the current state followed by the stream from the snapshot's
  position, with no gap and no duplication. Getting the handoff right is the hard part; modern
  connectors do incremental snapshotting (Netflix's DBLog algorithm) to avoid locking a large
  table.
- **Replication slots hold WAL.** If the consumer stops, the database cannot recycle WAL, and the
  disk fills. **This is the most common way CDC takes down a production database**, and it needs a
  monitored alert on replication lag from day one.
- **The log contains physical changes.** A schema migration, a `TRUNCATE`, a bulk update of every
  row — all appear in the stream, and downstream must cope.
- **Tombstones for deletes**, and compaction semantics if you use a log-compacted topic.

### 2.2 The outbox, again

DS-701 L08 covered the transactional outbox. The reason it reappears here is the comparison, which
is a real architectural decision:

| | Log-based CDC | Outbox |
|---|---|---|
| Application change | none | application writes events explicitly |
| Event shape | **row changes** (physical) | **domain events** (semantic) |
| Coupling | downstream couples to your schema | downstream couples to your published contract |
| Schema migration | leaks downstream | absorbed by the producer |
| Effort | low | moderate |

The distinction that matters: CDC gives you `customers` row version 7, and downstream must infer
that this means "the customer upgraded their plan". The outbox gives you `CustomerUpgraded`, which
is what the consumer actually wants and which survives a refactor of your tables.

The rule of thumb: **CDC for replication into analytical systems and caches you own; outbox for
events other teams consume.** The first is a technical copy of your data; the second is a published
interface, and interfaces should be designed rather than leaked (L10, and SE-521's information
hiding). Many systems use both, correctly.

### 2.3 Event sourcing

CDC derives events from state. **Event sourcing inverts it: events are the state.**

> The application stores an append-only sequence of domain events per aggregate. Current state is
> derived by folding those events. Nothing is ever updated or deleted.

Mechanics:

- **The event store** is append-only, keyed by aggregate ID, with a per-aggregate sequence number.
  Optimistic concurrency comes free: append with `expected_version`, and a conflict means someone
  else wrote first.
- **Rehydration**: to handle a command, load the aggregate's events and fold them into current
  state. For long-lived aggregates this gets slow, so **snapshots** are stored every N events —
  strictly a cache, always rebuildable, never the source of truth.
- **Projections / read models** consume the event stream and maintain query-optimised views. Each
  projection tracks its own position, and can be rebuilt from zero by resetting that position — the
  operation that makes the whole pattern worthwhile.
- **CQRS** frequently accompanies it: the write side handles commands against aggregates, the read
  side serves queries from projections. Note that CQRS and event sourcing are independent — you can
  have either without the other, and conflating them is a common source of over-engineering.

What you gain: a complete audit trail that is the actual source of truth rather than a parallel
log that can drift; temporal queries; new projections without migrations; debuggability (replay the
exact events that produced the bug); and a natural fit with domain-driven design (SE-521 L06),
because the events *are* the domain language.

### 2.4 What event sourcing costs

Be specific, because this is where projects fail:

- **Eventual consistency on the read side.** A command succeeds and the projection has not caught
  up, so the user's next read does not reflect their own write. Every event-sourced system needs an
  answer to this — the usual ones are to return the new state from the command handler, or to have
  the client wait for the projection to reach a known position.
- **Events are immutable and forever.** A bug that emitted wrong events cannot be fixed by an
  `UPDATE`. You emit a **compensating event** (the same semantic-not-physical rule as DS-701 L08's
  sagas), and every projection must handle it.
- **Schema evolution is unavoidable and permanent.** You will be folding events written five years
  ago by code that no longer exists. Upcasting — transforming old event versions into current shape
  on read — becomes a permanent part of the codebase (L10).
- **Queries are hard by construction.** "All customers in Texas with an unpaid invoice" is not
  answerable from an event log; it requires a projection, which must be designed and maintained.
  Every new question is a new projection.
- **GDPR and deletion.** "Erase all data about this person" against an immutable log is a genuine
  architectural conflict. The standard answer is **crypto-shredding**: encrypt personal data with a
  per-subject key held outside the log and delete the key, rendering the events unreadable. This
  works, and it has to be designed in from the beginning — it cannot be retrofitted.
- **Operational surface**: event store, projections, projection lag monitoring, rebuild procedures,
  snapshot management. All of it must be built and operated by you.

**When it fits**: domains where the history *is* the business (accounting — which is event-sourced
by centuries of practice and is the honest origin of the idea — trading, insurance, logistics,
anything regulated or audited), collaborative editing, and systems needing many different views of
the same facts.

**When it does not**: CRUD applications; small teams without operational capacity; domains where
state is genuinely the truth and history is uninteresting; and any team adopting it because it
sounds sophisticated. **Event-sourcing a CRUD application is a well-documented way to triple your
complexity for no benefit**, and it is common enough that the honest advice is: if you cannot name
the specific question that only the event log answers, do not do it.

A middle path worth knowing: event-source the few aggregates whose history matters, keep the rest
as ordinary tables, and publish events from both via an outbox. Most systems that "do event
sourcing" successfully are actually doing this.

### 2.5 Streams and tables are the same thing

The duality that unifies the whole course:

> A **table** is a snapshot of a stream's accumulated effect. A **stream** is the sequence of
> changes to a table.

Fold a changelog and you get a table; take the diff of a table over time and you get a changelog.
Kafka's log compaction makes the duality concrete: retain only the latest value per key and the
topic *is* a table, materialised as a stream.

Once you see it, several previously separate things become the same thing: database replication
(DS-701 L03) is a stream of changes materialised into a replica; a cache is a materialised view
with an invalidation stream; a search index is a projection; and CQRS read models, CDC pipelines
and stream–table joins (L08 §2.6) are all one pattern.

This also settles the cache invalidation problem in the only satisfying way available: **don't
invalidate, subscribe.** Instead of guessing when a cached value went stale, consume the changelog
and update the cache when the value actually changes. The cache becomes a materialised view, and
"the hardest problem in computer science" becomes an ordinary stream processing job. It is not
free — you have added a pipeline, and it can lag — but the failure mode changes from *silently
wrong* to *measurably late*, which is a much better failure mode.

### 2.6 Operating a log-centric system

- **Retention.** Replay requires the log to still exist. Decide between time-based retention (and
  therefore no full replay after that window, with periodic snapshots as the fallback) and infinite
  retention with compaction. Decide deliberately; discovering the answer during an incident is
  expensive.
- **Ordering.** Order is guaranteed per partition only. Anything requiring ordered processing must
  share a partition key, which means aggregate ID is almost always the partition key, and which
  makes hot aggregates a hot-partition problem (DS-701 L06).
- **Projection lag** is the metric that matters. Alert on it. It is the read side's freshness, and
  it is what users experience.
- **Rebuilds** must be rehearsed. A projection rebuild that has never been run is a recovery plan
  that does not exist. Rehearse it, time it, and know whether it takes minutes or days — because
  that number determines your options during an incident.
- **Poison messages.** One event that a projection cannot process blocks that partition forever.
  You need a dead-letter path and an explicit policy: skip and alert, or halt. Both are defensible;
  having neither is not.

## 3. Construction: a log-centric system

Build in `mpse/di721/l09/`, using your L02 store, your L08 processor and Kafka. This construction
produces the CDC pipeline required by the term artifact.

**Stage 1 — polling CDC, and its three bugs.** Implement `updated_at` polling. Then demonstrate
each failure: a delete that is missed; two updates between polls where the intermediate value is
lost; and the commit-order case where a row committed with an older `updated_at` after your
watermark advanced. That third one requires care to construct and is the one worth the effort — it
is the reason polling CDC is unsafe rather than merely limited.

**Stage 2 — log-based CDC.** Set up Postgres logical replication and consume the change stream
(`wal2json` or `pgoutput`). Verify that all three Stage 1 failures disappear. Then stop the
consumer for ten minutes under write load and observe WAL retention growing — and write the
monitoring query and alert you would deploy.

**Stage 3 — snapshot plus stream.** Bootstrap a consumer from an empty state: take a consistent
snapshot, note its LSN, stream from there, and verify no gap and no duplicate. Test by writing
continuously throughout the bootstrap and checking the final state against the source exactly.

**Stage 4 — outbox, and the comparison.** Add an outbox emitting domain events. Build the same
downstream projection from CDC and from the outbox. Then perform a schema refactor on the source
(split a column, rename a table) and record what breaks in each pipeline. This experiment is the
lesson's central deliverable, and its result is the argument in §2.2 made concrete.

**Stage 5 — an event-sourced aggregate.** Implement an event store (append-only, per-aggregate
sequence, optimistic concurrency on `expected_version`) and one aggregate with real domain events.
Implement command handling by rehydration, then add snapshots and measure rehydration time against
event-count with and without them.

**Stage 6 — projections.** Build two projections over the same stream — one row-oriented for
lookups, one aggregated for reporting. Implement position tracking and rebuild-from-zero. Then time
a full rebuild at 10⁵ and 10⁶ events, and state what that implies for your recovery options.

**Stage 7 — the read-your-writes problem.** Demonstrate the anomaly: a command succeeds, the
immediate read is stale. Then implement two fixes — returning the new state from the command
handler, and having the client wait for the projection to reach the command's position — and
compare their latency and complexity.

**Stage 8 — the hard cases.** (a) Emit a wrong event, then correct it with a compensating event,
and make every projection handle it. (b) Implement crypto-shredding: encrypt one subject's personal
fields with a per-subject key, delete the key, and verify the events are now unreadable while the
non-personal structure survives. (c) Introduce a poison event and implement your dead-letter
policy. Each of these is a case that a design must have an answer for before it goes to production.

**Stage 9 — cache as a materialised view.** Take a cache with TTL-based invalidation and rewrite it
as a projection over the changelog. Measure staleness distribution for both under a write-heavy
workload, and report the change in failure mode as well as the numbers.

## 4. Failure modes

- **Polling CDC in production.** Missed deletes and lost intermediate states, silently.
- **An unmonitored replication slot.** The disk fills and the primary stops. Alert on it before you
  ship the pipeline.
- **CDC as a public interface.** Downstream consumers couple to your table schema, and you can no
  longer refactor. Use an outbox for events others consume.
- **Event sourcing a CRUD application.** Complexity with no corresponding benefit.
- **No projection rebuild rehearsal.** The recovery plan is theoretical.
- **Unbounded rehydration.** No snapshots, and a hot aggregate with 500,000 events takes minutes to
  load.
- **Events as a leaked internal model.** If your events are your class fields serialised, every
  refactor is a breaking change (L10).
- **No plan for deletion requests** against an immutable log. Design crypto-shredding in from the
  start.
- **Ignoring the read-your-writes problem** until users report it as a bug.
- **Assuming global ordering** across partitions. Order is per-partition.
- **No dead-letter path.** One bad event stalls a partition indefinitely.

## 5. Exercises

### Warm-up (30 min)

1. Give three CDC implementations and, for polling, all three failure modes.
2. Compare CDC and the outbox across event shape, coupling and schema migration, and state the rule
   of thumb.
3. Explain the stream–table duality and use it to explain what log compaction does.

### Core (3.5 h)

4. Complete Stages 1–3, including the commit-order failure construction and the WAL-retention
   monitoring query.
5. Complete Stage 4 and deliver the refactor comparison — this is the required deliverable.
6. Complete Stages 5–6, with the rehydration measurements and the rebuild timings.
7. For a system you work with: identify every place data is written to two systems, and for each
   say whether the dual-write problem applies and how it is currently handled. Most systems have at
   least one place where the honest answer is "it is not".

### Challenge

8. Complete Stages 7–9, including all three hard cases in Stage 8.
9. Build a **complete log-centric slice**: an event-sourced order aggregate, three projections (a
   lookup view, a search index in SQLite FTS, and a columnar analytical view from L06), all fed
   from one log, all rebuildable, all with monitored lag. Then perform two exercises against it:
   add a *fourth* projection answering a question nobody anticipated, and time how long from "we
   need this view" to "it is serving accurate results"; and corrupt one projection's state, rebuild
   it while the system serves traffic, and verify correctness afterwards. Write up both, because
   together they are the entire argument for this architecture — and if the numbers are bad, that
   is the argument against it, honestly obtained.

## 6. Self-check

1. State the "log as system of record" claim and four specific problems it resolves.
2. Give the three CDC implementations and why log-based is correct.
3. What is the snapshot-plus-stream problem and why is the handoff hard?
4. How does CDC take down a production database?
5. Compare CDC and the outbox, and give the rule of thumb.
6. Define event sourcing, and distinguish it from CQRS.
7. What is a snapshot in an event store, and why is it never the source of truth?
8. Give six costs of event sourcing.
9. What is crypto-shredding and why must it be designed in from the start?
10. State the stream–table duality and explain how it dissolves cache invalidation.

## 7. Primary sources

- **Kleppmann, "Turning the Database Inside-Out" (2015)** — the talk and transcript; the clearest
  statement of the idea.
- **Kreps, "The Log: What Every Software Engineer Should Know About Real-Time Data's Unifying
  Abstraction" (2013)** — long, and worth every paragraph.
- Kleppmann, *Designing Data-Intensive Applications*, ch. 11 (especially "Databases and Streams").
- Sax et al., "Streams and Tables: Two Sides of the Same Coin" (BIRTE 2018).
- Fowler, "Event Sourcing" and "CQRS" — the canonical short definitions, including the warnings.
- Young, "CQRS Documents" — from someone who later spent years telling people not to over-apply it;
  read the caveats as carefully as the pattern.
- Debezium's documentation on snapshots, and Netflix's "DBLog: A Watermark Based Change-Data-Capture
  Framework" (2019) — the incremental snapshot algorithm.
- Vernon, *Implementing Domain-Driven Design*, chapter 8 — events in a DDD context (with SE-521
  L06).

---

**Previous:** [L08](L08-stream-processing.md) · **Next:**
[L10 — Schema Evolution, Contracts, and Data Modelling](L10-schema-evolution-and-modelling.md)
