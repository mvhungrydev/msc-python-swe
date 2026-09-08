# DS-701 · Lesson 01 — Failure Models, Fallacies, and What Is Impossible

**Estimated study time:** 4 hours
**Prerequisites:** SE-521 L09

---

## 1. Orientation

Leslie Lamport, 1987:

> "A distributed system is one in which the failure of a computer you didn't even know
> existed can render your own computer unusable."

The definition that matters for this course is narrower and sharper:

> **A distributed system is one where components fail independently and cannot reliably
> determine each other's state.**

Both halves do work. *Independent failure* is why you cannot reason about the system as a
whole. *Cannot determine state* is the deeper problem: when you send a request and get no
reply, you cannot distinguish "the node is dead", "the node is slow", "the network dropped
your request", "the network dropped the reply", and "the node executed it and then died".

Those five cases require different responses, and **no algorithm can tell them apart.** That
is not a limitation of current technique; it is a theorem, and it is why this subject is hard.

## 2. Theory

### 2.1 The eight fallacies

Peter Deutsch and James Gosling's list, from Sun in the 1990s. Every one is still assumed
daily:

1. **The network is reliable.** It is not. Packets are dropped, reordered, duplicated.
2. **Latency is zero.** A same-datacenter round trip is ~0.5 ms; cross-continent ~150 ms.
   PY-602 L06's hierarchy puts that at 10⁶ times a cache access.
3. **Bandwidth is infinite.** And it is shared.
4. **The network is secure.** Assume an adversary on the wire.
5. **Topology doesn't change.** Nodes come and go; routes change; DNS changes.
6. **There is one administrator.** There are many, with different priorities, on different
   schedules.
7. **Transport cost is zero.** Serialization is often the dominant CPU cost (PY-602 L08 §2.7).
8. **The network is homogeneous.** Different versions, different configs, different hardware.

The list is worth more as a *checklist for a design review* than as history. Take any design
and ask, for each fallacy: where does this assume it? The answers are the failure modes.

### 2.2 Failure models

A design is only meaningful relative to a stated failure model. From weakest assumption to
strongest:

| Model | What can happen | Cost to tolerate |
|---|---|---|
| **Crash-stop** | A node stops and never returns | cheapest |
| **Crash-recovery** | A node stops and may return, with or without its state | needs stable storage |
| **Omission** | Messages are lost (send or receive) | retries, acknowledgements |
| **Timing** | Messages and processing take unbounded time | the hard one — §2.3 |
| **Byzantine** | Arbitrary behaviour, including malicious lies | 3f+1 nodes for f failures |

**Byzantine fault tolerance** (Lamport, Shostak & Pease, 1982) requires `3f+1` nodes to
tolerate `f` Byzantine faults, versus `2f+1` for crash faults. It is expensive and, in a
single trust domain, usually unnecessary — you are not defending against your own servers
lying. It matters for: blockchains, multi-party systems, and — increasingly relevant —
hardware corruption at scale, since Google and Meta have both published on
"silent data corruption" from CPUs producing wrong answers, which is Byzantine behaviour by
another name.

**State your model.** "We assume crash-recovery with fair-loss links and partial
synchrony" is a sentence that should appear in any serious design document, and its absence is
a sign that nobody has thought about it.

### 2.3 Synchrony assumptions

The most consequential axis, and the one people skip.

**Synchronous.** Known upper bounds on message delay and processing time. You can detect
failure perfectly: if no reply arrives within the bound, the node is dead. Consensus is easy.
**Real networks are not synchronous** — a GC pause, a VM migration, or a network hiccup
produces arbitrary delay.

**Asynchronous.** No bounds at all. You can never distinguish slow from dead. This is where
FLP bites (§2.4).

**Partially synchronous** (Dwork, Lynch & Stockmeyer, 1988). There *exist* bounds, but you do
not know them, or they hold only after some unknown time. This is the realistic model, and it
is what every practical consensus protocol assumes.

The practical consequence: **timeouts are a guess.** A timeout converts "I do not know" into
"I will assume dead", and the choice of value trades false positives (declaring a live node
dead, causing unnecessary failover) against slow detection. There is no correct value, only a
position on that trade (L09).

