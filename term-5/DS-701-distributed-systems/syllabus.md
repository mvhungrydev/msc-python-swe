# DS-701 — Distributed Systems

**Term:** 5 · **Credits:** 15 · **Nominal hours:** 140
**Prerequisites:** PY-601, SE-521, CS-621
**Co-requisite:** DI-721

---

## Driving question

> What can you guarantee when parts of your system are lying to you?

Not "failing" — *lying*. A node that is slow is indistinguishable from a node that is dead. A
network that drops a message is indistinguishable from one that delays it by ten minutes. A
clock that says 14:02:17 may be wrong by seconds. A response that never arrives may mean the
request was never received, or that it was executed twice.

Distributed systems is the discipline of building reliable behaviour on top of components that
are individually unreliable and that cannot tell you which failure they are experiencing. Its
results are mostly *impossibility* theorems and *trade-off* statements, and knowing them is
what separates a design you can defend from one that works until it does not.

## Learning outcomes

On completion you will be able to:

1. **State** the failure model a design assumes, and identify what breaks when the real world
   exceeds it.
2. **Reason** about time and causality without wall clocks: happens-before, Lamport and vector
   clocks, and hybrid logical clocks.
3. **Distinguish** the consistency models precisely — linearizability, sequential, causal,
   eventual — and state which one a system actually provides.
4. **Explain** CAP and PACELC correctly, including the ways both are routinely misused.
5. **Describe** how Paxos and Raft achieve consensus, and what FLP means for them.
6. **Design** replication and partitioning schemes with a stated durability and availability
   model.
7. **Apply** CRDTs where they fit and say where they do not.
8. **Build** systems that are correct under at-least-once delivery: idempotence, sagas, and
   the outbox.
9. **Detect** failure with timeouts and failure detectors, and reason about the accuracy
   trade-off.
10. **Instrument** a distributed system so that a failure is diagnosable, and test it with
    fault injection.

## Lessons

| # | Title | Hours |
|---|---|---|
| L01 | Failure Models, Fallacies, and What Is Impossible | 4 |
| L02 | Time, Clocks, and Causality | 4.5 |
| L03 | Replication, Quorums, and Durability | 4.5 |
| L04 | Consistency Models | 5 |
| L05 | Consensus: Paxos, Raft, and FLP | 5 |
| L06 | Partitioning, Rebalancing, and Membership | 4 |
| L07 | CRDTs and Eventual Consistency | 4 |
| L08 | Transactions, Sagas, and Idempotence | 4.5 |
| L09 | Failure Detection, Timeouts, and Resilience | 4 |
| L10 | Observability, Tracing, and Chaos Engineering | 4 |

## Assessment

| Component | Weight |
|---|---|
| Problem set 1 (L01–L04): a replicated store with a stated consistency model | 20% |
| Problem set 2 (L05–L07): consensus implemented, and a CRDT-based system | 20% |
| Problem set 3 (L08–L10): correctness under failure, tested adversarially | 20% |
| Written exam | 25% |
| Course position paper (1,500 words) | 15% |

## Term 5 build artifact (with DI-721)

**A replicated key-value store** with a consistency model you can state and defend:

- Multiple nodes, real network (containers), real partitions.
- A replication scheme with a stated durability model.
- Either a consensus protocol (Raft) or a quorum-based eventually consistent design with
  CRDTs — your choice, defended.
- Idempotent operations with at-least-once delivery.
- Failure detection with a documented accuracy/completeness trade-off.
- Distributed tracing.
- A **Jepsen-style test**: fault injection under load, with a linearizability or
  causal-consistency checker verifying the model you claim.

The last item is what makes this a graduate artifact. Claiming a consistency model is easy;
testing it is what distinguishes an engineer who has read about distributed systems from one
who has built one.

## Required reading

- Kleppmann, *Designing Data-Intensive Applications*, chs. 5, 8, 9. (Chs. 3, 7, 10–12 are
  DI-721's.)
- van Steen & Tanenbaum, *Distributed Systems*, 4th ed. (free from the authors).
- **Lamport, "Time, Clocks, and the Ordering of Events in a Distributed System" (1978).**
- **Fischer, Lynch & Paterson, "Impossibility of Distributed Consensus with One Faulty
  Process" (1985).**
- Gilbert & Lynch, "Brewer's Conjecture…" (2002).
- Herlihy & Wing, "Linearizability" (1990).
- Lamport, "Paxos Made Simple" (2001); Ongaro & Ousterhout, "In Search of an Understandable
  Consensus Algorithm" (2014).
- DeCandia et al., "Dynamo" (2007); Corbett et al., "Spanner" (2012).
- **Waldo et al., "A Note on Distributed Computing" (1994).**

## Recommended

- Cachin, Guerraoui & Rodrigues, *Introduction to Reliable and Secure Distributed
  Programming*.
- Kyle Kingsbury (aphyr), the Jepsen reports — read at least three.
- Shapiro et al., "Conflict-Free Replicated Data Types" (2011).
- Bailis & Ghodsi, "Eventual Consistency Today" (ACM Queue, 2013).
