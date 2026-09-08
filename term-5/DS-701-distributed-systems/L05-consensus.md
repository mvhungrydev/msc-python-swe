# DS-701 · Lesson 05 — Consensus: Paxos, Raft, and FLP

**Estimated study time:** 5 hours
**Prerequisites:** L01–L04

---

## 1. Orientation

Consensus is the problem of getting a set of unreliable nodes to agree on a value. It sounds
narrow. It is not: **almost every hard problem in distributed systems reduces to it.**

- Leader election → agree on who leads.
- Distributed locks → agree on who holds the lock.
- Atomic commit → agree on commit or abort.
- Replicated state machines → agree on the log order.
- Membership → agree on who is in the cluster.
- Configuration → agree on the current configuration.

That last framing — **replicated state machine** — is the important one. If all replicas apply
the same deterministic operations in the same order, they end in the same state. So consensus
on a *log order* gives you a fault-tolerant version of *any* deterministic service. That is
what etcd, ZooKeeper, Consul, and every consensus-backed system actually provide.

## 2. Theory

### 2.1 The problem, stated

Each process proposes a value. The protocol must guarantee:

- **Agreement**: no two correct processes decide differently.
- **Validity** (non-triviality): the decided value was proposed by some process.
- **Termination**: every correct process eventually decides.
- **Integrity**: each process decides at most once.

FLP (L01 §2.4) says all four are unachievable deterministically in an asynchronous system with
one crash. Practical protocols therefore **guarantee agreement, validity, and integrity
always, and termination only when the network behaves.** Safety always; liveness when
possible.

That asymmetry is the design principle: **never trade safety for liveness.** A protocol that
stops making progress during a partition is doing its job; one that decides two different
values is broken.

### 2.2 Paxos

Lamport, 1998 (submitted 1990). Notoriously hard to understand from the original paper, and
the essential idea is simple once separated from the presentation.

**Roles**: proposers, acceptors, learners. In practice one process plays all three.

**Two phases**, both requiring a majority:

**Phase 1 (prepare).**
- A proposer picks a proposal number `n` (globally unique and increasing) and sends
  `prepare(n)` to a majority of acceptors.
- An acceptor receiving `prepare(n)`: if `n` is greater than any it has responded to, it
  **promises** not to accept any proposal numbered below n, and returns the highest-numbered
  proposal it has already accepted (if any).

**Phase 2 (accept).**
- If the proposer receives promises from a majority: it picks a value — **the value from the
  highest-numbered accepted proposal it heard about, or its own if none** — and sends
  `accept(n, v)`.
- An acceptor accepts unless it has promised to a higher n.
- If a majority accepts, the value is **chosen**.

**Why it is safe.** Two majorities intersect, so at least one acceptor sees both proposals.
The rule "adopt the highest-numbered accepted value" ensures that once a value is chosen, every
subsequent proposal proposes *that same value*. That single rule is the whole safety argument,
and it is worth writing out until it is obvious.

**Why it may not terminate.** Two proposers can duel: A prepares with n=1, B prepares with
n=2 invalidating A, A retries with n=3 invalidating B, forever. This is FLP made concrete.
The fix is a **distinguished proposer** (a leader) — and once you add that, you are most of
the way to Raft.

**Multi-Paxos** runs Paxos repeatedly for a log of values, with a stable leader skipping phase
1 for subsequent entries. This is what real systems implement, and it is precisely the part the
original paper does not specify — hence the folklore that "everyone implements a different
Paxos".

### 2.3 Raft

Ongaro & Ousterhout, 2014. Explicitly designed for **understandability**, and the paper is
unusual and worth reading for that alone: they treated comprehensibility as a design goal and
measured it with a user study.

Three decompositions make it tractable:

**1. Leader election.** Terms (logical time, monotonically increasing). Nodes are follower,
candidate, or leader. A follower that hears nothing for an election timeout becomes a candidate,
increments the term, and requests votes. A node grants at most one vote per term, and only to a
candidate whose log is at least as up to date as its own. A candidate with a majority becomes
leader.

**Randomized election timeouts** (150–300 ms, randomized per node) prevent the split-vote
duelling that plagues naive Paxos — a small, elegant solution to a real problem.