### 2.4 FLP: the foundational impossibility

**Theorem (Fischer, Lynch & Paterson, 1985).** *In an asynchronous system with even one
faulty process, there is no deterministic algorithm that solves consensus.*

Consensus means: all correct processes decide the same value (agreement), the value was
proposed by someone (validity), and every correct process eventually decides (termination).

FLP says you cannot have all three, deterministically, asynchronously, with one crash.

The proof idea: there exists a "bivalent" configuration — one from which both decisions are
still reachable — and an adversarial scheduler can always delay the critical message to keep
the system bivalent forever. It cannot force a *wrong* answer; it can force *no* answer.

**What FLP does not say**, and this is where people over-read it:

- It does not say consensus is impossible in practice. Paxos and Raft solve it every day.
- It says no *deterministic* algorithm guarantees *termination* in a fully *asynchronous*
  model.

The three escape routes, and every practical system uses one or more:

1. **Partial synchrony.** Assume bounds eventually hold. Paxos and Raft terminate when the
   network behaves, and merely fail to make progress when it does not — never deciding
   *wrongly*. This is the standard choice: **sacrifice liveness, never safety.**
2. **Randomization.** Ben-Or's algorithm terminates with probability 1. Used in some
   Byzantine protocols.
3. **Failure detectors.** Chandra & Toueg showed that a failure detector with specific
   properties (◇W — eventually weak) is the *weakest* additional assumption sufficient for
   consensus. A beautiful result: it precisely characterizes what you must add.

**The lesson to carry**: when a protocol claims to solve consensus, ask which escape it uses.
"It's Paxos" answers it: partial synchrony, safety always, liveness when the network behaves.

### 2.5 CAP, correctly

**Theorem (Gilbert & Lynch, 2002, formalizing Brewer's 2000 conjecture).** *A distributed data
store cannot simultaneously provide Consistency (linearizability), Availability (every request
to a non-failing node receives a response), and Partition tolerance.*

The near-universal misreading is "pick two". You cannot pick two, because **partitions are not
optional** — networks partition, and you do not get to decline. So the real statement is:

> **When a partition occurs, you must choose between consistency and availability.**

- **CP**: refuse to serve requests that cannot be made consistent. The minority side of a
  partition stops. ZooKeeper, etcd, Spanner, HBase.
- **AP**: keep serving, accept divergence, reconcile later. Dynamo, Cassandra (tunable),
  Riak.

The other misreadings worth naming:

- **"Consistency" here means linearizability**, not ACID's C (which is about integrity
  constraints) and not "the data is correct". Three different meanings of one word.
- **"Availability" means every non-failing node responds** — a very strong requirement.
  A system with 99.99% availability is not "A" in the CAP sense.
- **It is about one operation, not a system.** Different operations in the same system can
  make different choices.
- **It says nothing about the non-partitioned case**, which is 99.9% of the time.

**PACELC** (Abadi, 2012) fixes that last gap:

> **If Partitioned, choose Availability or Consistency; Else, choose Latency or Consistency.**

The `else` half is the one that matters daily: even with no partition, synchronous replication
for consistency costs latency. Spanner is CP/EC — consistent always, paying latency. Dynamo is
AP/EL. Most systems are somewhere in between and have never articulated where.

**The useful framing** for a design review: not "are we CP or AP" but *"for each operation,
what do we do during a partition, and what latency do we pay when there isn't one?"*

### 2.6 The other impossibility results

Worth knowing so you recognize them:

- **The Two Generals Problem.** Two parties cannot reach certain agreement over an unreliable
  channel: any protocol's last message might be lost, so the sender cannot know it arrived.
  Consequence: **exactly-once delivery over a network is impossible.** What people call
  "exactly-once" is at-least-once delivery plus idempotence (L08), and calling it anything
  else is marketing.
- **CALM theorem** (Hellerstein & Alvaro). A program has a coordination-free (eventually
  consistent) implementation *if and only if* it is monotonic — it never retracts a
  conclusion. This is the theoretical basis of CRDTs (L07) and it tells you *which* problems
  can avoid coordination.
