# DS-701 · Lesson 03 — Replication, Quorums, and Durability

**Estimated study time:** 4.5 hours
**Prerequisites:** L01, L02

---

## 1. Orientation

Replication exists for three different reasons, and conflating them is the source of most bad
replication designs:

1. **Durability** — survive the loss of a machine or a datacenter.
2. **Availability** — keep serving when a machine is down.
3. **Read throughput / latency** — serve reads from more places, and from closer places.

These pull in different directions. Durability wants synchronous replication to distant
places; latency wants nothing of the sort. **A design that has not said which of the three it
is optimizing will get all three wrong.**

This lesson is about the mechanisms and, more importantly, about being able to state precisely
what your replication scheme actually guarantees.

## 2. Theory

### 2.1 Single-leader replication

One node accepts writes; followers replicate the log.

```
       writes
         │
         ▼
      ┌──────┐   replication log    ┌──────────┐
      │Leader│ ──────────────────►  │Follower 1│ ── reads
      └──────┘ ──────────────────►  │Follower 2│ ── reads
                                    └──────────┘
```

Used by: PostgreSQL, MySQL, MongoDB, Kafka (per partition), most relational databases.

**Synchronous versus asynchronous replication** is the central choice:

| | Synchronous | Asynchronous | Semi-sync |
|---|---|---|---|
| Write latency | leader + slowest follower | leader only | leader + fastest follower |
| Durability on leader loss | guaranteed | **data loss window** | guaranteed with ≥1 follower |
| Availability | a slow follower blocks writes | unaffected | tolerates n−1 slow followers |

**Asynchronous replication has a data-loss window**, and its size is your replication lag. If
the leader dies with 200 ms of unreplicated writes, those writes are gone — even though the
client was told they succeeded. That is a *correctness* property you are trading away, and it
should be a stated, measured number, not an accident.

Most production systems use semi-synchronous: at least one follower must acknowledge. That
bounds the loss to zero while tolerating slow followers.

**Replication lag** produces observable anomalies (§2.4).

**Failover** is where single-leader systems get into trouble:

- **Choosing a new leader** requires consensus, or you get split-brain (L05).
- **Data loss**: an async follower promoted to leader has lost the un-replicated tail.
- **Split-brain**: the old leader comes back and thinks it is still leader. Fencing tokens
  (§2.6) are the answer.
- **The timeout problem**: too short and you fail over on a GC pause; too long and you are down
  for that duration. There is no right answer (L09).
- **"Ghost" writes**: the old leader accepts a write after the new leader was elected. This is
  the split-brain data-corruption case and it is why fencing is not optional.

### 2.2 Multi-leader replication

Several nodes accept writes and replicate to each other. Used for: multi-datacenter operation,
offline-capable clients, and collaborative editing.

The benefit is obvious: writes are local, so they are fast and available during a partition.

**The cost is conflicts**, and they are unavoidable: two leaders accept conflicting writes to
the same key and there is no global order to appeal to. The resolution strategies:

- **Last-write-wins.** Loses data, and depends on clocks (L02 §2.1). Cassandra's default, and
  a documented data-loss mode.
- **Application-defined merge.** The client resolves, as in Dynamo's siblings.
- **CRDTs.** Structure the data so conflicts cannot arise (L07).
- **Avoid conflicts by partitioning writes** — route each key to a home region. This is the
  most common practical answer and it is under-used: if each user's data is written only in
  their home region, multi-leader becomes conflict-free by construction.

### 2.3 Leaderless replication and quorums

Dynamo's model: the client (or a coordinator) writes to several replicas directly.

With **N** replicas, **W** required write acknowledgements, and **R** required read responses:

> **If `W + R > N`, every read set intersects every write set**, so a read sees at least one
> replica with the latest write.

Common configurations, and the reasoning behind each:

| N | W | R | Property |
|---|---|---|---|
| 3 | 2 | 2 | strong-ish; tolerates one node down for both reads and writes |
| 3 | 3 | 1 | fast reads, writes need everyone (no write availability under any failure) |
| 3 | 1 | 3 | fast writes, slow reads |
| 3 | 1 | 1 | fast everything, **eventual only** |

