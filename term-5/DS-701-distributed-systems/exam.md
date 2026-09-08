# DS-701 — Written Examination

**Time allowed: 3 hours. Closed book, no machine.**
**Answer FOUR from Section A and ONE from Section B.**
Section A: 15 marks each. Section B: 40 marks.

Where a question asks for a guarantee, state it precisely — what holds, for which operations,
under which conditions, and what a client may assume. Vague guarantees score nothing. Where a
question asks you to construct a history or a schedule, draw it as a timeline with explicit
events; a description in prose is worth at most half marks.

---

## Section A — answer FOUR

**A1.** Define the crash-stop, crash-recovery, omission and Byzantine failure models, and give
one real system or fault for each. Then explain what a *partial* failure is and why it is the
defining characteristic of a distributed system rather than merely a common one. Finally, state
what your failure model must say about disks, and why "the disk is durable" is an assumption
rather than a fact. *(15)*

**A2.** State the FLP impossibility result precisely, including every assumption. Explain what it
does *not* say. Then explain how Raft's randomised election timeouts relate to it — being clear
about whether they evade the result, weaken an assumption, or something else. *(15)*

**A3.** State CAP precisely, then explain the two most common misreadings of it. Give PACELC and
explain what it adds. Then classify three real systems in PACELC terms and defend each
classification. *(15)*

**A4.** Define the happens-before relation. Explain what a Lamport timestamp guarantees and what
it does not, and give a concrete pair of events that demonstrates the gap. Then define vector
clocks, state their space cost, and explain what a hybrid logical clock buys and what it assumes.
*(15)*

**A5.** Explain last-write-wins conflict resolution and construct, as an explicit timeline, a
scenario in which it silently loses an acknowledged write. State exactly which assumption fails.
Then give two alternatives and the cost of each. *(15)*

**A6.** State the quorum condition R+W>N and give the intersection argument for why it works. Then
construct a scenario in which R+W>N *still* returns a stale value, and explain precisely why the
intersection argument does not apply there. Finally, explain read repair and anti-entropy and what
each is for. *(15)*

**A7.** Define linearizability and sequential consistency, and give a history that distinguishes
them. Then explain why linearizability is *composable* and sequential consistency is not, and why
that property matters when a system has more than one object. *(15)*

**A8.** Define read-your-writes, monotonic reads, monotonic writes and writes-follow-reads. For
each, construct the anomaly that occurs in its absence. Then explain why session guarantees are
often the right target and what they cost compared with linearizability. *(15)*

**A9.** Explain the Raft leader election protocol, including terms, the voting rules and the
purpose of randomised timeouts. Then state the log matching property and explain the mechanism
that maintains it during `AppendEntries`. *(15)*

**A10.** Explain how a partitioned Raft leader can serve a stale read despite Raft being a
consensus protocol. Give two fixes, and for each state the assumption it depends on and what
happens when that assumption is violated. *(15)*

**A11.** Explain consistent hashing and the problem it solves, quantifying the improvement over
modulo hashing when a node is added. Explain the role of virtual nodes, and describe two
distributions of load that consistent hashing does *not* fix. *(15)*

**A12.** Describe the SWIM protocol: direct probe, indirect probe, suspicion and dissemination.
Explain what the incarnation number is for and construct the scenario that requires it. Then
explain why suspicion exists at all rather than direct eviction on a failed probe. *(15)*

**A13.** Define strong eventual consistency and state the three algebraic properties a CRDT merge
must satisfy. Explain precisely why those three properties make the merge safe over an unreliable
network — naming which network property each one neutralises. *(15)*

**A14.** Explain why deletion is hard in a CRDT. Describe tombstones and the garbage-collection
problem they create, and explain why that garbage collection generally requires coordination —
which is the thing CRDTs exist to avoid. *(15)*

**A15.** State the invariant "the account balance must never go negative" and prove informally
that no CRDT can enforce it under partition. Then describe the escrow pattern, state exactly what
it guarantees, and identify what has been given up. Relate your answer to the CALM theorem. *(15)*

**A16.** State the atomic commit problem and explain why it requires unanimity where consensus
requires only a majority. Derive from that the availability consequence, and explain why a
distributed transaction's failure probability compounds across participants. *(15)*

**A17.** Describe two-phase commit in both phases, being precise about what a `YES` vote commits a
participant to. Then explain the in-doubt window, state what a participant may and may not do
while in it, and explain why no protocol can eliminate it in an asynchronous system. *(15)*

**A18.** Explain the dual-write problem with a concrete example. Describe the transactional outbox
pattern and state exactly how many systems are written to atomically. Explain the delivery
guarantee that results and what must be true of consumers because of it. *(15)*

**A19.** Define a saga and state its guarantee in ACID terms, identifying the property it does not
provide. Explain why compensation is semantic rather than physical, and give an operation with no
compensation and the design consequence. *(15)*

**A20.** Give four saga isolation countermeasures and the anomaly each addresses. Then explain the
"by value" countermeasure and what its existence concedes about the whole approach. *(15)*

**A21.** Explain why exactly-once delivery is impossible and exactly-once effect is achievable.
Give four techniques for making a non-idempotent operation idempotent, and explain why an
idempotency key must be persisted in the same transaction as the effect. *(15)*

**A22.** Define completeness and accuracy for failure detectors. State what ◇S is and the result
that makes it important. Then explain the design principle "safety must never depend on a timeout,
only liveness may", and give a design that violates it and the corruption that follows. *(15)*

