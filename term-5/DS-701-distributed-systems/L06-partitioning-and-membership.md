# DS-701 · Lesson 06 — Partitioning, Rebalancing, and Membership

**Estimated study time:** 4 hours
**Prerequisites:** L03, L05; CS-621 L06

---

## 1. Orientation

Replication puts the same data in several places. **Partitioning (sharding) puts different data
in different places**, and it is what lets a system exceed the capacity of one machine.

The two are orthogonal and almost always combined: each partition is replicated, so a
100-partition cluster with 3× replication has 300 units of data spread over N machines.

The hard parts are not the partitioning itself. They are:

- **Choosing the key**, which determines whether load is even and whether queries are cheap —
  and which is expensive to change later.
- **Rebalancing** when nodes join or leave, without moving everything and without downtime.
- **Membership**: agreeing on who is in the cluster, in a system where "is that node down?" has
  no reliable answer (L01 §2.3).

## 2. Theory

### 2.1 Partitioning schemes

**Range partitioning.** Keys sorted; each partition owns a contiguous range.

- **Good**: range scans are one partition; ordered iteration works.
- **Bad**: hot spots. Timestamp-prefixed keys send all current writes to one partition — the
  single most common sharding mistake.
- Used by: HBase, Bigtable, CockroachDB, TiKV, and any B-tree-backed store.

**Hash partitioning.** `partition = hash(key) mod N`.

- **Good**: even distribution, no hot spots from key skew.
- **Bad**: range scans hit every partition; ordering is lost.
- **Worse**: changing N reshuffles nearly everything (§2.2).

**Consistent hashing.** Both keys and nodes are hashed onto a ring; a key belongs to the next
node clockwise.

- Adding or removing a node moves only `K/N` keys instead of nearly all.
- **Virtual nodes** (each physical node placed at many ring positions) are essential: without
  them the load variance is large, and with 100–200 virtual nodes per physical node the
  variance becomes acceptable. This is CS-621 L06 §2.5's material, and the load distribution is
  measurable.
- Used by: Dynamo, Cassandra, Riak, memcached clients, and most CDN request routing.

**Fixed partitions with assignment.** Create many more partitions than nodes (say 1,024 for 10
nodes) at the start, and assign partitions to nodes in a table. Rebalancing moves whole
partitions between nodes; the number of partitions never changes.

- **Good**: simple, explicit, controllable, and rebalancing is a data-movement decision rather
  than a hash function's emergent behaviour.
- **Bad**: the partition count is fixed at creation, and choosing it badly is expensive to fix.
- Used by: Kafka (topic partitions), Elasticsearch (primary shards), Riak.

**This last scheme is under-appreciated and is usually the right choice** for a system you
control: it gives you explicit control over placement, which matters for failure domains
(§2.4), and it avoids the operational surprises of implicit rehashing.

### 2.2 The key choice

The most consequential decision, and the hardest to reverse.

**Criteria:**

1. **Even distribution.** Measure the key's cardinality and frequency distribution *on real
   data*. A partition key with 20 distinct values cannot spread across 50 nodes.