**2. Log replication.** Only the leader accepts writes. It appends to its log and sends
`AppendEntries` to followers. When a majority have stored an entry, it is **committed** and can
be applied to the state machine. Followers whose logs diverge are overwritten to match the
leader's.

**3. Safety.** Two properties do the work:

- **Election restriction**: a candidate cannot win unless its log contains all committed
  entries. Enforced by voters refusing candidates with less-up-to-date logs. This is what
  guarantees a new leader never discards a committed entry.
- **Log matching**: if two logs have an entry with the same index and term, all preceding
  entries are identical. Maintained by the consistency check in `AppendEntries`.

Together these give the **State Machine Safety** property: if a server has applied an entry at
an index, no other server will ever apply a *different* entry at that index.

**Membership changes** are the subtle part, and the place naive implementations break: adding
or removing nodes naively can produce two disjoint majorities and therefore two leaders. Raft
uses **joint consensus** (a transitional configuration requiring majorities of both old and
new) or single-server changes (add or remove one at a time, which is provably safe and is what
most implementations do).

**Log compaction** via snapshots, because the log cannot grow forever.

### 2.4 The consensus systems you actually use

| System | Protocol | Used for |
|---|---|---|
| ZooKeeper | Zab (similar to Multi-Paxos) | coordination, locks, config, membership |
| etcd | Raft | Kubernetes' entire state |
| Consul | Raft | service discovery, config |
| Spanner | Paxos per shard | globally distributed SQL |
| CockroachDB | Raft per range | distributed SQL |
| Kafka | KRaft (Raft), previously ZooKeeper | metadata, controller election |
| TiKV | Raft per region | distributed KV |

**The practical guidance is emphatic: do not implement consensus for production.** Use etcd
or ZooKeeper. The protocols are subtle, the implementations take years to harden, and the
failure modes are silent data loss. Implement one to *understand* it — which is exactly what
this lesson asks — and then use someone else's.

The corollary: **know what your consensus system guarantees and what it costs.** etcd is not a
database; it is a small, strongly consistent store for metadata, and using it for high-volume
data is a well-known way to take down a Kubernetes cluster.

### 2.5 The costs

Consensus is expensive, in specific and predictable ways:

- **Latency**: at least one round trip to a majority for every write. Cross-region, that is
  50–150 ms per write, and it cannot be avoided — it is the price of the guarantee.
- **Throughput**: bounded by the leader. A single Raft group does perhaps 10⁴–10⁵ writes/s;
  scaling means *sharding* into many groups (as CockroachDB and TiKV do, one Raft group per
  range).
- **Availability**: needs a majority. 3 nodes tolerate 1 failure; 5 tolerate 2. **An even
  number is strictly worse than the odd number below it** — 4 nodes tolerate 1 failure, same
  as 3, with more coordination. Always use an odd count.
- **Write amplification**: every write goes to every replica and is fsync'd.
- **Operational complexity**: membership changes, snapshots, and disk-full conditions are all
  places where consensus clusters get into trouble.

**Consequently: use consensus for the small, critical, low-volume decisions** — who is the
leader, what is the configuration, which shard owns which range — and keep bulk data out of it.
That layering is the standard architecture and it is why etcd holds Kubernetes' *state* and not
its container images.

### 2.6 Alternatives to consensus

Ask, before reaching for it:

- **Is a single node acceptable?** With a fast failover and a bounded data-loss window, often
  yes. A Postgres primary with a standby is not consensus and serves an enormous number of
  systems.
- **Can the operation be made commutative?** Then CRDTs (L07) and no coordination at all
  (CALM, L01 §2.6).
- **Can you partition ownership?** One writer per key means no agreement is needed
  (L03 §2.2).
- **Is a lease sufficient?** With **fencing tokens** (L03 §2.6), a lease from a single
  coordinator is much cheaper — though the coordinator itself usually needs consensus, so this
  is a layering rather than an elimination.
- **Can the decision be deferred?** Detect the conflict later and reconcile.

**Coordination avoidance is the most under-used technique in distributed systems.** Bailis
et al.'s "Coordination Avoidance in Database Systems" shows that many constraints commonly
believed to require coordination do not — and the analysis of *which* ones is exactly CALM's
monotonicity question.

### 2.7 Atomic commit, and why it is different

Consensus and **atomic commit** are related but distinct:

- **Consensus**: agree on *some* proposed value. Any proposal is a valid outcome.
- **Atomic commit**: commit only if *every* participant can. One abort forces abort.