**A23.** Explain how to choose a timeout defensibly. Then explain deadline propagation, why a
deadline composes across a call chain where a duration does not, and why a timeout without
downstream cancellation makes recovery harder rather than easier. *(15)*

**A24.** Explain retry amplification in a three-layer call graph with per-call retry limits.
Describe the retry budget and what it bounds that a per-call limit does not. Then explain why full
jitter is required rather than a refinement. *(15)*

**A25.** Define metastable failure and state its diagnostic property. Give the general structure
of trigger and sustaining effect, illustrate with a retry-driven example and one non-retry-driven
example, and give the structural fixes. *(15)*

**A26.** Describe the circuit breaker state machine. Explain whom it primarily protects, why it
must trip on failure *rates* over a window rather than consecutive failures, and construct the
case in which a badly tuned breaker converts partial availability into total unavailability.
*(15)*

**A27.** Explain the arithmetic of tail latency under fan-out, then describe hedged requests and
tied requests. State the two conditions required to use hedging and what happens to the "5% extra
load" figure if one of them is not met. *(15)*

**A28.** Distinguish monitoring from observability in terms of the questions each can answer.
Explain cardinality and why it is the dividing line, and give a question that metrics cannot
answer by construction. Then explain why one wide event per unit of work beats fifteen log lines.
*(15)*

**A29.** Describe the trace/span model and context propagation. Compare head-based, tail-based and
weighted sampling, being specific about what head-based sampling discards that you needed. Then
explain why a child span can appear to start before its parent. *(15)*

**A30.** Define SLI, SLO and error budget. Explain burn-rate alerting and what multi-window
multi-burn-rate policies buy over a fixed threshold, using a 99.9%/30-day SLO in your worked
example. Then explain why averaging percentiles across instances is meaningless. *(15)*

---

## Section B — answer ONE

**B1. Design and defend.** You are designing the storage layer for a system with three
requirements: (i) a user's own writes must always be visible to them immediately; (ii) the system
must accept writes during a network partition between its two regions; (iii) a specific invariant
— no two users may hold the same username — must never be violated.

Design it. Your answer must: state which of L04's consistency models applies to which operations
and why; identify the operation that requires coordination and prove informally that it does;
describe your replication and quorum configuration with the arithmetic; state your failure model
and what you do not handle; describe what happens during and after a partition for each of the
three requirements; and identify the requirement that is in tension with the others and how you
resolved that tension. Then state, in one paragraph, what a client actually gets. *(40)*

**B2. The incident.** A payment service is degraded. The database's p99 rose from 20 ms to 800 ms
for ninety seconds because of an unrelated backup job. Four hours later the system is still down,
the database is healthy, and traffic is at normal levels. Some customers have been charged twice.

Write the analysis. Identify the trigger and the sustaining effect and name the mechanism that
should have bounded it. Explain the double charges precisely, in terms of what a timeout does and
does not tell the caller, and state the two changes that would have prevented them. Explain why
the system did not recover on its own and what an operator must now do to recover it. Then give
the six changes you would make, ranked, each with the specific failure it prevents and an honest
statement of its cost. *(40)*

**B3. Consensus or convergence.** A collaborative document editing product must work offline on
phones for days at a time, sync when connectivity returns, support real-time collaboration when
online, and enforce a per-document access control list.

Argue for a design. Your answer must: identify which parts of the problem are coordination-free
and which are not, using the CALM boundary explicitly; state what a CRDT-based design guarantees
about the document body and what it cannot guarantee; explain how the ACL is handled and why it
cannot be a CRDT in the same way; describe what happens when a user is removed from the ACL while
another user is offline with pending edits, and defend your choice of behaviour; and address
deletion, tombstones and garbage collection concretely. Finally, state the anomaly your design
permits that a consensus-based design would not, and defend permitting it. *(40)*

**B4. Prove it.** You have inherited a replicated key-value store whose documentation claims
"linearizable reads and writes, tolerating the failure of any one of five nodes."

Design the evidence that would justify or refute that claim, at the standard this course expects.
Your answer must: state precisely what the claim means and what would falsify it; describe the
test architecture, including how you generate histories, how you check them, and why a checker is
needed rather than assertions; enumerate the specific fault schedules that must be run and the
bug each is designed to expose; explain how you achieve reproducibility given that the bugs are
timing-dependent; explain what deterministic simulation gives you that random fault injection does
not, and its limitation; and state honestly what your testing can establish and what it cannot,
distinguishing "found no violation" from "there is no violation". Reference the Jepsen
methodology. *(40)*

**B5. The boundary.** A company has thirty microservices, and a common complaint is that "we need
distributed transactions." You have been asked to advise.

Write the advice. Your answer must: explain why the request is usually a symptom rather than a
requirement, using the aggregate/consistency-boundary argument from SE-521 L07; give the analysis
you would perform to determine whether a boundary is wrong; state the cases in which a genuine
distributed transaction *is* the right answer, and what it must be built on to be safe; describe
the saga alternative including its isolation problem and the countermeasures, being specific about
which countermeasure fits which situation; explain the outbox/inbox pattern and why it is the
foundation rather than an optimisation; and end with the three questions you would ask any team
that says it needs a distributed transaction. *(40)*

---

*Marks in Section A are awarded for precision. A correct mechanism described imprecisely — a
guarantee stated without its conditions, a protocol described without saying what is durable
before what — scores partial marks at best. In Section B, an answer that does not identify a
trade-off it made and a condition under which its design fails cannot reach the upper band,
however elegant the design.*
