# DS-701 · Lesson 08 — Transactions, Sagas, and Idempotence

**Estimated study time:** 4.5 hours
**Prerequisites:** L01, L03, L05; SE-521 L07 (aggregates and consistency boundaries)

---

## 1. Orientation

A single-node database gives you a transaction: a set of writes that either all happen or none
do, isolated from concurrent work, durable once committed. It is one of the great abstractions
in computing, and the reason is that it lets you reason about a multi-step change as if it were
a single step.

The moment your data lives on two machines that can fail independently, that abstraction stops
being free. This lesson is about what happens to it. There are three honest answers, and a
mature engineer knows which one they are choosing:

1. **Pay for real atomicity across machines** — atomic commit protocols. Correct, and it costs
   you availability in exactly the way L01's impossibility results predict.
2. **Give up atomicity and buy back correctness with compensation** — sagas. The business
   process, not the database, becomes responsible for the "all or nothing" property.
3. **Design the boundary so the question never arises** — put everything that must change
   together inside one consistency boundary (one aggregate, one partition, one node). This is
   the answer that senior engineers reach for first and juniors reach for last.

The third answer is not a dodge. It is the direct application of SE-521 L07: an aggregate is
defined as the unit of transactional consistency, and choosing aggregate boundaries well is
choosing where distributed transactions are *not* needed. Most systems that "need" distributed
transactions have a partitioning mistake upstream.

Underneath all three sits one skill that you will use every single day regardless of which you
choose: **idempotence**. Because L01 §2.6 established that exactly-once *delivery* is
impossible, exactly-once *effect* has to be manufactured by the receiver. Everything in §2.7 of
this lesson is the practical craft of doing that.

## 2. Theory

### 2.1 What atomicity means when the machines are separate

Local atomicity is easy because there is one place that decides. A write-ahead log entry either
made it to disk or it did not, and one process reads that log on recovery.

Distributed atomicity is the **atomic commit problem**: a set of participants, each of which can
independently commit or abort, must reach a *unanimous* decision.

> **Atomic commit.** All correct participants decide the same value (agreement); a participant
> can only decide *commit* if every participant voted *yes* (validity); every correct participant
> eventually decides (termination).

Note the asymmetry against consensus (L05). Consensus needs a majority; atomic commit needs
**unanimity for commit** — any single "no" vote, or any participant that cannot be reached and
whose vote is therefore unknown, forces abort. A distributed transaction is only as available as
the *least* available participant, and its failure probability compounds across participants
rather than being masked by a majority. This is the single most important fact in the lesson.

### 2.2 Two-phase commit

**2PC** is the classical protocol. A coordinator drives two rounds.

**Phase 1 (prepare / vote).** The coordinator sends `PREPARE`. Each participant does everything
short of committing — validates, acquires locks, writes its changes and its vote durably to its
log — then replies `YES` or `NO`. A `YES` is a **promise**: the participant has given up its
right to abort unilaterally, and must be able to commit even if it crashes and restarts.

**Phase 2 (commit / abort).** If all votes are `YES`, the coordinator durably logs `COMMIT` and
broadcasts it; otherwise `ABORT`. Participants apply the decision, release locks, and acknowledge.

The protocol is correct. Its problem is precisely characterised:

> **2PC is a blocking protocol.** If the coordinator fails after a participant has voted `YES`
> but before that participant learns the decision, the participant is stuck: it cannot commit
> (the decision may have been abort) and cannot abort (the decision may have been commit). It
> holds its locks until the coordinator recovers.

This is the **in-doubt** or **uncertainty** window, and it is not an implementation defect — a
theorem (Skeen and Stonebraker, 1983) shows that *no* atomic commit protocol can be non-blocking
in an asynchronous system where the coordinator may fail. It is FLP (L01) in another costume.

Practical consequences that show up in real incidents:

- **The coordinator's log is the system of record.** If it is lost, the transaction's outcome is
  unrecoverable, and operators are reduced to *heuristic decisions* — a human or a timeout
  guessing "commit" or "abort" per participant. Heuristics can produce genuine atomicity
  violations, which the XA standard acknowledges with a distinct `XA_HEURHAZ` return code. That
  an industry-standard API has an error code meaning "atomicity may have been broken" is worth
  sitting with for a moment.
