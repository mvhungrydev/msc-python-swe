# DS-701 · Lesson 04 — Consistency Models

**Estimated study time:** 5 hours
**Prerequisites:** L01–L03; PY-601 L01 §2.4

---

## 1. Orientation

"Is it consistent?" is not a question. There are more than fifty published consistency models,
they form a partial order, and a system's marketing material and its actual guarantee are
frequently different documents.

The practical problem this causes: an engineer builds on an assumption the system does not
provide, and the resulting bug appears once a month under load, is unreproducible, and is
eventually attributed to something else.

This lesson gives you the vocabulary to state precisely what a system provides, the ability to
*test* the claim, and the judgement to choose. The single most valuable output is being able to
say, of any system you use: **"it provides X, which means Y is possible, and here is the code
we wrote to handle Y."**

## 2. Theory

### 2.1 The hierarchy

Strongest at the top; each implies the ones below it.

```
                Strict serializability  (= linearizability + serializability)
                          │
        ┌─────────────────┴─────────────────┐
   Linearizability                    Serializability
   (single object,                    (transactions,
    real-time order)                   some serial order)
        │                                   │
   Sequential consistency            Snapshot isolation
        │                                   │
   Causal consistency ─────────────── Read committed
        │
   PRAM / FIFO
        │
   Eventual consistency
```

Two independent axes are collapsed in that picture, and separating them is the key:

- **Single-object versus multi-object.** Linearizability is about one register; serializability
  is about transactions over many objects. They are *not* comparable — a system can have one
  without the other.
- **Real-time versus not.** Linearizability respects real-time order (if A completes before B
  starts, A is ordered first); sequential consistency does not.

**Strict serializability** = both. It is what Spanner provides, and it is the strongest useful
model.

### 2.2 Linearizability

From PY-601 L01 §2.4, now in the distributed setting.

> Each operation appears to take effect **instantaneously at some point between its invocation
> and its response**, and that ordering is consistent with a correct sequential execution.

The consequences that matter:

- **It respects real time.** If your write returns before my read starts, my read sees your
  write. This is what makes the system feel like a single machine.
- **It is composable** (a local property): if every object is linearizable, the system is.
  This is why it is the model to reach for when you can afford it.
