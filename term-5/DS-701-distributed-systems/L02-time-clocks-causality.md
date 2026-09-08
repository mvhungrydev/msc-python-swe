# DS-701 · Lesson 02 — Time, Clocks, and Causality

**Estimated study time:** 4.5 hours
**Prerequisites:** L01

---

## 1. Orientation

Two servers, two writes to the same key, timestamps 14:02:17.003 and 14:02:17.001. Which
happened first?

**You cannot tell.** Server B's clock may be 5 ms ahead. Or 500 ms. Or the write with the
later timestamp may have been *caused* by the earlier one, or the two may be entirely
independent. A wall-clock timestamp from another machine is a number, not an ordering.

This lesson replaces wall clocks with something you can actually reason about: **causality**.
Lamport's 1978 paper is where distributed systems became a subject rather than a collection of
tricks, and its central move — define ordering by *communication* rather than by time — is the
foundation of everything that follows.

## 2. Theory

### 2.1 Why physical clocks fail

Three separate problems, and they compound:

**Skew.** Two clocks read different values at the same instant. NTP typically holds skew to
tens of milliseconds on a LAN, occasionally seconds, and there are documented cases of
minutes when NTP is misconfigured or a VM is migrated.

**Drift.** Clocks run at slightly different rates. A typical quartz oscillator drifts
~30 ppm — about 2.5 seconds per day uncorrected. Temperature changes it.

**Non-monotonicity.** NTP *steps* the clock backwards to correct it. A leap second may be
inserted or smeared. A VM resumes from a snapshot with a stale clock. So:

> **`time.time()` can go backwards.** Code that computes an elapsed time from two wall-clock
> readings can get a negative result.

Hence, in every language: use a **monotonic** clock for durations (`time.monotonic()`,
`time.perf_counter()` — PY-602 L01 §2.7) and a wall clock only for displaying times to humans
and for coarse-grained correlation.

Three real consequences worth knowing:

- The 2012 leap second caused widespread outages (Linux kernel and JVM bugs triggered by the
  repeated second).
- Cassandra's last-write-wins conflict resolution uses wall-clock timestamps, so a clock skew
  of a few seconds can cause a *newer* write to be silently discarded. This is a documented
  data-loss mode, not a hypothetical.
- Certificate validity, token expiry, and cache TTLs all depend on clocks that may disagree.

### 2.2 Happens-before

Lamport's definition. For events a and b, `a → b` ("a happens-before b") if:

1. a and b are in the same process and a comes first;
2. a is the sending of a message and b is its receipt;
3. transitively: `a → b` and `b → c` implies `a → c`.

If neither `a → b` nor `b → a`, the events are **concurrent** (`a ∥ b`).

This is a **partial order**, and the partiality is the point. In a distributed system there is
no total order of events — only the order that communication imposes. Two events on different
nodes with no message between them are genuinely unordered, and any total order you assign is
arbitrary.

Happens-before is exactly **potential causality**: `a → b` means a *could have* influenced b.
It does not mean it did. And `a ∥ b` means neither could have influenced the other, which is
the useful direction: **concurrent events can be reordered without changing anything**, which
is what licenses out-of-order processing and what CRDTs exploit (L07).

### 2.3 Lamport clocks

A counter per process, giving a total order consistent with happens-before:

```python
class LamportClock:
    def __init__(self) -> None:
        self.t = 0

    def local_event(self) -> int:
        self.t += 1
        return self.t

    def send(self) -> int:
        self.t += 1
        return self.t                       # attach to the message

    def receive(self, msg_t: int) -> int:
        self.t = max(self.t, msg_t) + 1
        return self.t
```

**Property**: `a → b` implies `L(a) < L(b)`.

**Not the converse.** `L(a) < L(b)` does *not* imply `a → b` — they may be concurrent.
Lamport clocks give you a total order that *respects* causality but cannot *detect* it.