**The quorum guarantee is weaker than it looks**, and this is the part people miss. `W+R>N`
does *not* give you linearizability. The documented gaps (Kleppmann, *DDIA* ch. 5):

- **Sloppy quorums with hinted handoff**: under a partition, writes go to *any* N reachable
  nodes, which may not be the home replicas. The intersection guarantee no longer holds.
- **Concurrent writes** need conflict resolution; the quorum does not order them.
- **A read concurrent with a write** may see either value, and two successive reads may see
  the new value then the old one.
- **A failed write** that reached some replicas is not rolled back; a later read may see it.
- **Node restarts from an empty state** can lose a replica's data, weakening the intersection.

So a quorum system provides "you will probably read a recent value", not a consistency model
with a name. If you want linearizability you need consensus (L05) or a linearizable read path
(read from a leader, or a quorum read with repair before returning).

**Anti-entropy** keeps replicas converging: **read repair** (fix stale replicas on read) and
**Merkle-tree comparison** (background full comparison, exchanging O(log n) hashes to find
differences). Dynamo and Cassandra use both.

### 2.4 Replication lag anomalies

Asynchronous replication produces specific, named, user-visible anomalies. Each has a standard
remedy:

**Read-your-writes.** A user posts a comment, the read goes to a lagging follower, the comment
is missing, the user posts it again. Remedies: read the user's own data from the leader; route
by session; or track the write's position and read from a replica that has caught up.

**Monotonic reads.** Successive reads go to different followers with different lag, so time
appears to move backwards. Remedy: pin a session to a replica, or track the last-seen version.

**Consistent prefix reads.** With partitioned data, a reader sees an answer before the
question. Remedy: causally related writes to the same partition, or explicit causal tracking
(L04 §2.4).

These are **session guarantees** (Terry et al., 1994), and they are the practical middle
ground: much weaker than linearizability, much cheaper, and sufficient for most user-facing
applications. Naming them is what lets you have the conversation with a product owner about
which anomalies are acceptable.

### 2.5 Durability, precisely

"The write is durable" is ambiguous. Say which:

| Level | Survives | Cost |
|---|---|---|
| In the leader's memory | nothing | ~0 |
| In the leader's page cache | process crash | ~0 |
| `fsync`'d on the leader | machine power loss | 0.1–10 ms (PY-602 L08 §2.4) |
| Replicated to a follower's memory | leader loss | network RTT |
| `fsync`'d on W replicas | W−1 simultaneous losses | RTT + fsync |
| Replicated across availability zones | AZ loss | cross-AZ RTT (~1 ms) |
| Replicated across regions | region loss | cross-region RTT (50–150 ms) |

**A durability claim must name the failure it survives.** "Three replicas" survives losing two
machines and *not* losing the datacenter. "Multi-AZ" survives a zone and not a region.
"Multi-region synchronous" survives a region and costs 100 ms per write.

The other axis nobody states: **correlated failure**. Three replicas on the same rack, on the
same power supply, running the same buggy version, deployed at the same time, are not three
independent failure domains. Availability arithmetic that assumes independence (SE-521 L09
§2.1) systematically overstates reliability, and correlated failure is the actual cause of
most multi-replica outages.

### 2.6 Fencing

The split-brain problem, and the only correct solution.

Node A is the leader. It pauses (GC, VM migration, `SIGSTOP` — L01 §3 stage 3). The system
declares it dead and elects B. A wakes up, unaware, and writes to storage.

**Leases and timeouts do not solve this**, because A's clock might be wrong or A might be
paused past its lease and not know. Any scheme relying on A voluntarily standing down is
unsound.

**Fencing tokens** do solve it: every leadership grant carries a monotonically increasing
token; every write to shared storage includes the token; **the storage system rejects any write
with a token lower than the highest it has seen.**

```
A is leader with token 33; pauses.
B becomes leader with token 34; writes with token 34.
A wakes, writes with token 33 → REJECTED by the storage layer.
```