- **It requires coordination.** Under a partition, you must refuse service on the minority
  side (CAP, L01 §2.5), and even without a partition you pay the latency of coordination
  (PACELC's "else").

Where you genuinely need it: **uniqueness constraints** (one username, one seat), **locks and
leader election**, **cross-channel consistency** (the user is told "done" through one channel
and then checks through another), and anything where a stale read causes an irreversible
action.

Where you do not: analytics, timelines, caches, recommendations, counters, and most read paths
in most applications.

**Testing it**: record a history of concurrent invocations and responses, then search for a
valid linearization. This is NP-complete in general but tractable for short histories, and it
is what Jepsen's `knossos`/`elle` do. **Building this checker is the exercise that makes the
definition real** (§3).

### 2.3 Sequential consistency

> Operations appear in *some* sequential order consistent with each process's program order.

It drops the real-time requirement. So: your write completes, my read starts afterwards, and
my read may still return the old value — as long as *some* consistent global order exists.

It is **not composable**, which is a real practical defect: two sequentially consistent objects
composed do not give a sequentially consistent system. That non-composability is the main
reason linearizability is preferred as a specification.

Mostly of theoretical interest for storage systems, but it is the model most hardware memory
models are described against (PY-601 L03 §2.1).

### 2.4 Causal consistency

> Operations that are causally related (L02 §2.2) are seen in the same order by everyone.
> Concurrent operations may be seen in different orders.

**This is the sweet spot**, and the most under-used model in practice.

Why: Attiya, Ellen & Morrison and others have shown that causal consistency is **the strongest
model achievable in an always-available, partition-tolerant system**. You can have causality
without coordination; you cannot have more.

What it prevents:

- A reply appearing before the message it replies to.
- Reading your own write and then not reading it.
- The consistent-prefix anomaly (L03 §2.4).

What it permits: two concurrent writes to the same key being seen in different orders by
different readers, which you must resolve (L07).

Implementations: COPS, Eiger, Bolt-on causal consistency, MongoDB's causal-consistency sessions,
Azure Cosmos DB's session level. The mechanism is dependency tracking — each write carries the
versions it depends on, and a replica delays applying a write until its dependencies are
present.

The cost: metadata. Tracking full causality needs vector clocks (L02 §2.4) or explicit
dependency lists, and both grow.

**Session guarantees** (L03 §2.4) — read-your-writes, monotonic reads, monotonic writes, and
writes-follow-reads — are a *per-client* approximation of causal consistency, and together they
give it for a single session. That is much cheaper and is what most systems that claim
"causal" actually implement.

### 2.5 Eventual consistency

> If updates stop, all replicas eventually converge.

Notice what it does not say: *anything about what you read in the meantime, or when
"eventually" is.* You may read arbitrarily stale data; you may read newer data then older; two
successive reads may disagree.

It is the weakest useful model and it is the default for AP systems (Dynamo, Cassandra with low
consistency levels, DNS, most caches).

**Strong eventual consistency** (SEC) adds a real guarantee: **replicas that have received the
same set of updates are in the same state**, regardless of order. That is what CRDTs provide
(L07), and it removes the need for conflict resolution. The difference between EC and SEC is
substantial and the terms are used interchangeably by people who should know better.

The practical question for any eventually-consistent system: **how long is "eventually", at
p99?** It is measurable (write, then poll a replica until it converges), it is almost never
measured, and it is what determines whether the model is acceptable for a given use.

### 2.6 Transaction isolation levels

The database-side vocabulary, which is genuinely a different axis. From ANSI SQL and Berenson
et al.'s critique:

| Level | Prevents | Permits |
|---|---|---|
| Read uncommitted | — | dirty reads |
| Read committed | dirty reads | non-repeatable reads, phantoms |
| Repeatable read | non-repeatable reads | phantoms |
| Snapshot isolation | most anomalies | **write skew** |
| Serializable | everything | — |

Points that matter:

- **The ANSI definitions are defined by which anomalies they prevent**, which Berenson et al.
  (1995) showed is ambiguous and implementation-dependent. Adya's (1999) formulation in terms
  of dependency graphs is the rigorous version.
- **Most databases' "repeatable read" is actually snapshot isolation.** PostgreSQL's is.
- **Snapshot isolation permits write skew**, and this is the anomaly worth knowing:

  > Two transactions read overlapping data, make disjoint writes based on what they read, and
  > both commit. Each was valid alone; together they violate an invariant.
  >
  > *Example*: a hospital requires at least one doctor on call. Alice and Bob are both on
  > call. Both simultaneously check "is someone else on call?" (yes), and both take themselves
  > off. Now nobody is on call. Neither transaction wrote what the other read, so snapshot
  > isolation permits it.

  Fixes: `SELECT ... FOR UPDATE` to materialize the conflict; serializable isolation; or a
  constraint the database can enforce.

- **Serializable is available and cheaper than its reputation.** PostgreSQL's SSI
  (serializable snapshot isolation) has a modest overhead and aborts conflicting transactions
  rather than blocking. Most applications that "cannot afford" serializable have never measured
  it.

**Serializability is not linearizability.** Serializable transactions can be ordered in *any*
serial order — including one that puts a transaction before another that completed earlier in
real time. Strict serializability adds the real-time constraint. This distinction matters for
exactly the cross-channel case: a user commits a transaction, is told it succeeded, and then
reads through a different path and does not see it.

### 2.7 Choosing, and stating

The procedure:

1. **What breaks if a read is stale?** If the answer is "a user sees an old number for 200 ms",
   eventual is fine. If it is "we sell the same seat twice", you need linearizability *for that
   operation*.
2. **Choose per operation, not per system.** A single system can serve most reads eventually
   and route the few that need it through a linearizable path. This is the design most systems
   should have and few do.
3. **State the model in the interface.** `get(key, consistency=EVENTUAL)` is honest;
   `get(key)` with a footnote is not.
4. **Test the claim.** §3.
5. **Measure the staleness.** For anything eventual, p50/p99/max convergence time, as a
   monitored metric.
6. **Handle the anomalies the model permits.** Explicitly, in code, with a comment naming the
   anomaly.

**The failure this prevents**: a system that provides eventual consistency, is described in
the design document as "consistent", and has application code assuming linearizability. That
combination produces bugs that are individually inexplicable and collectively a pattern nobody
sees.

## 3. Construction: a linearizability checker

The centrepiece of this lesson, and the thing that turns consistency models from vocabulary
into a testable property.

**Stage 1 — record histories.** Instrument your L03 store so that every operation records:

```python
@dataclass(frozen=True)
class Event:
    process: int
    kind: Literal["invoke", "ok", "fail", "info"]   # info = unknown outcome
    op: Literal["read", "write", "cas"]
    value: Any
    time_ns: int
```

The `info` case is essential: a timed-out operation has an **unknown** outcome (L01 §2.7) and
the checker must consider both possibilities — it may or may not have taken effect.

**Stage 2 — the checker.** Search for a linearization:

- Maintain a model (a simple register or key-value map).
- At each step, consider every operation that is *pending* (invoked, not yet returned) or whose
  return has been observed and whose linearization point could be now.
- Try linearizing each; recurse; backtrack on failure.
- Prune: an operation cannot be linearized before its invocation or after its response.

This is exponential in the worst case (it is NP-complete), so:

- Keep histories short (10–20 operations per checked window).
- Use the Wing–Gong algorithm or Lowe's improvements for pruning.
- Partition by key — operations on independent keys can be checked independently, which is a
  large win.

**Stage 3 — verify the checker.** Feed it histories you *know* are linearizable and ones you
know are not:

```
# linearizable
P1: invoke write(1) ... ok
P2:                       invoke read ... ok(1)

# NOT linearizable
P1: invoke write(1) ................. ok
P2:      invoke read ... ok(0)     invoke read ... ok(1)   invoke read ... ok(0)
```

The third read returning 0 after the second returned 1 cannot be linearized. **A checker you
have not seen reject a bad history is a checker you cannot trust.**

**Stage 4 — check your store.** Run your L03 replicated store under load with fault injection,
record the history, and check it. Then:

- Check your leader-based store with reads from followers → **should fail** (stale reads).
- Check with reads from the leader only → should pass.
- Check your quorum store with W+R>N → **should fail**, and finding the exact history that
  fails is the payoff of L03 §2.3's argument.

**Report the failing histories.** They are the concrete evidence that the guarantee is weaker
than the folk claim.

**Stage 5 — weaker models.** Implement checkers for:

- **Read-your-writes**: per session, a read after a write in the same session must see it or
  later.
- **Monotonic reads**: within a session, versions never go backwards.
- **Causal consistency**: build the dependency graph from the recorded causality and verify
  every replica's apply order is a linear extension of it.

Check your store against each and find the strongest model it actually satisfies.

**Stage 6 — implement a stronger model.** Add a linearizable read path to your store (read from
the leader with a fresh quorum check, or route through a consensus layer once you have L05).
Verify with the checker. Then **measure the latency cost** of linearizable versus eventual
reads, and the availability cost during a partition.

That trade curve — latency and availability against consistency — is the deliverable.

**Stage 7 — write skew.** In a real database, construct the doctors-on-call anomaly under
snapshot isolation (PostgreSQL's `REPEATABLE READ`). Then fix it three ways: `FOR UPDATE`,
`SERIALIZABLE`, and a check constraint. Measure the throughput of each under contention.

**Stage 8 — the statement.** For your store, write the consistency guarantee as it would appear
in documentation: the model, the anomalies it permits with an example of each, the measured
staleness distribution, and the per-operation exceptions.

## 4. Failure modes

- **"Is it consistent?"** as a question. Name the model.
- **Assuming a system provides what its marketing says.** Test it.
- **Confusing serializability with linearizability.** Different axes.
- **Confusing eventual with strong eventual consistency.**
- **Believing `W+R>N` gives linearizability.** L03 §2.3.
- **"Repeatable read" assumed to prevent write skew.** It is snapshot isolation and it does
  not.
- **Never measuring "eventually".** The p99 convergence time is the number that decides
  whether the model is acceptable.
- **One consistency model for a whole system.** Choose per operation.
- **Cross-channel violations.** The user is told "done" via one path and checks via another;
  this is exactly the real-time requirement, and it is why linearizability rather than
  serializability is what you need for it.
- **An unverified checker.** If it has never rejected a bad history, it proves nothing.
- **Not handling the `info` (unknown) case** in a history. A timed-out write may have applied.

## 5. Exercises

### Warm-up (30 min)

**W1.** Write two histories on a single register: one linearizable, one not. Prove each by
exhibiting or ruling out a linearization order.

**W2.** For three systems you use, find the documented consistency model and the anomalies it
permits. Where the documentation is vague, say so.

**W3.** Construct the write-skew anomaly on paper, and give three fixes.

### Core (3.5 h)

**C1 — The linearizability checker.** Complete §3 stages 1–3. Deliverable: the history format
including the unknown case, the checker with pruning, and the verification against known-good
and known-bad histories including at least five of each.

**C2 — Check your store.** Complete §3 stages 4–5. Deliverable: histories from your store under
fault injection, the failing histories with the exact violation identified, the weaker-model
checkers, and a statement of the strongest model your store actually satisfies.

**C3 — The trade curve.** Complete §3 stage 6. Deliverable: a linearizable read path, verified;
and measured latency (p50/p99) and availability-during-partition for linearizable versus
eventual reads. Plot the trade.

**C4 — Write skew.** Complete §3 stage 7. Deliverable: the anomaly reproduced in a real
database, three fixes implemented, throughput under contention for each, and a recommendation.

### Challenge

**X1.** Read a Jepsen report on a system you use, in full. Reproduce one of its findings in
your own laboratory (or with the real system in containers). Report whether the issue is
fixed, and what the vendor's response was. Then write 800 words on what the report's
methodology does that ordinary testing does not.

**X2.** Implement causal consistency properly in your store: dependency tracking on writes,
delayed application until dependencies are satisfied, and a causal-consistency checker. Then
measure: the metadata overhead as a function of the number of writers, the added write latency,
and the delay before a write becomes visible on a remote replica. Compare with the same store
run eventually consistent and linearizably, and report the three-way trade. Write 800 words on
why causal consistency is the strongest always-available model and why it is nonetheless rare
in practice.

## 6. Self-check

1. Draw the consistency hierarchy and name the two axes it collapses.
2. Define linearizability and give the three properties that follow.
3. Why is linearizability composable and sequential consistency not?
4. Why is causal consistency the strongest always-available model?
5. Distinguish eventual from strong eventual consistency.
6. Give the isolation levels and the anomaly each permits, and describe write skew concretely.
7. Distinguish serializability from linearizability, and give the case where the difference
   is user-visible.
8. Give the six steps of choosing and stating a consistency model.

## 7. Primary sources

- **Herlihy & Wing, "Linearizability: A Correctness Condition for Concurrent Objects"
  (TOPLAS 1990).**
- Lamport, "How to Make a Multiprocessor Computer That Correctly Executes Multiprocess
  Programs" (1979) — sequential consistency.
- Ahamad et al., "Causal Memory" (1995); Lloyd et al., "Don't Settle for Eventual" (COPS,
  SOSP 2011).
- **Berenson et al., "A Critique of ANSI SQL Isolation Levels" (SIGMOD 1995).**
- Adya, "Weak Consistency: A Generalized Theory and Optimistic Implementations for Distributed
  Transactions" (MIT PhD, 1999).
- Bailis et al., "Highly Available Transactions: Virtues and Limitations" (VLDB 2014).
- Viotti & Vukolić, "Consistency in Non-Transactional Distributed Storage Systems"
  (Computing Surveys, 2016) — the survey of 50+ models.
- Kingsbury, the Jepsen analyses and the consistency-model reference at jepsen.io/consistency.

---

**Previous:** [L03](L03-replication-and-quorums.md) · **Next:**
[L05 — Consensus: Paxos, Raft, and FLP](L05-consensus.md)