Ties are broken by process id, giving a total order — which is enough for algorithms that
need *some* consistent order (Lamport's mutual exclusion algorithm, for example) but not for
detecting conflicts.

**Where they are enough**: assigning a consistent order to operations, generating unique
sortable ids, and any algorithm that needs an arbitrary but agreed order.

**Where they are not**: detecting whether two writes conflict. That needs vector clocks.

### 2.4 Vector clocks

One counter per process, so you can *detect* concurrency:

```python
class VectorClock:
    def __init__(self, node: str) -> None:
        self.node = node
        self.v: dict[str, int] = {}

    def tick(self) -> dict[str, int]:
        self.v[self.node] = self.v.get(self.node, 0) + 1
        return dict(self.v)

    def merge(self, other: dict[str, int]) -> None:
        for k, val in other.items():
            self.v[k] = max(self.v.get(k, 0), val)
        self.tick()

def compare(a: dict, b: dict) -> str:
    keys = a.keys() | b.keys()
    le = all(a.get(k, 0) <= b.get(k, 0) for k in keys)
    ge = all(a.get(k, 0) >= b.get(k, 0) for k in keys)
    if le and ge: return "equal"
    if le:        return "a happens-before b"
    if ge:        return "b happens-before a"
    return "concurrent"
```

**Property**: `V(a) < V(b)` **if and only if** `a → b`. Vector clocks characterize causality
exactly — that is what they buy over Lamport clocks and it is why Dynamo uses them.

**The cost**: O(n) space per timestamp, where n is the number of nodes that have ever written.
For a system with many clients this grows without bound, so real implementations prune
(with heuristics that can lose information) or use **dotted version vectors** (Riak), which
handle the client-server case correctly.

**Version vectors** are the same idea applied to replicas of an object rather than to
processes. Dynamo's context blobs are version vectors: on a conflicting read the client gets
multiple *siblings* and must merge them (L07).

### 2.5 Hybrid logical clocks

The practical compromise. Combine a physical timestamp with a logical counter:

```python
@dataclass(order=True, frozen=True)
class HLC:
    wall: int      # physical time, milliseconds
    logical: int   # counter for ties and for causality

def hlc_send(now_ms: int, last: HLC) -> HLC:
    if now_ms > last.wall:
        return HLC(now_ms, 0)
    return HLC(last.wall, last.logical + 1)

def hlc_receive(now_ms: int, last: HLC, msg: HLC) -> HLC:
    w = max(now_ms, last.wall, msg.wall)
    if w == last.wall == msg.wall: l = max(last.logical, msg.logical) + 1
    elif w == last.wall:           l = last.logical + 1
    elif w == msg.wall:            l = msg.logical + 1
    else:                          l = 0
    return HLC(w, l)
```

**Properties**: it respects happens-before (like a Lamport clock), it stays close to physical
time (bounded by the clock skew), and it is a fixed size. So you get causally-consistent
timestamps that are *also* meaningful to a human and usable for time-range queries.

Used by CockroachDB, MongoDB, and YugabyteDB. **This is the right default for a new system
that needs causal ordering with human-meaningful timestamps.**

### 2.6 TrueTime and the bounded-uncertainty approach

Spanner's answer: instead of pretending clocks agree, **quantify the uncertainty**.

TrueTime returns an *interval* `[earliest, latest]` guaranteed to contain the true time, with
the bound maintained by GPS receivers and atomic clocks in every datacenter. The uncertainty
ε is typically under 7 ms.

Then, to commit a transaction at timestamp T, Spanner **waits out the uncertainty**: it
delays until `TT.now().earliest > T`, guaranteeing that every subsequent transaction gets a
larger timestamp. That "commit wait" costs a few milliseconds of latency per transaction, and
in exchange Spanner provides **external consistency** (linearizability across the globe) with
ordinary timestamps.

The engineering lesson generalizes beyond Google's hardware: **making uncertainty explicit and
paying for it is better than assuming it away.** AWS's time-sync service and cloud
providers' improving clock infrastructure have made bounded-uncertainty approaches more
accessible; CockroachDB implements a similar idea with looser bounds and correspondingly more
retries.

### 2.7 Consistent snapshots

Related problem: capture the global state of a distributed system without stopping it.

**Chandy–Lamport snapshot algorithm** (1985): a process records its own state and sends a
marker on every outgoing channel; on receiving a marker for the first time, a process records
its state and sends markers onward; messages received between recording state and receiving a
marker on a channel are recorded as *in-flight* on that channel.

The result is a **consistent cut**: a set of states such that for every recorded receive, the
corresponding send is also recorded. It may not correspond to any instant that actually
occurred — but it is a state the system *could* have been in, which is what you need for
checkpointing and for detecting stable properties (deadlock, termination).

Used in: Flink's checkpointing (DI-721 L07), distributed debugging, and garbage collection.

The related concept for logs: a **consistent cut** is what makes "the state as of offset X"
meaningful across partitions.

### 2.8 Practical guidance

The rules:

1. **Never order events across machines by wall clock.** Use logical clocks, or accept that
   the ordering is arbitrary and say so.
2. **Use monotonic clocks for durations.** Always.
3. **Wall clocks for display and for coarse correlation only.**
4. **If you use timestamps for conflict resolution** (last-write-wins), know that you are
   choosing to lose data on clock skew, and bound the skew you tolerate. Cassandra's users
   frequently discover this the hard way.
5. **Prefer HLC** when you need causal order plus human-meaningful times.
6. **Vector clocks when you must detect concurrency** — and plan for the size growth.
7. **Measure your clock skew.** `chronyc tracking` / `ntpq -p`, exported as a metric with an
   alert. Very few teams do this, and it is a ten-minute job that turns a class of mystery
   into a visible signal.
8. **Never trust a client-supplied timestamp.** It is attacker-controlled and clock-skewed.

## 3. Construction: clocks in the laboratory

Using the fault-injection laboratory from L01.

**Stage 1 — measure real skew.** Across your five containers, measure the pairwise clock skew
over an hour. Then use your clock fault injection to skew one node by 5 seconds and observe
what breaks in a naive last-write-wins store. Quantify the data loss.

**Stage 2 — Lamport clocks.** Implement them across your nodes. Have each node perform local
events and send messages; log every event with its Lamport timestamp. Then verify empirically:
for 10,000 event pairs, check that `a → b` implies `L(a) < L(b)`, and **find pairs where
`L(a) < L(b)` but the events are concurrent** — demonstrating that the converse fails.

**Stage 3 — vector clocks.** Implement them. Verify the biconditional: `V(a) < V(b)` iff
`a → b`, over the same 10,000 pairs. Then measure the *size growth* of the vectors over a
long run with many clients, and implement one pruning strategy. Report what information the
pruning loses and construct a case where it causes a wrong answer.

**Stage 4 — conflict detection.** Build a replicated register with three replicas. Write
concurrently to different replicas under a partition, then heal. Compare three conflict
policies:

- **Last-write-wins by wall clock** — measure how much is lost under 100 ms and 5 s skew.
- **Last-write-wins by Lamport clock** — deterministic, but still loses one write.
- **Vector clocks with sibling return** — the client sees both and merges.

Report the data loss for each under identical fault schedules. **This is the experiment that
makes the trade concrete** and it is the one that turns "last-write-wins is dangerous" from a
slogan into a number.

**Stage 5 — hybrid logical clocks.** Implement HLC. Verify: it respects happens-before; it
stays within the clock skew of physical time; and it is a fixed size. Then run stage 4's
experiment with HLC and report where it sits between the other three.

**Stage 6 — Chandy–Lamport.** Implement the snapshot algorithm across your nodes with a
simple distributed application (money transfers between accounts on different nodes, with the
invariant that the total is conserved). Take snapshots during active transfers and verify the
invariant holds in every snapshot. Then take a *naive* snapshot (each node records its state
at its own local time) and demonstrate that the invariant is violated. The contrast is the
lesson.

**Stage 7 — the skew monitor.** Export clock skew as a metric from every node, with an alert
threshold. Then run the term's remaining experiments with it in place, and see whether it ever
fires.

## 4. Failure modes

- **Ordering events across machines by wall clock.** The foundational error.
- **`time.time()` for durations.** It can go backwards.
- **Last-write-wins without bounding clock skew.** Silent data loss.
- **Trusting a client-supplied timestamp.**
- **Vector clocks without a growth strategy.** They grow with the number of writers, forever.
- **Pruning vector clocks naively**, losing causality information and producing wrong
  conflict decisions.
- **Assuming NTP keeps clocks close.** It usually does, and "usually" is the problem.
- **A naive distributed snapshot.** Each node recording at its own local time gives an
  inconsistent cut, and the invariant violations are baffling.
- **Not monitoring skew.** A whole class of incident becomes unexplainable.
- **Using logical clocks for anything a human reads.** They are not times.

## 5. Exercises

### Warm-up (30 min)

**W1.** Construct a three-process execution where `L(a) < L(b)` but a and b are concurrent.
Draw the space-time diagram.

**W2.** For the same execution, show that the vector clocks correctly report concurrency.

**W3.** Demonstrate `time.time()` going backwards (step the clock in a container) and show
that `time.monotonic()` does not.

### Core (3 h)

**C1 — The clock experiments.** Complete §3 stages 1–3. Deliverable: the measured skew, the
Lamport converse counterexamples found empirically, the vector-clock biconditional verified,
and the size-growth measurement with a pruning strategy and its constructed failure case.

**C2 — Conflict resolution, measured.** Complete §3 stage 4. Deliverable: the three policies
implemented, identical fault schedules applied to each, and a table of data loss under 0, 100
ms, and 5 s of skew. Plus 400 words on which you would ship and why.

**C3 — HLC.** Complete §3 stage 5. Deliverable: the implementation, the three properties
verified, and its position in the stage 4 comparison. Then read CockroachDB's or MongoDB's
description of their HLC use and write 300 words on what they add beyond the basic algorithm.

**C4 — Snapshots.** Complete §3 stage 6. Deliverable: the Chandy–Lamport implementation, the
invariant verified across snapshots taken during activity, and the naive-snapshot violation
demonstrated. Explain in 300 words why a consistent cut need not correspond to any real
instant.

### Challenge

**X1.** Read Lamport (1978) in full. Implement his distributed mutual exclusion algorithm using
Lamport clocks, verify it provides mutual exclusion under fault injection, and measure its
message complexity. Then explain in 800 words why the algorithm needs a *total* order and why
Lamport clocks suffice even though they do not detect concurrency.

**X2.** Read the Spanner paper's TrueTime section. Implement a simplified bounded-uncertainty
clock in your laboratory: a `now()` returning an interval, with the bound derived from your
measured skew, plus commit-wait. Measure the latency cost of commit-wait as a function of the
uncertainty bound, and the correctness benefit (external consistency) by testing with a
linearizability checker. Report the trade curve.

## 6. Self-check

1. Give the three ways physical clocks fail and one real incident caused by each.
2. Define happens-before with all three clauses, and say what concurrency means and what it
   licenses.
3. State the Lamport clock property and its converse, and say why the converse fails.
4. State the vector clock biconditional and the cost that buys it.
5. What does an HLC give you that neither Lamport nor wall clocks do?
6. Explain TrueTime and commit-wait, and the general lesson about uncertainty.
7. What is a consistent cut, and why need it not correspond to a real instant?
8. Give the eight practical rules, and the one most teams skip.

## 7. Primary sources

- **Lamport, "Time, Clocks, and the Ordering of Events in a Distributed System" (CACM 1978).**
  The founding paper. Read it twice.
- Fidge (1988) and Mattern (1989) — vector clocks.
- Kulkarni et al., "Logical Physical Clocks and Consistent Snapshots in Globally Distributed
  Databases" (2014) — HLC.
- Corbett et al., "Spanner: Google's Globally-Distributed Database" (OSDI 2012) — TrueTime.
- Chandy & Lamport, "Distributed Snapshots: Determining Global States of Distributed Systems"
  (TOCS 1985).
- Kleppmann, *DDIA*, ch. 8 (the "Unreliable Clocks" section).
- Preguiça et al., "Dotted Version Vectors" (2010).

---

**Previous:** [L01](L01-failure-models-and-impossibility.md) · **Next:**
[L03 — Replication, Quorums, and Durability](L03-replication-and-quorums.md)