2. **Query locality.** Queries that filter by the partition key hit one partition; everything
   else fans out to all of them. **A fan-out query's latency is the p99 of the slowest
   partition** (PY-601 L09 §2.6's tail amplification), so a system with 100 partitions and a
   p99 of 10 ms per partition has a fan-out p99 of well over 100 ms.
3. **Transaction locality.** Data that must be updated atomically should share a partition
   (SE-521 L07 §2.2 — this is the same argument as aggregate design, and it is not a
   coincidence: an aggregate is a partition-locality decision made in the domain).
4. **Bounded partition size.** A key whose values grow without limit (one user with 10⁹ rows)
   creates a partition that cannot be split.

**Hot spots** and their remedies:

| Cause | Remedy |
|---|---|
| Timestamp-prefixed keys | prefix with a hash, or a random bucket |
| A celebrity key (one user, one product) | split the key with a random suffix, and fan out reads |
| Low-cardinality key | compound key including something higher-cardinality |
| Sequential ids | hash the id, or use a random UUID |

The celebrity-key remedy deserves a note: appending a random suffix (`user123:0` through
`user123:15`) spreads the writes across 16 partitions at the cost of every read fanning out to
16. That trade is usually right for a write-hot key and wrong for a read-hot one.

**Secondary indexes** are the part that surprises people. Two options:

- **Local (document-partitioned)**: each partition indexes its own data. Writes are local and
  cheap; reads by the secondary key must **scatter-gather across every partition**.
- **Global (term-partitioned)**: the index is itself partitioned by the indexed term. Reads
  are targeted; writes must update a *remote* index partition, so the write becomes
  distributed and — if you want consistency — needs a transaction.

Elasticsearch and MongoDB use local; DynamoDB's global secondary indexes are global and
asynchronously updated (hence eventually consistent). **Neither is free, and knowing which
your system uses tells you which of your queries are expensive.**

### 2.3 Rebalancing

When you add or remove nodes, data must move. The requirements:

- **Move as little as possible.** `hash mod N` moves nearly everything; consistent hashing and
  fixed partitions move `K/N`.
- **Keep serving during the move.** Read from the old location until the new one is caught up,
  then cut over.
- **Bound the rate.** Rebalancing competes with production traffic for disk and network. An
  unthrottled rebalance is a self-inflicted outage, and it is a common one.
- **Be resumable.** A rebalance takes hours; it must survive a node restart.

**Automatic versus manual rebalancing** is a genuine operational trade. Automatic is convenient
and can be catastrophic: a node is declared dead (falsely — a GC pause, L01 §2.3), the system
starts moving its data, the extra load makes other nodes look dead, and the cascade takes the
cluster down. This has happened to many teams.

**The standard mitigation**: automatic detection, *manual or heavily rate-limited* execution.
Elasticsearch's `cluster.routing.allocation.*` settings and Cassandra's manual `nodetool` moves
both reflect this lesson learned.

### 2.4 Placement and failure domains

Where a replica goes matters as much as how many there are (L03 §2.5).

**Rack awareness**: never put all replicas of a partition in one rack, one AZ, or one power
domain. Every serious system supports this and many deployments do not configure it.

**The correlated-failure point again**: three replicas on three machines running the same
version, deployed by the same pipeline, in the same AZ, are not three independent failures.
Availability arithmetic assuming independence overstates reliability, sometimes by orders of
magnitude.

**Balancing multiple constraints** — even load, failure-domain diversity, and minimal data
movement — is a constrained optimization problem. It is NP-hard in general (CS-621 L08's
recognition table: this is a variant of bin packing with constraints), so real systems use
heuristics and accept a suboptimal placement. Knowing that stops you expecting the scheduler
to be optimal.

### 2.5 Membership

Who is in the cluster? The question is harder than it appears because "is that node down?" has
no reliable answer (L01 §2.3, L09).

**Approaches:**

**Consensus-based** (etcd, ZooKeeper, Consul). Membership is a value agreed by consensus
(L05). Strongly consistent, everyone agrees, and it does not scale past a few hundred nodes
because every membership change is a consensus round.

**Gossip / epidemic** (Cassandra, Consul's LAN gossip, Serf). Each node periodically exchanges
state with a random peer. Information spreads in `O(log n)` rounds. Scales to thousands of
nodes, is eventually consistent about membership, and nodes may temporarily disagree.

**SWIM** (Das, Gupta & Motivala, 2002) is the protocol worth knowing:

- **Failure detection**: each node periodically pings a random peer. On no reply, it asks
  `k` other nodes to ping the peer **indirectly** — which distinguishes "the target is dead"
  from "the network path between us is broken". That indirection is the key idea and it
  substantially reduces false positives.
- **Suspicion mechanism**: a node is marked *suspect* first, broadcast, and given time to
  refute before being declared dead. This further reduces false positives from transient
  slowness.
- **Dissemination**: membership updates piggyback on the ping traffic, so there is no separate
  gossip cost.

SWIM is used by Consul (via `memberlist`), HashiCorp's tooling generally, and many service
meshes. Implementing it is a good exercise and the indirect-probe idea is reusable.

**The layering that works in practice**: consensus for the *authoritative* small state (which
nodes exist, what the configuration is), gossip for *fast* propagation of liveness. Consul
does exactly this.

### 2.6 Coordination and service discovery

Related problems, usually solved with the same infrastructure:

- **Service discovery**: which instances are serving? DNS, Consul, etcd, or the platform's
  service abstraction.
- **Configuration**: distributed, with change notification. Watches on etcd or ZooKeeper.
- **Distributed locks**: with the fencing-token caveat (L03 §2.6). **A lock without a fencing
  token reaching the storage layer is not a lock.**
- **Leader election**: consensus (L05), or a lease from a consensus store.
- **Barriers and coordination**: ZooKeeper recipes.

**The uniform guidance**: use an existing coordination service. The recipes are subtle
(ZooKeeper's own documentation lists the herd effects and the ephemeral-node edge cases), and
the failure modes are the kind that lose data silently.

### 2.7 Designing a partitioning scheme

The procedure:

1. **Characterize the data.** Volume, growth rate, key cardinality, and the *frequency
   distribution* of keys — measured, on real data, not assumed.
2. **Characterize the queries.** Which filter by the candidate key (one partition) and which do
   not (fan-out)? What proportion of traffic is each?
3. **Choose the key.** Even distribution, query locality, transaction locality, bounded size.
4. **Choose the scheme.** Range if you need ordered scans; hash or consistent hashing if you
   do not; fixed partitions if you want explicit control.
5. **Choose the partition count.** More than the node count, by enough to allow growth —
   typically 10–100× the initial node count. Too few and you cannot rebalance finely; too many
   and per-partition overhead dominates.
6. **Decide the secondary index strategy**, knowing the cost of each.
7. **Plan the rebalancing**: rate limits, resumability, and whether it is automatic.
8. **Plan the placement**: failure domains, explicitly.
9. **Plan the resharding path.** How will you change the key or the count in three years? If
   the answer is "we cannot", say so now, because it is the decision most likely to be regretted
   (SE-521 L08 §2.5's reversibility).

## 3. Construction: partitioning the store

Extend the L05 store.

**Stage 1 — partition it.** Add partitioning to your replicated store: a partition map, routing
of requests to the owning partition, and one Raft group per partition (as CockroachDB and TiKV
do). Verify that partitions fail independently — killing all replicas of partition 3 must not
affect partition 7.

**Stage 2 — three schemes.** Implement hash, consistent hashing with virtual nodes, and fixed
partitions with an assignment table. For each, on a realistic key distribution, measure:

- **Load distribution**: the ratio of max to mean keys per node, and per node bytes.
- **Movement on rebalance**: the fraction of keys that move when adding one node, and when
  removing one.
- The effect of the virtual-node count (1, 10, 100, 500) on load variance.

Report the table. The consistent-hashing-without-virtual-nodes variance is usually startling.

**Stage 3 — hot spots.** Deliberately create one, three ways: a timestamp-prefixed key, a
celebrity key, and a low-cardinality key. Measure the load imbalance for each. Then implement
the remedy for each and measure again. For the celebrity key, measure the *read* cost of the
fan-out that the remedy introduces.

**Stage 4 — secondary indexes.** Implement both local and global secondary indexes. Measure:

- Write latency for each.
- Read latency for a secondary-key query, at 4, 16, and 64 partitions.
- Demonstrate the fan-out tail amplification: with per-partition p99 of X, measure the
  scatter-gather p99 and compare with the prediction.

**Stage 5 — rebalancing.** Implement it: move a partition from one node to another while
serving. Requirements: no failed requests during the move, resumable after a node restart, and
rate-limited. Then measure the effect of an *unthrottled* rebalance on production request
latency, and of a throttled one. Report the curve of rebalance duration against production p99.

**Stage 6 — SWIM.** Implement the membership protocol: direct probe, indirect probe through `k`
peers, the suspicion mechanism, and piggybacked dissemination. Then test with your fault
injection:

- A dead node is detected. Measure the time.
- A **slow** node (200 ms latency injected) is *not* falsely declared dead — this is what
  indirect probing buys, and demonstrating it is the point.
- A one-way partition: A cannot reach B, but C can reach both. Verify the indirect probe
  resolves it correctly.
- Measure the false-positive rate under 10% packet loss, with and without the suspicion
  mechanism.

**Stage 7 — failure-domain placement.** Add rack/AZ labels to nodes and a placement constraint
that no two replicas of a partition share a domain. Then verify: killing an entire "AZ" loses
no partition entirely. Measure the constraint's cost in load imbalance.

**Stage 8 — the design document.** Answer §2.7's nine questions for your store, including — and
especially — question 9.

## 4. Failure modes

- **Timestamp-prefixed partition keys.** All writes to one partition.
- **`hash mod N`.** Adding a node reshuffles everything.
- **Consistent hashing without virtual nodes.** Large load variance.
- **Too few partitions.** Cannot rebalance finely; a node cannot be added usefully.
- **Too many partitions.** Per-partition overhead (metadata, file handles, Raft groups)
  dominates.
- **Not measuring the key distribution on real data.**
- **Unthrottled rebalancing.** A self-inflicted outage.
- **Automatic rebalancing on false failure detection.** The cascade.
- **Replicas in one failure domain.** Availability arithmetic that assumes independence.
- **Local secondary indexes with heavy secondary-key traffic.** Every query fans out.
- **Distributed locks without fencing tokens.**
- **No resharding plan.** The decision you cannot reverse and did not know you were making.
- **Fan-out queries without accounting for tail amplification.**

## 5. Exercises

### Warm-up (30 min)

**W1.** For a table you work with, measure the actual frequency distribution of three candidate
partition keys. Report max/mean for each.

**W2.** Compute the fraction of keys that move when adding one node to a 10-node cluster, under
`hash mod N` and under consistent hashing.

**W3.** For a fan-out query over 50 partitions with per-partition p99 = 20 ms, estimate the
overall p99 and explain the mechanism.

### Core (3 h)

**C1 — Schemes compared.** Complete §3 stages 1–3. Deliverable: three schemes implemented, the
load-distribution and movement table, the virtual-node variance curve, and the three hot spots
created and remedied with measurements.

**C2 — Secondary indexes.** Complete §3 stage 4. Deliverable: both index types, the latency
measurements, and the demonstrated tail amplification compared with the prediction.

**C3 — Rebalancing.** Complete §3 stage 5. Deliverable: live rebalancing with no failed
requests, resumability tested by killing a node mid-move, and the throttle/duration/p99 curve.

**C4 — SWIM.** Complete §3 stage 6. Deliverable: the implementation, the four tests, and the
false-positive rates with and without the suspicion mechanism under packet loss.

### Challenge

**X1.** Read the Dynamo and Cassandra papers on partitioning, and Kafka's partition assignment
documentation. Implement a rebalancing *planner*: given current placement, node capacities,
failure domains, and a target, compute a plan minimizing data movement subject to the
constraints. Note that this is NP-hard (CS-621 L08); implement a heuristic, measure its
quality against an exact solver on small instances (CS-621 L09 §2.1), and report the gap.

**X2.** Design and implement online resharding: change the partition count of a live system
without downtime, with no lost writes and bounded staleness. This is genuinely hard —
Kafka cannot decrease partition count at all, and Elasticsearch requires a reindex. Report what
you could and could not achieve, and write 800 words on why this is the decision that most
deserves to be got right the first time.

## 6. Self-check

1. Give four partitioning schemes with their trade-offs, and say which is under-appreciated.
2. Give the four criteria for a partition key, and connect one to aggregate design.
3. Name four causes of hot spots and a remedy for each, including the cost of the celebrity-key
   remedy.
4. Distinguish local from global secondary indexes on read and write cost.
5. Give four requirements of a rebalancing implementation and the cascade failure mode.
6. Explain SWIM's indirect probe and suspicion mechanism and what each prevents.
7. Why is optimal replica placement NP-hard, and what follows?
8. Give the nine questions of a partitioning design, and say which is most often skipped.

## 7. Primary sources

- Kleppmann, *DDIA*, ch. 6.
- Karger et al., "Consistent Hashing and Random Trees" (STOC 1997).
- **DeCandia et al., "Dynamo" (SOSP 2007)** — partitioning and membership sections.
- Das, Gupta & Motivala, "SWIM: Scalable Weakly-consistent Infection-style Process Group
  Membership Protocol" (DSN 2002).
- Demers et al., "Epidemic Algorithms for Replicated Database Maintenance" (PODC 1987) — the
  origin of gossip.
- Chang et al., "Bigtable" (OSDI 2006) — range partitioning and tablet splitting.
- Kafka's documentation on partitions and the `KIP` archive on rebalancing protocols.

---

**Previous:** [L05](L05-consensus.md) · **Next:**
[L07 — CRDTs and Eventual Consistency](L07-crdts.md)
