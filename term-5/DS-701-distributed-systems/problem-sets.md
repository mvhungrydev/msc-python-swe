# DS-701 — Problem Sets

The three problem sets build one artifact: a replicated key-value store that you can partition,
kill, slow down and observe. Each set adds a layer to the same system, so do them in order and
keep the code. By the end of Problem Set 3 you should be able to hand someone the repository, have
them run one command, and watch your cluster survive a partition on their machine.

**A note on evidence.** In this course a claim without a measurement is not an answer. "It handles
partitions" is worth nothing; "here is the partition injected at t=12s, here is the trace, here is
the linearizability checker's verdict, here are the 40 seconds of unavailability on the minority
side" is the standard. Every part below that says *demonstrate* means with a reproducible script
and recorded output, not with prose.

**A note on honesty.** You will find bugs in your own implementations that you do not fully
understand. Write them down as open questions rather than papering over them. A design note that
says "under this fault my system loses a write and I have not yet found the reason" is worth more
than one that quietly avoids running that fault.

---

## Problem Set 1 — A Replicated Store With a Stated Consistency Model
**Covers L01–L04 · Budget: 20–24 hours**

**Part A — The fault-injection laboratory (L01).** The harness through all its stages: a
multi-node in-process cluster with a message layer that can drop, delay, reorder, duplicate and
partition, driven by a seed so runs are reproducible. Deliver the seed-replay demonstration: the
same seed produces the identical message schedule twice.

**Part B — The failure model, written down (L01).** For your system, state the model explicitly:
crash-stop or crash-recovery, synchrony assumptions, what you assume about message loss and
duplication, and what you explicitly do *not* handle (Byzantine faults, disk corruption, clock
jumps backwards). One page. This document is referenced by every later part, and Problem Set 3
will test whether you honoured it.

**Part C — Time and causality (L02).** Lamport clocks and vector clocks over your message layer.
Demonstrate, with recorded traces: two events that Lamport clocks order but that are actually
concurrent; the same pair correctly identified as concurrent by vector clocks; and the storage
growth of vector clocks as nodes are added. Then implement a hybrid logical clock and show it
stays within a bounded offset of physical time while preserving causality.

**Part D — The clock-skew hazard (L02).** Build the last-write-wins register, then inject 500 ms
of clock skew on one node and produce a *lost update*: a write that is acknowledged and then
silently discarded. Record it. Then implement the same register with a version vector and show the
conflict is surfaced rather than lost. Your write-up must state precisely what LWW trades away.

**Part E — Replication and quorums (L03).** Leaderless replication with configurable N, R, W.
Demonstrate: R+W>N giving read-your-writes under normal operation; the case where R+W>N still
returns stale data (concurrent writes in flight — be precise about why the quorum intersection
argument does not save you here); read repair; and hinted handoff with its recovery. Measure
availability and latency for at least four (N,R,W) configurations under a fixed fault schedule.

**Part F — Durability (L03).** Implement three durability levels: acknowledge on receipt,
acknowledge on fsync, acknowledge on quorum-fsync. Measure the latency of each. Then kill nodes
with `SIGKILL` at controlled moments and show, by counting, exactly which acknowledged writes each
level can lose. A durability claim you have not falsified is a durability guess.

**Part G — Consistency models (L04).** Implement a linearizability checker (or use Porcupine and
justify the choice) and wire it into your harness. Then: demonstrate a history your system
produces that is *not* linearizable; add whatever it takes to make the single-key case
linearizable; and demonstrate a history that is linearizable per key but not across keys, which is
the distinction most people get wrong.

**Part H — Sessions (L04).** Implement read-your-writes, monotonic reads and monotonic writes as
session guarantees on top of your eventually consistent store. For each, construct the violation
that occurs without it and show it disappearing with it.

**Design note (1,500–2,000 words).** State your system's consistency model in the vocabulary of
L04, precisely, including what it guarantees per key and across keys, and what a client can and
cannot assume. Then: what the clock-skew lost update taught you that reading about LWW did not;
where your quorum intuition was wrong; and the durability level you would actually ship, with the
argument.

---