- **The end-to-end argument** (Saltzer, Reed & Clark, 1984). A function can only be
  correctly implemented with knowledge held at the endpoints, so implementing it at a lower
  layer is at best an optimization. Why TCP's reliability does not give you application-level
  reliability, and why you still need application acknowledgements.

### 2.7 What this means in practice

The design habits that follow:

**Assume the network will partition.** Not "might" — will. Design the behaviour for it, and
test it (L10).

**Every remote call has three outcomes, not two**: success, failure, and *unknown*. The third
is the one code forgets. A timeout is an unknown, not a failure — the operation may have
succeeded.

**Idempotence is the universal answer.** Because the unknown case forces a retry, and a retry
must be safe. This is the conclusion PY-601 L07 §2.6, SE-521 L07 §2.5, and PY-502 L07 all
reached from different directions, and it is the single most useful design principle in this
course.

**Prefer coordination-free designs where the problem allows** (CALM). Coordination costs
latency and availability; the question is whether you need it, and often you do not.

**State your model.** Failure model, synchrony assumption, consistency model, durability
model. Four sentences in a design document, and their absence is a red flag.

## 3. Construction: building the failure laboratory

You cannot study this subject without the ability to break things on purpose. Build the
laboratory now; you will use it for the rest of the term.

**Stage 1 — a multi-node environment.** Five nodes in containers, on a network you control
(`docker compose` with a user-defined bridge). A trivial service on each: a key-value store
with `get` and `put`, no replication yet.

**Stage 2 — network fault injection.** Using `tc netem` inside the containers (or a proxy such
as Toxiproxy):

- **Latency**: add 100 ms, 1 s, and a distribution with a long tail.
- **Loss**: drop 1%, 10%, 50% of packets.
- **Partition**: cut node A from node B, both ways and one way. **Asymmetric partitions —
  where A can reach B but B cannot reach A — are real and are the ones that break naive
  designs**; make sure your tool can produce them.
- **Reorder and duplicate.**
- **Slow**: bandwidth limiting.

Wrap it in a Python API: `with partition(nodes=["a"], from_=["b","c"]): ...`

**Stage 3 — process fault injection.** Kill (`SIGKILL`, no cleanup), pause (`SIGSTOP` — which
simulates a long GC pause and is *not* the same as a crash, since the process comes back with
its state), and restart with and without preserved state.

The `SIGSTOP` case is the important one and is usually missing from test suites: a paused node
looks dead, then wakes up and acts as if no time passed. Real systems break on this.

**Stage 4 — clock fault injection.** Skew a node's clock forwards and backwards, and make it
jump. You will need this in L02, and the fact that you can build it now will change what you
believe about wall clocks.

**Stage 5 — the observability.** Every node logs with a node id and a monotonic timestamp;
logs are aggregated; a request carries a trace id end to end. Without this, every experiment in
this course produces a mystery instead of a finding.

**Stage 6 — the first experiment.** With no replication, measure: what happens to `get` and
`put` under each fault. Record the *client-observable* behaviour: success, error, timeout,
wrong answer. Build the table. This is your baseline and it will be a useful contrast when you
add replication.

**Stage 7 — the five-outcomes demonstration.** Write a client that issues a `put` and
experiences: success, connection refused, timeout with the write applied, timeout with the
write not applied, and a successful response that arrives after the client gave up. Construct
each deliberately with your fault injection. **This is the exercise that makes §2.7's "three
outcomes" real**, and most engineers have never seen the third and fifth cases produced on
purpose.

## 4. Failure modes

- **No stated failure model.** The design is unfalsifiable.
- **Assuming synchrony.** "It responds within 100 ms" is not a guarantee.
- **Treating a timeout as a failure.** It is an unknown; the operation may have succeeded.
- **"Pick two" CAP.** Partitions are not optional.
- **Confusing the three meanings of "consistency"** — linearizability, ACID's C, and "the data
  looks right".