**Two-phase commit (2PC)** is the standard protocol: a coordinator asks all participants to
prepare, and commits only if all vote yes.

Its fatal flaw: **it is blocking.** If the coordinator fails after participants have prepared,
they hold locks and cannot decide — they do not know whether the coordinator saw a unanimous
yes. They must wait for the coordinator to recover. In-doubt transactions holding locks are the
classic 2PC production incident.

**Three-phase commit** attempts to fix this and does not, under network partitions.

The practical answers:

1. **Make the coordinator fault-tolerant** by replicating it with consensus. This is what
   Spanner does (Paxos groups plus 2PC across them) and it works.
2. **Avoid distributed transactions** by keeping the transaction within one partition. The
   most common and best answer, and it is an argument for aggregate design (SE-521 L07 §2.2).
3. **Use sagas** with compensations (L08) and accept the lack of isolation.

The last is what most systems do, and knowing *why* — that 2PC blocks and that making it
non-blocking requires consensus — is what makes the choice principled rather than fashionable.

## 3. Construction: implementing Raft

This is the largest implementation exercise in the term, and it is worth it.

**Stage 1 — leader election.** Terms, the three states, `RequestVote` RPC, randomized
timeouts. No log yet. Test in your L01 laboratory:

- A leader is elected from a quiescent cluster.
- Kill the leader → a new one is elected within a bounded time. Measure it.
- Partition the cluster 2/3 → **only the majority side elects a leader**. Verify the minority
  side does not.
- Heal → the old leader steps down on seeing a higher term.
- **Symmetric and asymmetric partitions.** The asymmetric case is where naive implementations
  elect two leaders.

**Stage 2 — log replication.** `AppendEntries` with the consistency check, commit index,
applying to a state machine (your key-value store from L03).

- Writes go to the leader, replicate, commit on majority, apply.
- Kill a follower, write, restart it → it catches up.
- Partition a follower for a while → it catches up on heal.
- Kill the leader mid-replication → verify no committed entry is lost.

**Stage 3 — safety, tested.** The properties from §2.3:

- **Election restriction**: construct a scenario where a node with a stale log tries to become
  leader and verify it is rejected. This requires deliberately diverging logs, which is the
  interesting part.
- **Log matching**: verify after every operation that all logs agree on every entry up to the
  commit index.
- **State machine safety**: run a linearizability checker (L04 §3) against the whole cluster
  under continuous fault injection. **This is the real test** — it should pass, and if it does
  not you have a bug in your Raft.

**Stage 4 — the split-brain test.** Use `SIGSTOP` on the leader for longer than the election
timeout, elect a new leader, then `SIGCONT` the old one. Verify:

- The old leader does not commit anything after waking.
- No client observed a committed write disappearing.
- Verify with the linearizability checker.

**Stage 5 — membership changes.** Implement single-server changes. Then verify the unsafe case:
implement a *naive* multi-server change and demonstrate two disjoint majorities electing two
leaders. Then show your safe implementation prevents it.

**Stage 6 — snapshots.** Log compaction: snapshot the state machine, truncate the log, and
`InstallSnapshot` for a follower too far behind. Measure log growth with and without.

**Stage 7 — measure the costs.** From §2.5:

- Write latency versus cluster size (3, 5, 7 nodes) and versus inter-node latency (0, 10, 50,
  150 ms injected).
- Throughput ceiling for a single group.
- Availability: with 3 and with 5 nodes, how many simultaneous failures are tolerated?
- The cost of an even node count, demonstrated.

**Stage 8 — compare with etcd.** Run the same workload and the same fault schedule against a
real etcd cluster. Compare latency, throughput, and recovery times. Report the gap — it will be
large, and understanding *why* (batching, pipelining, optimized fsync, years of tuning) is the
lesson that concludes §2.4's advice.

**Stage 9 — the Jepsen-style test.** Continuous random fault injection (partitions, kills,
pauses, clock skew) for an hour, under load, with the linearizability checker running on the
recorded history. Report every violation. This is the term artifact's core.

## 4. Failure modes

- **Implementing consensus for production.** Use etcd.
- **An even number of nodes.** Strictly worse than the odd number below.
- **No fencing tokens** at the storage layer (L03 §2.6), so a stale leader corrupts data.
- **Naive membership changes** producing two disjoint majorities.
- **Election timeouts too short**, causing failover on GC pauses; too long, extending outages.
  Randomization is essential and is often omitted.
