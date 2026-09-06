# SE-521 · Lesson 07 — Aggregates, Transactions, and Consistency Boundaries

**Estimated study time:** 4 hours
**Prerequisites:** L05, L06

---

## 1. Orientation

Every system makes promises about consistency, and most systems make them by accident.

"An order's total always equals the sum of its lines." "A customer's credit limit is never
exceeded." "Every shipment references an existing order." Are these guaranteed by the
database, by a transaction, by a check that runs before the write, or by hope? For most
systems the answer varies rule by rule, nobody has written it down, and the rules that are
protected by hope fail under concurrency.

An **aggregate** is DDD's answer: the unit of consistency. Getting aggregate boundaries right
is the single most consequential modelling decision, because it determines what you can
guarantee atomically and what must be eventually consistent — and that determines your
transaction shape, your locking, your scaling limits, and your failure modes.

## 2. Theory

### 2.1 Aggregates

An **aggregate** is a cluster of objects treated as a single unit for data changes. It has:

- A **root** entity, the only object outside code may hold a reference to.
- A **boundary** — what is inside.
- **Invariants** that must hold at the end of every transaction.

The rules:

1. **Reference other aggregates by identity only.** An `Order` holds a `CustomerId`, not a
   `Customer` object. This keeps the object graph small and makes the boundary real.
2. **One transaction, one aggregate.** A single transaction modifies exactly one aggregate
   instance. Changes to others happen in later transactions, driven by events.
3. **Invariants inside an aggregate are enforced immediately; invariants across aggregates
   are eventually consistent.** This is the whole point.
4. **Load and save the whole aggregate.** The repository deals in aggregate roots, not in
   internal parts.

Rule 2 is the one people resist and it is the one that matters. It forces you to decide what
must be immediately consistent, which forces you to *know* your consistency requirements
rather than getting them from whatever the ORM's cascade rules happen to do.

### 2.2 Sizing the boundary

The trade-off is exactly:

- **Bigger aggregate** → more invariants enforced immediately → more contention, larger
  loads, worse concurrency, and eventually a scaling wall.
- **Smaller aggregate** → better concurrency and simpler loads → more invariants become
  eventually consistent, and you must handle the window.

The procedure, which you can actually run:

1. **List the invariants.** Precisely, as statements that must be true.
2. **For each, ask: must this be true at every instant, or is a brief violation acceptable
   if it is detected and corrected?** This is a *business* question. Ask the business.
3. **The invariants requiring instant truth define the boundaries** — the objects they span
   must be in one aggregate.
4. **Everything else is eventual**, and you must design the correction: a compensating
   action, a reconciliation job, or an accepted overdraft.

The canonical example: *"an order's total may not exceed the customer's credit limit."*
Naively this puts `Customer` and all their `Order`s in one aggregate — meaning every order
placement locks the customer, and a customer with 10,000 orders loads all of them.
Unworkable.

Ask the business the step-2 question and the answer is usually: "a small, brief overdraft is
acceptable if we detect it and act." Now `Order` and `Customer` are separate aggregates, the
check is best-effort at placement time, and a reconciliation process handles violations.
That is not a compromise of correctness; it is *the actual requirement*, which the naive
model got wrong by assuming.

Vernon's rules of thumb, which hold up: prefer small aggregates; use eventual consistency
outside the boundary; and if you find yourself needing to modify two aggregates in one
transaction, the boundary is probably wrong — or the "invariant" is not really one.

### 2.3 Transactions and the Unit of Work

A **Unit of Work** tracks the objects affected by a business transaction and coordinates
writing them out, plus the transaction boundary.

```python
class UnitOfWork(Protocol):
    orders: OrderRepository
    def __enter__(self) -> "UnitOfWork": ...
    def __exit__(self, *exc) -> None: ...      # rollback if not committed
    def commit(self) -> None: ...
    def rollback(self) -> None: ...
```

Where it belongs: the **application layer** (L05 §2.4). The domain knows nothing of
transactions; the adapter cannot see the scope of a use case.

Three properties of a correct implementation:

- **Rollback by default.** `__exit__` rolls back unless `commit()` was called. A forgotten
  commit must not silently persist a partial change.
- **The transaction boundary equals the use case boundary.** One use case, one transaction,
  one aggregate.
- **No external I/O inside it.** L05 §3 step 5 — a network call inside a transaction holds
  locks across an unbounded wait and creates an atomicity gap.

### 2.4 Concurrency: the check-then-act problem

Two requests reserve the last item simultaneously. Both read `stock = 1`, both check, both
decrement. This is a **lost update**, and it is the default behaviour of naive code under
any isolation level below serializable.

The three mechanisms:

**Optimistic concurrency.** Every aggregate has a version. Read it with the aggregate; on
write, `UPDATE ... WHERE id = ? AND version = ?`; if zero rows are affected, someone else
won — retry or fail.

```python
def save(self, order: Order) -> None:
    rows = self._exec(
        "UPDATE orders SET data=%s, version=version+1 WHERE id=%s AND version=%s",
        (dump(order), order.id, order.version))
    if rows == 0:
        raise ConcurrentModification(order.id)
```