- **Believing in exactly-once delivery.** Two Generals.
- **Testing only crash failures.** Pauses, asymmetric partitions, and clock skew break
  different things.
- **No asymmetric partition testing.** The most common gap, and the source of split-brain
  bugs.
- **Ignoring the else half of PACELC.** The latency cost of consistency is paid every day, not
  only during partitions.
- **Assuming a single administrator, a stable topology, or a homogeneous fleet.** Fallacies 5,
  6, 8, and each has a characteristic outage.

## 5. Exercises

### Warm-up (30 min)

**W1.** For a system you work on, go through the eight fallacies and identify, for each, one
place the design assumes it.

**W2.** For a remote call in your code, enumerate the five possible states after a timeout and
say what your code does in each.

**W3.** State the failure model, synchrony assumption, consistency model, and durability model
of a system you use. Where you cannot, note it — that is the finding.

### Core (3 h)

**C1 — The laboratory.** Complete §3 stages 1–5. Deliverable: the container environment, the
fault-injection API (including asymmetric partitions and `SIGSTOP`), clock skew, and the
observability. This is infrastructure for the whole term, so build it properly.

**C2 — The baseline experiments.** Complete §3 stages 6–7. Deliverable: the fault/behaviour
table, and the deliberate construction of all five post-timeout outcomes with evidence for
each.

**C3 — CAP in practice.** For three systems you use (a database, a cache, a message broker),
determine empirically or from documentation: what happens to reads and writes on the minority
side of a partition? Report the answers and whether the documentation says. Then classify each
on PACELC, both halves.

**C4 — Read FLP.** Read Fischer, Lynch & Paterson (1985). Write 600 words explaining the
bivalence argument in your own words, stating precisely what the theorem forbids, and naming
which escape route each of Raft, Paxos, and a Byzantine protocol of your choice takes.

### Challenge

**X1.** Read Bailis & Kingsbury, "The Network is Reliable" (ACM Queue, 2014) — a collection of
real partition incidents. Then write 1,200 words on the gap between the failure models
engineers assume and the failures that actually occur, using at least four of the incidents.
Conclude with a checklist for a design review.

**X2.** Extend your laboratory into a general fault-injection framework: a scenario DSL
(`partition(a, b) for 30s; then kill(c); then heal()`), deterministic replay from a seed, and
an assertion API for invariants that must hold throughout. This becomes the harness for every
subsequent problem set and for the term artifact's Jepsen-style test.

## 6. Self-check

1. Give the two-part definition of a distributed system and say why each half matters.
2. List the eight fallacies and give one design assumption each.
3. Give the five failure models and the cost of tolerating each.
4. Distinguish synchronous, asynchronous, and partially synchronous, and say which is
   realistic.
5. State FLP precisely, and give the three escape routes with a system using each.
6. State CAP correctly and give three common misreadings.
7. State PACELC and explain why the "else" half matters more day to day.
8. Explain why exactly-once delivery is impossible and what people mean when they claim it.

## 7. Primary sources

- **Fischer, Lynch & Paterson, "Impossibility of Distributed Consensus with One Faulty
  Process" (JACM 1985).**
- **Gilbert & Lynch, "Brewer's Conjecture and the Feasibility of Consistent, Available,
  Partition-Tolerant Web Services" (SIGACT News 2002).**
- Abadi, "Consistency Tradeoffs in Modern Distributed Database System Design" (IEEE Computer,
  2012) — PACELC.
- Dwork, Lynch & Stockmeyer, "Consensus in the Presence of Partial Synchrony" (JACM 1988).
- Chandra & Toueg, "Unreliable Failure Detectors for Reliable Distributed Systems" (1996).
- Lamport, Shostak & Pease, "The Byzantine Generals Problem" (1982).
- Saltzer, Reed & Clark, "End-to-End Arguments in System Design" (1984).
- Bailis & Kingsbury, "The Network is Reliable" (ACM Queue, 2014).
- **Waldo et al., "A Note on Distributed Computing" (1994).**

---

**Next:** [L02 — Time, Clocks, and Causality](L02-time-clocks-causality.md)