- **Not testing asymmetric partitions.** Where dual leaders come from.
- **Putting bulk data in the consensus store.** The classic way to destroy a Kubernetes
  cluster.
- **Unbounded log growth.** No snapshots, then a disk-full outage.
- **Assuming consensus gives you linearizability automatically.** Reads from a follower, or a
  leader that does not confirm it is still leader, are stale. Raft's read path needs a
  ReadIndex or a lease.
- **2PC with a single coordinator.** Blocking on coordinator failure, with locks held.
- **Reaching for consensus when partitioning ownership would do.**

## 5. Exercises

### Warm-up (30 min)

**W1.** Trace Paxos through a duelling-proposers scenario and show it fails to terminate.

**W2.** Construct a Raft scenario where a candidate is correctly denied a vote because its log
is stale. Draw the logs.

**W3.** Show why a 4-node cluster tolerates the same number of failures as a 3-node one.

### Core (4 h — the largest implementation in the term)

**C1 — Raft, elections and replication.** Complete §3 stages 1–2. Deliverable: the
implementation, and the six election tests and four replication tests passing under fault
injection, with measured election times.

**C2 — Safety, tested.** Complete §3 stages 3–5. Deliverable: the three safety properties
verified, the split-brain test with `SIGSTOP`, the linearizability checker passing under
continuous faults, and the naive-membership-change failure demonstrated then prevented.

**C3 — Costs and comparison.** Complete §3 stages 6–8. Deliverable: snapshots, the cost
measurements (latency versus cluster size and inter-node latency, throughput ceiling,
availability), and the etcd comparison with the gap analysed.

**C4 — The Jepsen test.** Complete §3 stage 9. Deliverable: the harness, an hour of continuous
faults under load, the checked history, and every violation found — or a statement of the
number of operations checked with none found.

### Challenge

**X1.** Read the Raft paper and the extended thesis, then implement the optimizations they
describe: batching, pipelining, the ReadIndex and lease-based read paths, and pre-vote (which
prevents a partitioned node from disrupting the cluster with term inflation on heal). Measure
each optimization's contribution to throughput and latency. Report which mattered most.

**X2.** Read "Paxos Made Simple" and "Paxos Made Live" (Chandra, Griesemer & Redstone, 2007 —
Google's account of what it actually takes to build a production Paxos). Implement basic
Paxos, then write 1,200 words on the gap between the algorithm and a production system, using
their catalogue of issues (disk corruption, membership, master leases, testing) and your own
experience from C1–C4.

## 6. Self-check

1. State the four consensus properties and which one practical protocols sacrifice.
2. Explain Paxos's two phases and the single rule that makes it safe.
3. Why can Paxos fail to terminate, and what is the standard fix?
4. Give Raft's three decompositions and the two safety properties.
5. What is the election restriction and what does it guarantee?
6. Give five costs of consensus and the architectural conclusion that follows.
7. Distinguish consensus from atomic commit, and explain why 2PC blocks.
8. Give five alternatives to consensus and the question that selects each.

## 7. Primary sources

- **Ongaro & Ousterhout, "In Search of an Understandable Consensus Algorithm" (USENIX ATC
  2014)** and Ongaro's PhD thesis (the extended version, with membership changes and
  optimizations).
- **Lamport, "Paxos Made Simple" (2001).** Then "The Part-Time Parliament" (1998) if you want
  the original.
- Chandra, Griesemer & Redstone, "Paxos Made Live — An Engineering Perspective" (PODC 2007).
  The most useful paper here for a practitioner.
- Howard & Mortier, "Paxos vs Raft: Have we reached consensus on distributed consensus?"
  (2020).
- Hunt et al., "ZooKeeper" (USENIX ATC 2010); Junqueira et al., "Zab" (2011).
- Bailis et al., "Coordination Avoidance in Database Systems" (VLDB 2014).
- The Raft visualization at raft.github.io and Ongaro's `raftscope` — genuinely useful for
  building intuition before implementing.

---

**Previous:** [L04](L04-consistency-models.md) · **Next:**
[L06 — Partitioning, Rebalancing, and Membership](L06-partitioning-and-membership.md)