Best default: no locks held across think time, scales well, and the failure is explicit.
Costs: the caller must handle retry, and under high contention it degrades badly (live-lock
of retries).

**Pessimistic locking.** `SELECT ... FOR UPDATE` holds a row lock for the transaction.
Correct, simple, and it serializes access to the aggregate — which is fine for low contention
and catastrophic for a hot row. Also introduces deadlock risk, which must be handled by
consistent lock ordering.

**Atomic operations.** Push the check into the database:
`UPDATE stock SET qty = qty - 1 WHERE sku = ? AND qty >= 1`. Zero rows means insufficient
stock. No read-modify-write, no lock, no retry. **When it is expressible, this is the best
option** and it is frequently overlooked because the logic ends up in SQL rather than in the
domain — a real trade-off worth stating in an ADR.

Isolation levels are DI-721 L04's subject; the thing to know here is that **read committed
(the default in PostgreSQL and most systems) does not prevent lost updates**, so if you are
not doing one of the three above, you have a race.

### 2.5 Crossing aggregates: domain events

Since one transaction touches one aggregate, cross-aggregate work is driven by events.

```python
result = order.cancel(now, policy)          # returns new state + events
repo.save(result.order); uow.commit()       # transaction 1: one aggregate
publish(result.events)                       # then: everything else
```

The gap between commit and publish is the problem, and it has exactly two failure modes:
publish before commit (the event describes something that did not happen) or commit before
publish (the event may be lost). Neither is acceptable, and you cannot fix it by reordering.

**The outbox pattern** is the fix:

1. In the same transaction as the state change, insert the event into an `outbox` table.
   Atomic — either both or neither.
2. A separate relay reads the outbox and publishes, marking rows as sent.
3. The relay may publish the same event twice (crash between publish and mark), so delivery
   is **at-least-once** and consumers must be **idempotent**.

This is the standard, correct solution and it is worth implementing once by hand so you
understand what a "transactional outbox" in a library is doing (DI-721 L08 goes deeper;
change data capture is the alternative implementation).

**Idempotency at the consumer**: a processed-message table keyed by event id, checked and
inserted in the same transaction as the effect. Or design the effect to be naturally
idempotent (setting a status is idempotent; incrementing a counter is not).

### 2.6 Sagas: multi-step business transactions

When a business operation spans several aggregates or services and cannot be one
transaction, model it as a **saga**: a sequence of local transactions, each with a
**compensating action** for rollback.

```
Reserve stock  →  Charge payment  →  Create shipment
     ↓ fail           ↓ fail
  (nothing)      Release stock     Release stock + Refund payment
```

Two shapes:

- **Choreography** — each step publishes an event; the next step listens. No central
  coordinator. Simple for two or three steps; beyond that nobody can see the whole flow, and
  debugging means reconstructing it from logs.
- **Orchestration** — a coordinator process holds the state machine and issues commands.
  More infrastructure; the flow is visible in one place; and you can query "where is this
  saga?" Prefer it beyond three steps.

What sagas do *not* give you: isolation. Intermediate states are visible — stock is
reserved but not paid; a user may see it. You must decide whether that is acceptable, and
often it requires a semantic lock ("pending") that the rest of the system understands.

**Compensation is not rollback.** You cannot un-send an email; you send an apology. You
cannot un-charge; you refund, and the customer sees both lines on their statement. Designing
compensations is a *business* design task, not a technical one, and asking "what does the
business do when step 3 fails after step 2 succeeded?" is a question that has an answer
today — usually a manual process — which you should learn before designing the automated
one.

### 2.7 Stating what you actually guarantee

The deliverable of this lesson, for any system:

| Invariant | Scope | Mechanism | Window | Detection | Correction |
|---|---|---|---|---|---|
| Order total = sum of lines | within aggregate | transaction | none | — | — |
| Credit limit not exceeded | across aggregates | best-effort check | ~seconds | nightly reconciliation | manual review |
| Every shipment has an order | across contexts | FK in same DB | none | — | — |
| Search index matches DB | across systems | async indexer | ~1s (p99 30s) | drift monitor | reindex |
| Stock never negative | within aggregate | atomic UPDATE | none | — | — |

Filling in this table for a real system takes a day and typically finds three rules that are
protected by nothing. That finding is worth the day.

## 3. Construction: sizing an aggregate

Take a domain with contention — inventory, seat booking, account balances, rate limits.

**Step 1 — list the invariants.** Precisely. Aim for ten.

**Step 2 — the instant-or-eventual question, asked properly.** For each, write the *business
consequence* of a brief violation: what happens, who notices, what does it cost, how is it
currently handled? "Two people book the same seat" has an answer today (someone is bumped
and compensated). That answer tells you which category the invariant is in.

**Step 3 — draw the boundaries** from the instant-consistency set. Then check the size: how
many objects does the largest aggregate load? How many concurrent writers will contend on
one instance? If a single aggregate instance is written more than a few times a second, you
have a hot spot and must revisit step 2.