- **Locks are held across a network round trip plus a possible failure window.** Throughput
  collapses under contention and the tail latency is unbounded.
- **The coordinator must itself be replicated** to be trustworthy — at which point you are
  paying for consensus *and* 2PC.

**Three-phase commit** adds a pre-commit round to remove blocking, but only under a synchronous
model with reliable failure detection — assumptions L01 told you not to rely on. It is not used
in practice.

The construction that *does* work is **Paxos Commit** (Gray and Lamport, 2006): run a consensus
instance per participant's vote, so the decision survives coordinator failure. This is what
Spanner does — 2PC over Paxos groups, with the coordinator's log itself replicated. The takeaway
is not "2PC is broken" but "**2PC over a fault-tolerant log is fine; 2PC over a single
coordinator is a liability.**"

### 2.3 Sagas

If you cannot hold a transaction open across services, hold a **sequence** of local transactions
and make the sequence recoverable.

> **Saga.** A long-lived transaction decomposed into a sequence of local transactions
> `T1, T2, …, Tn`, each with a **compensating transaction** `C1, …, Cn` that semantically undoes
> it. If `Tk` fails, execute `C(k−1), …, C1` in reverse order.

The guarantee is **ACD, not ACID** — atomicity (in the "all effects, or all compensated" sense),
consistency and durability, but explicitly **no isolation**. That missing I is where all the
difficulty lives (§2.6).

The other essential property: **compensation is semantic, not physical.** You cannot un-send an
email; you send an apology. You cannot un-charge a card; you issue a refund, which is a new and
visible transaction. Compensation therefore has to be designed at the *business* level, and some
operations have no compensation at all — which is a design signal to reorder the saga so the
irreversible step comes last.

Three practical rules:

1. **Compensations must be idempotent and must not fail permanently.** They are retried until
   they succeed. A compensation that can fail requires a compensation for the compensation, and
   that recursion has to bottom out in a retry loop plus an alert.
2. **Order steps by reversibility**: reversible first, then the *pivot* (the step after which the
   saga will only ever go forward), then retriable-only steps. Once the pivot commits, you drive
   forward, never back.
3. **The saga's state must be durable before any step executes**, or a crash leaves effects in
   the world with no record that a saga owned them.

### 2.4 Orchestration versus choreography

**Orchestration**: a central saga coordinator holds the state machine and calls each service.
Explicit, testable, easy to observe; it introduces a component that knows about every
participant, and can become a distributed monolith if it also absorbs business rules.

**Choreography**: each service reacts to events and emits the next event. No central component
and low coupling on paper; but the process exists nowhere as a written artefact, cycles are easy
to create by accident, and answering "why is order 12345 stuck?" means reconstructing the flow
from logs across seven services.

The defensible default is **orchestration for anything with more than three steps or any
compensation logic**, choreography for simple fan-out notification. The deciding question is
SE-521's: *where should this knowledge live?* A refund policy is business knowledge and belongs
in one place; "the invoicing service cares when an order ships" is local knowledge and belongs in
the invoicing service.

### 2.5 The dual-write problem and the outbox

Here is the bug that appears in almost every first attempt at an event-driven system:

```python
def place_order(order):
    db.insert(order)                     # local transaction commits
    broker.publish(OrderPlaced(order))   # ... and then the process dies
```

Two writes to two systems with no atomicity between them. Crash in the middle and you have an
order nobody downstream knows about. Reverse the order and you can publish an event for an order
that was never persisted. Wrap them in 2PC and you have coupled your database's availability to
your broker's.

> **Transactional outbox.** Write the event into an `outbox` table *in the same local
> transaction* as the business change. A separate relay reads the outbox and publishes to the
> broker, marking rows as sent.

Now there is exactly one atomic write, to one system. The relay may publish a message twice (it
can crash after publishing and before marking), so delivery is **at-least-once** — which is fine,
because §2.7 makes consumers idempotent. The relay can poll the table, or tail the database's
replication log (**change data capture**, e.g. Debezium reading the WAL), which avoids polling
latency and load.