The essential property: **the storage layer enforces it.** A cannot cheat, because it does not
have to cooperate. This is the difference between a protocol that assumes good behaviour and
one that does not.

ZooKeeper's `zxid`, etcd's revision, and Kafka's epoch are all fencing tokens. If you build a
leader-election scheme and there is no fencing token reaching the storage layer, **you have a
split-brain bug waiting for a GC pause**, and it will be found in production.

### 2.7 Choosing

The decision procedure:

1. **Why are you replicating?** Durability, availability, or read scaling. Usually one
   dominates.
2. **What is your write availability requirement during a partition?** This is the CAP choice
   (L01 §2.5), per operation.
3. **What data-loss window is acceptable?** Zero forces synchronous replication and its
   latency; a bounded window permits async and must be *measured*.
4. **Are conflicts possible, and can you avoid them?** Partitioning writes by key ownership
   often eliminates them entirely.
5. **Which session guarantees do users need?** Read-your-writes is almost always required;
   the others depend.
6. **What are your real failure domains?** Rack, AZ, region, and — the one nobody lists —
   deployment (a bad version reaching all replicas simultaneously).

Then write the answers down. **A replication design without stated answers to those six is not
a design.**

## 3. Construction: a replicated store

Extend the L01 laboratory.

**Stage 1 — single-leader, asynchronous.** One leader, two followers, a replication log.
Writes to the leader; reads from any node.

Then measure the anomalies:

- Under load, measure the replication lag distribution (p50, p99, max).
- Demonstrate **read-your-writes** violation: write, immediately read from a follower, observe
  the miss. Report the probability as a function of lag.
- Demonstrate **monotonic reads** violation with successive reads from different followers.
- Kill the leader with unreplicated writes in flight and **measure the data loss**: how many
  acknowledged writes were lost? Repeat 20 times and report the distribution.

That data-loss number is the deliverable. Most engineers have never measured it for their own
systems.

**Stage 2 — semi-synchronous.** Require one follower acknowledgement. Re-run the data-loss
experiment (it should be zero) and measure the write-latency cost. Then make one follower slow
(200 ms latency injection) and show that semi-sync still works while full sync would block.

**Stage 3 — session guarantees.** Implement read-your-writes three ways:

- Read from the leader for the session.
- Track the write's log position and read only from a replica at or beyond it.
- Sticky sessions to a replica.

Compare on: latency, leader load, and behaviour when the chosen replica fails.

**Stage 4 — leaderless quorums.** Implement Dynamo-style N/W/R with a coordinator. Then run the
experiments that show the quorum guarantee's limits:

- With N=3, W=2, R=2, construct a case where a read returns a stale value. (Concurrent
  write plus a failed write is the easiest.)
- Implement sloppy quorum with hinted handoff, and demonstrate that the intersection
  guarantee is broken during a partition.
- Implement read repair and measure convergence time after a partition heals.

**Stage 5 — anti-entropy.** Implement Merkle-tree comparison between replicas. Measure: the
bytes exchanged to detect a single differing key among 10⁶, versus a naive full comparison.
Report the ratio.

**Stage 6 — fencing.** Implement leader election (a simple lease is fine for now; Raft comes in
L05) *without* fencing tokens. Then use `SIGSTOP` on the leader to produce a split-brain and
**demonstrate the data corruption**. Then add fencing tokens enforced at the storage layer and
demonstrate that the same fault schedule is now safe.

**This is the most important experiment in the lesson.** Producing a split-brain corruption on
purpose, and then fixing it correctly, is what makes fencing memorable.

**Stage 7 — the durability table.** For your system, fill in §2.5's table with *measured*
latencies for each level on your infrastructure. Then state the failure your default
configuration survives, and the one it does not.

**Stage 8 — the design document.** Answer §2.7's six questions for your store, in writing.
This is the artifact.

## 4. Failure modes