## Problem Set 2 — Consensus, and a System That Does Not Need It
**Covers L05–L07 · Budget: 22–26 hours**

**Part A — Raft (L05).** A working implementation: leader election with randomised timeouts, log
replication, commit index advancement, and persistence of `currentTerm`, `votedFor` and the log
across restarts. It must pass, under your L01 harness with adversarial schedules, the invariants:
election safety, leader append-only, log matching, leader completeness, state machine safety.
Each invariant checked automatically, not by inspection.

**Part B — Raft under adversity (L05).** Demonstrate, each with a recorded run: an election under
partition; a leader partitioned into the minority and correctly stepping down; log divergence on
a minority node and its repair; a node restarting and catching up; and the split-vote case
resolving through randomised timeouts. Then measure election latency across a range of timeout
settings and explain the shape of the curve.

**Part C — The stale leader (L05).** Construct the case where a partitioned old leader believes it
is still leader and serves a stale read. Then fix it twice — once with a read-index / heartbeat
quorum check, once with leader leases — and state the assumption each fix relies on. Explain why
the lease version's safety depends on a clock bound and what happens if that bound is violated.

**Part D — FLP, concretely (L05).** Write a 600-word explanation of FLP that would satisfy an
examiner, then explain — with reference to *your own code* — exactly which line makes your
implementation not a counterexample to it.

**Part E — Partitioning and membership (L06).** Consistent hashing with virtual nodes: measure
load distribution across 5, 50 and 500 vnodes per physical node, and measure how many keys move
when a node is added and when one is removed. Compare against modulo hashing on the same workload
and produce the chart. Then implement range partitioning and state the workload for which you
would choose each.

**Part F — Failure detection and rebalancing (L06).** SWIM-style membership: direct probe,
indirect probe, suspicion with an incarnation number, and dissemination. Demonstrate a false
positive under a slow-node injection and show the suspicion mechanism preventing a spurious
eviction. Then implement rebalancing that moves partitions without dropping writes, and prove it
by running a write workload throughout the rebalance and checking for lost acknowledged writes.

**Part G — CRDTs (L07).** Implement G-Counter, PN-Counter, OR-Set and an LWW register, each with
property-based tests asserting commutativity, associativity and idempotence of merge over randomly
generated update sequences and delivery orders (this is where SE-511 L04's Hypothesis work pays
off). Then build a small collaborative document — a text or list CRDT — and demonstrate three
replicas converging after arbitrary offline editing.

**Part H — The impossibility (L07).** Take the invariant "the account balance must never go
negative" and demonstrate that no CRDT can enforce it under partition. Then implement the escrow
pattern and state exactly what it guarantees and what it costs. This part is the one that
establishes whether you understood the CALM boundary or merely read it.

**Part I — The comparison.** Build the *same* small application — a shopping cart is the standard
example — twice: once on your Raft-replicated store, once on CRDTs. Under an identical fault
schedule, measure availability, write latency and the anomalies each exhibits. Produce the table.

**Design note (1,500–2,000 words).** Which design you would ship for the cart, and why. Where your
Raft implementation was hardest to get right, and what that says about implementing consensus
yourself versus using an existing implementation. The specific thing the escrow exercise taught
you about coordination.

---

## Problem Set 3 — Correctness Under Failure, Tested Adversarially
**Covers L08–L10 · Budget: 22–26 hours**

**Part A — The dual write (L08).** Stages 1–4 of L08 §3: the naive version with its crash test,
the outbox, the relay with its observed duplicate, and the inbox absorbing it. The evidence must
be row counts under injected crashes at every step boundary, not logs.

**Part B — Idempotency keys (L08).** Stage 5, with all three tests: sequential retry, concurrent
retry from two threads, and key reuse with a different body. Then state your retention policy and
demonstrate what happens to a retry that arrives after the window — and argue that your chosen
window and your client retry deadline are consistent with each other.

**Part C — The saga (L08).** Stages 6–8: the durable saga engine surviving a crash at every step
boundary, compensation in reverse order with idempotent compensations, the pivot, and both
isolation anomaly fixes with the throughput comparison. Deliver the state diagram and the
step/compensation/partially-done table.