The inverse pattern, the **inbox**, dedupes on the consumer side: record the processed message ID
in the same transaction as the effect. Outbox plus inbox gives you end-to-end effectively-once
semantics built entirely out of local transactions. This is the single most valuable pattern in
this lesson.

### 2.6 Isolation: the part sagas do not give you

A saga's intermediate states are **visible**. Between "reserve inventory" and "charge card",
another transaction can read a world where inventory is reserved for an order that will never
exist. Garcia-Molina and Salem called these **dirty reads**; the practical taxonomy is:

- **Lost updates**: two sagas write the same field and one overwrites the other's change.
- **Dirty reads**: a saga reads state that a later compensation will undo.
- **Fuzzy / non-repeatable reads**: a saga reads the same record twice and sees different values.

The standard countermeasures (Richardson's catalogue, itself derived from the saga literature):

| Countermeasure | Mechanism |
|---|---|
| **Semantic lock** | Mark the record with an in-progress flag (`payment_state = PENDING`); other sagas either wait or fail fast. Reintroduces locking, but at business granularity and without holding a database transaction open. |
| **Commutative updates** | Design operations to commute (`credit` / `debit` rather than `set_balance`) so ordering does not matter — L07's CALM/CRDT insight applied to sagas. |
| **Pessimistic view** | Reorder steps so the risky read happens after the value is safe. |
| **Reread value** | Re-read and verify the record has not changed before writing (optimistic concurrency, via a version check). |
| **Version file** | Record operations and reorder them at apply time, tolerating out-of-order arrival. |
| **By value** | Route by risk: a real ACID transaction for high-value requests, a saga for low-value ones. |

Notice that the last one is an explicit admission: sometimes the right answer *is* a distributed
transaction, and the engineering judgement is about *which requests* warrant the cost.

### 2.7 Idempotence

> An operation is **idempotent** if applying it more than once has the same effect as applying it
> once.

Since retries are unavoidable (L01: you cannot distinguish a slow node from a dead one), every
mutating handler in a distributed system must be idempotent, or you will eventually double-charge
someone.

Some operations are **naturally idempotent**: `SET x = 5`, `DELETE /orders/7`, a `PUT` of a full
resource. Some are **naturally not**: `x += 1`, "send email", "charge card", "append to list".
The techniques for the second group:

1. **Idempotency keys.** The *client* generates a unique key per logical operation and sends it
   with every retry. The server records `(key → result)` and returns the stored result on replay.
   Critically, the key must be recorded **in the same transaction as the effect** — a separate
   "have I seen this key?" table is itself a dual write and reintroduces the very bug you were
   fixing. Stripe's API is the reference design; note also that it must handle the *concurrent*
   duplicate (two retries in flight at once), which needs a unique constraint on the key, not a
   read-then-write check.
2. **Natural keys / deduplication on business identity.** `INSERT … ON CONFLICT DO NOTHING`
   keyed on `(order_id, line_no)`.
3. **Conditional writes / version numbers.** `UPDATE … WHERE version = 7`; a replayed write fails
   the predicate and is a no-op. This gets you lost-update protection at the same time.
4. **Sequence numbers per sender.** The receiver tracks the highest contiguous sequence number
   seen and discards anything at or below it — TCP's mechanism, and the one used inside
   replication protocols (L03).
5. **Making the operation itself a fact.** Instead of `balance += 10`, record the *event*
   `credit(txn_id, 10)` and derive the balance. Duplicate events with the same `txn_id` collapse.
   This is L07's idempotent merge in a different dress, and it is why event sourcing composes so
   well with unreliable delivery.

**Retention.** Every dedup mechanism needs a policy for how long keys are kept. Too short and a
delayed retry after the window duplicates the effect; too long and the table grows without bound.
State the window explicitly (24 hours is a common choice), make the client's retry deadline
shorter than the window, and *reject* requests whose key is older than the window rather than
silently treating them as new.

**Retries need discipline too**: exponential backoff with full jitter to avoid the synchronised
retry storms of L09, a bounded attempt count, a dead-letter queue for what exhausts it, and the
awareness that retrying a request that is merely *slow* adds load to an already-degraded system —
the retry amplification failure covered in L09.

## 3. Construction: a saga engine with an outbox and idempotent handlers

Build this incrementally in `mpse/ds701/l08/`, reusing the fault-injection harness from L01. Use
SQLite as each "service's" local database so that local transactions are real.

**Stage 1 — the naive version, and watch it break.** Write `place_order()` exactly as in §2.5:
insert the order, then publish. Run it under the L01 harness with a crash injected between the
two statements. Assert that the consumer never sees the event while the order exists. You now
have a failing test that names the dual-write problem.

**Stage 2 — the outbox.** Add an `outbox(id, aggregate_id, type, payload, created_at, sent_at)`
table. Move the publish into an `INSERT` inside the same `BEGIN … COMMIT` as the order insert.
Re-run Stage 1's crash test: either both rows exist or neither does.

**Stage 3 — the relay.** Write a poller that selects unsent outbox rows ordered by `id`, publishes
each, then sets `sent_at`. Inject a crash *between* publish and mark. Observe a duplicate
delivery. Do not fix it here — this is the at-least-once guarantee working as designed.

**Stage 4 — the inbox.** On the consumer, create `inbox(message_id PRIMARY KEY, processed_at)`.
Process a message by, in one transaction, inserting the ID (failing on conflict) and applying the
effect. Re-run Stage 3; the duplicate is now absorbed. Write the test that proves the effect
happened exactly once by counting rows, not by trusting logs.

**Stage 5 — idempotency keys at the edge.** Add an HTTP-shaped entry point taking an
`Idempotency-Key` header. Store `(key, request_hash, response_body)` with a unique constraint on
`key`, written in the same transaction as the order. Test three cases: a sequential retry returns
the stored response; a *concurrent* retry (two threads) results in exactly one order; a reused key
with a *different* body returns a 422 rather than the stale response.

**Stage 6 — the saga.** Define a saga as an ordered list of steps, each `(action, compensation)`,
with a durable `saga_state(saga_id, step_index, status, payload)` row written before each step
runs. Implement `advance()` so that it is safe to call after a crash at any point: it reads the
persisted step index and continues. Model an order saga — reserve inventory, charge payment,
create shipment — and make the actions call the Stage 5 endpoint so that re-execution after a
crash is harmless.

**Stage 7 — compensation.** Make payment fail. Assert that compensations run in reverse order,
that they are idempotent (invoke each twice in the test), and that a crash *during* compensation
resumes correctly on restart. Add a `pivot` marker after which the engine refuses to compensate
and instead retries forward.

**Stage 8 — the isolation anomaly.** Run two sagas concurrently against the same inventory item
and produce a lost update. Then fix it twice: once with a semantic lock (a `reserved_by` column),
once with a commutative update (`quantity = quantity - n WHERE quantity >= n`). Measure the
throughput difference under contention and write two paragraphs on the trade-off. This is the
deliverable that shows you understood §2.6.

## 4. Failure modes

- **Reaching for a distributed transaction to fix a bad boundary.** If two services must change
  together atomically and often, they are one service, or the aggregate boundary is wrong. Fix
  the boundary before reaching for 2PC.
- **A dedup table written outside the effect's transaction.** The most common way an "idempotent"
  handler is not.
- **Server-generated idempotency keys.** A key generated by the receiver changes on every retry
  and dedupes nothing. It must come from the client and be stable across retries.
- **Compensations that assume the forward step completed cleanly.** They must handle "partially
  done" and "not done at all" — write them to be idempotent and total.
- **Unbounded retries without jitter.** Turns a brief degradation into a self-sustaining overload
  (L09).
- **Choreography with no process view.** Nobody can answer where an order is stuck. If you choose
  choreography, you owe the system a correlation ID and a trace (L10).
- **Assuming the broker gives you exactly-once.** Broker-level "exactly-once" is exactly-once
  *within that broker's own boundaries*; the moment your handler touches an external system, it
  is your dedup logic or nothing.
- **No retention policy on dedup state.** Either an unbounded table, or a silent correctness bug
  when a delayed retry lands after expiry.

## 5. Exercises

### Warm-up (30 min)

1. Classify as naturally idempotent or not, and give the fix for each that is not:
   `PUT /users/7`, `POST /orders`, `DELETE /sessions/x`, `balance += 10`, `send_email()`,
   `append(log, entry)`, `UPDATE t SET status='PAID' WHERE id=7`.
2. State the atomic-commit problem and explain why unanimity, not majority, is required.
3. In one paragraph, explain the in-doubt window and what a participant may and may not do while
   it is in that window.

### Core (3 h)

4. Complete Stages 1–5 of §3. Your test suite must demonstrate, with assertions rather than
   prose, that (a) the dual write is eliminated, (b) duplicate delivery occurs, and (c) it is
   absorbed exactly once by the consumer.
5. Complete Stages 6–7. Produce a state diagram of your saga, including the pivot, and a table
   mapping each step to its compensation and to what "partially done" means for that step.
6. Take a system you know (or the checkout flow above) and write a **boundary analysis**: which
   invariants must hold atomically, which aggregate owns each, and which cross-boundary
   invariants you are choosing to enforce eventually. For each eventual one, state the window
   during which it may be violated and what a user sees during that window.
7. Implement Stage 8 both ways and write up the measurement.

### Challenge

8. Implement Paxos Commit on top of your L05 Raft implementation: one consensus instance per
   participant vote, with the decision derivable by any participant without the coordinator. Kill
   the coordinator after all votes are cast and show that the participants still terminate.
   Compare its latency to your 2PC implementation and account for the extra round trips.
9. Build a **saga log analyser**: given the durable saga state and the outbox/inbox tables from a
   crashed run, reconstruct which sagas are incomplete, which effects are orphaned, and what a
   recovery process should do about each. This is the tooling that turns a saga system from
   theoretically recoverable into operationally recoverable.

## 6. Self-check

1. Why is atomic commit strictly harder to make available than consensus?
2. What exactly is a participant promising when it votes `YES`?
3. What does `XA_HEURHAZ` mean, and what does its existence tell you?
4. Why is 3PC not used, and what is used instead?
5. What does a saga guarantee, and which ACID letter does it explicitly not provide?
6. Explain the dual-write problem and how the outbox eliminates it — being precise about how many
   systems are written to atomically.
7. Give five techniques for making a non-idempotent operation idempotent.
8. Why must the idempotency key be persisted in the same transaction as the effect?
9. Name four saga isolation countermeasures and the anomaly each addresses.
10. What is a saga pivot and why does it matter?

## 7. Primary sources

- **Gray & Lamport, "Consensus on Transaction Commit" (TODS 2006)** — the definitive treatment of
  2PC's relationship to consensus, and Paxos Commit.
- **Garcia-Molina & Salem, "Sagas" (SIGMOD 1987)** — the original paper; short, and still the
  clearest statement of the idea.
- Skeen & Stonebraker, "A Formal Model of Crash Recovery in a Distributed System" (TSE 1983) —
  the blocking result.
- Bernstein, Hadzilacos & Goodman, *Concurrency Control and Recovery in Database Systems* (1987),
  chapter 7 — free online, and the standard reference for atomic commit.
- Kleppmann, *Designing Data-Intensive Applications*, chapter 9, "Distributed Transactions in
  Practice".
- Richardson, *Microservices Patterns*, chapters 4–5 — the saga and countermeasure catalogue.
- Helland, "Life beyond Distributed Transactions: An Apostate's Opinion" (CIDR 2007) — read this
  one for the argument, not the mechanics.
- Corbett et al., "Spanner: Google's Globally-Distributed Database" (OSDI 2012) — 2PC done
  correctly, over Paxos groups.

---

**Previous:** [L07](L07-crdts.md) · **Next:**
[L09 — Failure Detection, Timeouts, and Resilience](L09-failure-detection-and-resilience.md)