- **Not stating why you replicate.** All three goals, none achieved.
- **Asynchronous replication with an unmeasured data-loss window.**
- **Believing `W+R>N` gives linearizability.** It does not.
- **Sloppy quorums presented as quorums.** The guarantee is gone.
- **Last-write-wins by wall clock.** L02 §2.1.
- **Leader election without fencing tokens.** Split-brain on the next GC pause.
- **Fencing enforced by the leader rather than by storage.** Unsound.
- **Failover timeouts tuned by guesswork**, causing failover on GC pauses.
- **Ignoring session guarantees**, then debugging "the data disappeared" reports from users
  who read their own write from a lagging replica.
- **Availability arithmetic assuming independent failures.** Replicas share racks, power,
  versions, and deploys.
- **No anti-entropy**, so replicas silently diverge forever after a partition.

## 5. Exercises

### Warm-up (30 min)

**W1.** For a database you use, determine: is replication synchronous, asynchronous, or
semi-sync by default? What is the data-loss window? Is it monitored?

**W2.** With N=5, list every (W, R) pair satisfying `W+R>N` and describe the availability
each gives for reads and for writes.

**W3.** Describe a fault schedule producing split-brain in a lease-based leader election, and
show how a fencing token prevents the corruption.

### Core (3 h)

**C1 — Single-leader, measured.** Complete §3 stages 1–3. Deliverable: the lag distribution,
the three anomalies demonstrated with probabilities, the measured data loss over 20 leader
kills, the semi-sync comparison, and the three read-your-writes implementations compared.

**C2 — Quorums and their limits.** Complete §3 stages 4–5. Deliverable: the N/W/R
implementation, the constructed stale read, the sloppy-quorum guarantee violation, read repair
with convergence times, and the Merkle-tree efficiency measurement.

**C3 — Split-brain and fencing.** Complete §3 stage 6. Deliverable: the corruption produced
deliberately with the exact fault schedule, the fencing implementation with storage-layer
enforcement, and the same schedule shown safe. Plus 400 words on why leader-side enforcement is
unsound.

**C4 — The design document.** Complete §3 stages 7–8. Deliverable: the measured durability
table for your infrastructure and the six answers. Then do the same for a production system you
work on, and report which questions nobody could answer.

### Challenge

**X1.** Read the Dynamo paper in full and implement its core: consistent hashing with virtual
nodes, N/W/R quorums, vector clocks with sibling return, hinted handoff, and Merkle-tree
anti-entropy. Then run a workload with continuous fault injection for an hour and report:
divergence, convergence time, sibling counts, and any data loss. Compare with what the paper
claims.

**X2.** Design and implement a replication scheme with *no* conflicts, by partitioning write
ownership by key. Then measure what it costs: cross-partition operations, rebalancing, and the
behaviour when a key's home partition is unavailable. Write 800 words on when conflict
avoidance beats conflict resolution — this is under-appreciated and it is often the right
answer.

## 6. Self-check

1. Give the three reasons to replicate and say why conflating them causes bad designs.
2. Compare synchronous, asynchronous, and semi-synchronous on three dimensions.
3. State the quorum condition and give four reasons it does not provide linearizability.
4. Name the three replication-lag anomalies and a remedy for each.
5. Give the durability ladder and say what "three replicas" does and does not survive.
6. Explain split-brain, why leases do not solve it, and how fencing tokens do.
7. Why must fencing be enforced at the storage layer?
8. Give the six questions a replication design must answer.

## 7. Primary sources

- Kleppmann, *DDIA*, ch. 5. The best treatment of replication anomalies and quorum limits.
- **DeCandia et al., "Dynamo: Amazon's Highly Available Key-value Store" (SOSP 2007).**
- Terry et al., "Session Guarantees for Weakly Consistent Replicated Data" (1994).
- Gray et al., "The Dangers of Replication and a Solution" (SIGMOD 1996) — why multi-master
  scales badly.
- Kleppmann, "How to do distributed locking" (2016) — the fencing-token argument, written as
  a critique of Redlock. Read it with the Redlock response for both sides.
- Merkle, "A Digital Signature Based on a Conventional Encryption Function" (1987) — the tree.

---

**Previous:** [L02](L02-time-clocks-causality.md) · **Next:**
[L04 — Consistency Models](L04-consistency-models.md)