**Step 4 — implement with optimistic concurrency.** Version column, conflict detection,
explicit `ConcurrentModification`. Then write the test that *proves* it: two threads or two
processes racing, asserting exactly one wins and the loser gets the error. Most systems have
never run this test.

**Step 5 — implement the eventual path.** The outbox, the relay, the idempotent consumer,
and — the part everyone skips — the **detection**: a reconciliation job that finds
violations of the eventually-consistent invariants and reports them. An eventual guarantee
with no detection is not a guarantee.

**Step 6 — a saga.** Take a three-step cross-aggregate operation. Implement it with
compensations. Test: failure at each step, compensation failure, and duplicate delivery of
each message. Report which of those your first implementation got wrong (it will be at
least one).

**Step 7 — the table.** Fill in §2.7 for your system, honestly. The rows with "hope" in the
Mechanism column are the deliverable.

## 4. Failure modes

- **Aggregates sized by the object graph** rather than by invariants — typically "everything
  reachable", producing enormous loads.
- **Modifying two aggregates in one transaction.** Usually means the boundary is wrong.
- **Holding object references across aggregates.** The boundary becomes fictional and lazy
  loading pulls in the world.
- **Check-then-act with no concurrency control.** Lost updates under load, invisible in
  testing.
- **Assuming read-committed prevents lost updates.** It does not.
- **Publishing events outside the transaction.** Lost or phantom events.
- **At-least-once delivery with non-idempotent consumers.** Double charges.
- **Sagas by choreography beyond three steps.** Nobody can see the flow.
- **Compensations that are not really possible.** "We'll roll back the email."
- **Eventual consistency with no detection or correction.** Silent divergence.
- **No stated consistency model.** §2.7 — the most common failure of all.

## 5. Exercises

### Warm-up (25 min)

**W1.** Find a check-then-act sequence in real code. Write the test that demonstrates the
lost update with two concurrent callers.

**W2.** Find a transaction that modifies two aggregates. Decide whether the boundary is
wrong or the invariant is not real.

**W3.** Find an event published outside its transaction. Describe both failure modes.

### Core (2.5 h)

**C1 — Aggregate sizing.** Complete §3, steps 1–4. Deliverable: the invariant list with the
business-consequence column, the boundary decision with the contention analysis, the
optimistic-concurrency implementation, and the passing race test.

**C2 — The outbox, end to end.** Complete §3 step 5: outbox table written in the same
transaction, a relay process, idempotent consumers, and a reconciliation job with alerting.
Tests: crash between commit and relay, relay crash after publish before mark, two relays
running concurrently, and consumer receiving a duplicate. Report which your first version
got wrong.

**C3 — A saga with compensations.** Complete §3 step 6. Include a failure at each step, a
compensation that itself fails, and a documented answer for what a human does then. Compare
choreography and orchestration for your case and recommend one.

**C4 — The consistency table.** Complete §3 step 7 for a real system. For every row whose
mechanism is "nothing", write the smallest change that would make the guarantee real, and
estimate its cost. Present the table as you would to a team.

### Challenge

**X1.** Read Vernon's "Effective Aggregate Design" (three-part paper, 2011) and Helland's
"Life Beyond Distributed Transactions" (2007). Write 1,500 words on Helland's claim that
almost-infinite scale forces entity-at-a-time programming with eventual consistency between
entities, and connect it to aggregate design. Then apply it: identify the point at which
your system's largest aggregate becomes a scaling limit and what you would do.

**X2.** Implement a seat-booking system with a genuinely hard invariant (no double booking)
under real concurrency. Try three mechanisms — optimistic, pessimistic, atomic UPDATE — and
measure throughput and failure rate under 10, 100, and 1,000 concurrent bookers. Report the
curves. Then add a hold/expiry mechanism (a semantic lock) and analyse the new failure
modes it introduces.

## 6. Self-check

1. Give the four rules of aggregate design.
2. State the aggregate-sizing trade-off and the four-step procedure.
3. Why is the credit-limit example the canonical one, and what does asking the business
   reveal?
4. Give three concurrency-control mechanisms with the cost of each.
5. Why does read-committed not prevent lost updates?
6. Describe the outbox pattern and what it requires of consumers.
7. Distinguish choreography from orchestration and say when to prefer each.
8. Why is compensation not rollback, and what does that imply about designing it?

## 7. Primary sources

- Vernon, "Effective Aggregate Design", parts I–III (2011). The clearest treatment
  available, and short.
- Evans, *Domain-Driven Design*, ch. 6.
- Helland, "Life Beyond Distributed Transactions: An Apostate's Opinion" (CIDR 2007).
- Garcia-Molina & Salem, "Sagas" (SIGMOD 1987) — the original.
- Kleppmann, *Designing Data-Intensive Applications*, ch. 7 (transactions) and ch. 9.

---

**Previous:** [L06](L06-domain-driven-design.md) · **Next:**
[L08 — Evolutionary Architecture and Fitness Functions](L08-evolutionary-architecture.md)