**Part D — Boundary analysis (L08).** For your key-value store and the cart application, write the
analysis: which invariants hold atomically, which aggregate owns each, which cross-boundary
invariants you enforce eventually, and the window and user-visible effect of each eventual one.

**Part E — Resilience, and the harm it can do (L09).** L09 Stages 1–6: the controllable
dependency, the reproduced metastable failure, deadline propagation with verified downstream
cancellation, the three-curve retry chart (no jitter / full jitter / retry budget), the circuit
breaker with both its success case and its misconfiguration case, and bulkheads with adaptive
concurrency limits. The three-curve chart and the breaker misconfiguration are the two required
deliverables.

**Part F — Failure detection, revisited (L09).** The φ-accrual detector over your L06 heartbeats,
with the plot of φ under slow network, partition and genuine crash, and the marked firing points
for a fixed timeout and for φ = 3 and φ = 8. Then the paragraph on which threshold you would use
for routing and which for destructive failover.

**Part G — Observability (L10).** Stages 1–6: wide structured events with a query script, real
OpenTelemetry tracing of a Raft election with W3C context propagation, the clock-skew trace
artefact, head-based versus tail-based versus weighted sampling with the retention comparison,
multi-window multi-burn-rate SLO alerting replayed against three incident shapes, and exemplars
linking a latency bucket to a trace.

**Part H — Chaos (L10).** The experiment runner, and at least six experiments with hypotheses
written *before* each run: leader kill, minority partition, leader-into-minority partition,
500 ms link latency, 10% packet loss, 5-second clock skew. Report each as hypothesis / result /
action. Then the required write-up of the one that surprised you.

**Part I — The adversarial pass.** Hand your system, and your Part B failure model from Problem
Set 1, to someone else (or to yourself after a two-week gap) with one instruction: *find a
schedule under which an acknowledged write is lost or a linearizability violation occurs, staying
within the stated failure model.* Spend at least four hours on this. Report what you found, or
report the search you conducted and why you believe the invariant holds. A negative result honestly
described is acceptable; not looking is not.

**Design note (2,000–2,500 words).** The failure model you stated in Problem Set 1, revisited:
which of its assumptions turned out to be load-bearing, and which you violated without noticing
until Part I. The metastable failure you reproduced, analysed as trigger and sustaining effect.
What tracing showed you that logs had not. And the answer to the question the whole course is
built around: *for the system you built, what does a client actually get, under what conditions,
and how do you know?*

---

## Course position paper (1,500 words)

Choose one:

1. **"Most systems that need distributed transactions have a partitioning mistake upstream."**
   Defend or refute, using your own boundary analysis and at least two of the primary sources.
2. **"CRDTs are a beautiful answer to a narrow question, and the industry over-applies them."**
   Same standard.
3. **"The dominant cause of large outages is the response to failure, not the failure."** Argue
   this from your own Stage 2 metastable reproduction and at least two published postmortems.
4. **"Strong consistency is now cheap enough that eventual consistency is rarely the right
   default."** Address Spanner, PACELC, and the actual latency numbers, not the folklore.

The structure is the one from `00-program/assessment-and-rubrics.md`: claim, grounds, the
strongest rebuttal you can construct, and the limits of your position. A paper that does not name
a condition under which its claim fails has not made a claim.

---

## Submission checklist (per set)

- [ ] Code in `courses/ds701/psN/`, runnable from a clean clone via the README.
- [ ] Tests passing, with a stated coverage figure *and* a sentence on what coverage does not tell
      you here.
- [ ] `mypy --strict` and `ruff` clean, or every exception documented with a reason.
- [ ] All measurements reproducible: every fault-injection run is seeded and the seed recorded, so any reported failure is reproducible.
- [ ] Charts and tables as files, not as descriptions — a plot referred to but not produced does not
      count.
- [ ] `NOTE.md` — the design note.
- [ ] `MARK.md` — self-marked against the five-criterion rubric with one line of justification per
      criterion.
- [ ] `log/failures.md` updated with everything you got wrong on the way, including the predictions
      that were incorrect.
