# FM-751 · Lesson 06 — Specifying a Distributed Protocol

**Estimated study time:** 5 hours
**Prerequisites:** L05; DS-701 L03–L08

---

## 1. Orientation

This is the lesson where the course's claim gets tested. Everything so far has been technique; this
lesson applies it to the thing it was built for — a real distributed protocol, with failures, with
non-determinism, with the interleavings that testing cannot reach.

The reason distributed protocols are the killer application for model checking is precisely DS-701's
diagnosis: **the bugs are in interleavings nobody imagined, under failures nobody enumerated, and
the system cannot be stepped through.** A model checker imagines all of them, mechanically, in
minutes.

The lesson's specific content is the two skills that separate a specification that finds bugs from
one that does not:

1. **Modelling the environment's misbehaviour** — the network, the crashes, the timing. A
   specification of the happy path finds nothing, because the happy path was never where the bug
   was.
2. **Strengthening an invariant until it is inductive** — the central technical difficulty of
   protocol verification, and the thing that is genuinely hard.

## 2. Theory

### 2.1 The anatomy of a protocol specification

A reusable structure, which you should adopt:

```
CONSTANTS    Nodes, Values, MaxTerm        \* small, and symmetric where possible
VARIABLES    nodeState,                    \* per-node local state
             messages,                     \* the network: a set or bag of in-flight messages
             history                       \* auxiliary state, for stating properties only

TypeOK       \* the shape of everything
Init         \* every node in its initial state, no messages in flight

\* Protocol actions
Send(...)    \* a node acts and emits messages
Receive(...) \* a node processes a message and updates state

\* Environment actions — where the bugs are
Drop         \* remove a message without delivering it
Duplicate    \* deliver a message and leave it in flight
Crash        \* a node stops
Restart      \* a node returns, with volatile state lost and durable state kept

Next == protocol actions \/ environment actions
Spec == Init /\ [][Next]_vars /\ Fairness
```

Three things to note.

**The network is state.** Modelling it as a *set* of in-flight messages gives you reordering and
duplication for free — any message in the set can be delivered at any time, and delivery need not
remove it if you also model duplication. A *sequence* per channel gives FIFO, which is a stronger
assumption you must justify. Choosing between them is a decision about what your transport actually
guarantees.

**Auxiliary (history) variables** are state you add solely to state properties — the set of all
values ever committed, the sequence of client operations. They do not affect the protocol's
behaviour, and they make properties expressible that otherwise are not. Use them freely, and mark
them clearly.

**Crash and restart must distinguish durable from volatile state.** A node that restarts with its
in-memory state intact is not a crash; it is a pause. Getting this distinction right is where
specifications of consensus protocols earn their value, because the whole point of Raft's persistence
requirements (DS-701 L05) is about exactly this boundary.

### 2.2 Modelling failure

The failure model (DS-701 L01) is the assumptions section (L01 §2.2), encoded. Enumerate
deliberately:

- **Message loss**: an action that removes a message from the network without delivering it.
- **Duplication**: delivery without removal.
- **Reordering**: free if the network is a set.
- **Delay**: implicit — a message can sit in the set arbitrarily long.
- **Corruption**: usually excluded, and the exclusion should be an explicit `ASSUME` or a comment,
  because it is a real assumption about your transport's checksums.
- **Node crash**: the node stops taking steps. Volatile state is lost.
- **Node restart**: durable state survives, volatile is reinitialised.
- **Byzantine behaviour**: usually excluded; say so.
- **Partition**: a subset of nodes cannot exchange messages. Model as a partition relation
  constraining which deliveries are enabled, and add a heal action.
- **Clock skew**: if the protocol uses time, model it. Most specifications avoid time entirely by
  making timeouts non-deterministic actions rather than time-driven ones — which is both simpler and
  a more honest reflection of an asynchronous system (DS-701 L01).

The judgement is what to include. Too little and you verify the happy path; too much and the state
space explodes. **Start with loss, duplication, reordering and crash — those four find most bugs.**
Add partition when the protocol's correctness depends on quorums.

The honesty obligation: **whatever you exclude is an assumption, and it belongs in the
specification's header as an explicit statement.** Every "we proved it correct and it failed in
production" story is a story about an excluded failure mode.

### 2.3 Inductive invariants: the hard part

You want to prove `□P`. The technique is to find an invariant true initially and preserved by every
step. The difficulty is that **the property you care about is usually not inductive**:

> `I` is **inductive** if `Init ⇒ I` and `I ∧ Next ⇒ I'`. Note the second clause: it must be
> preserved assuming *only* `I`, not assuming reachability.

Concretely: "there is at most one leader per term" is true of every reachable state of Raft, but you
cannot prove it preserved from that fact alone — you must also know that a node only votes once per
term, and that a leader was elected by a majority, and that any two majorities intersect. Those
extra facts are the **strengthening**.

The workflow, which is iterative and is the real work of protocol verification:

1. State the property `P` you want.
2. Try `I = P`. Check whether `I ∧ Next ⇒ I'`.
3. Where it fails, the checker (or your reasoning) shows a state satisfying `I` from which a step
   reaches `¬I'`. **Look at that state: is it reachable?** If not, add a conjunct to `I` that
   excludes it.
4. Repeat until inductive.

The final `I` is often several times longer than `P`, and it encodes the protocol's real design
rationale — the facts the designers were implicitly relying on. **Writing it down is frequently the
most valuable output of the whole exercise**, more than the check itself, because it makes explicit
what the protocol depends on.

Two practical notes. First, TLC checks invariants over *reachable* states, which is weaker than
inductiveness and is usually what you want for finding bugs; inductiveness matters when you want a
proof that holds for unbounded configurations (via TLAPS or `Apalache`). Second, an inductive
invariant that is hard to find usually indicates a protocol that is hard to reason about, which is
useful design feedback in itself.

### 2.4 Stating the properties

For a replicated data store, the properties you should be stating:

**Safety**
- **Agreement**: no two nodes decide differently for the same slot.
- **Validity**: a decided value was proposed by someone.
- **Durability**: an acknowledged write is never lost.
- **Consistency**: the history is linearizable, or causally consistent, or whatever you claim
  (DS-701 L04).
- **Uniqueness**: at most one leader per term.

**Liveness**
- **Progress**: with a stable majority, some value is eventually decided.
- **Availability**: with a stable majority, every request is eventually answered.

Two techniques you will need:

**Checking linearizability in TLA+.** Keep a history variable recording client operations with
their invocation and response points, and state that there exists a sequential ordering consistent
with the real-time order and with the sequential specification. This is expensive to check and it is
the property you actually claim, so it is worth doing at very small scope (DS-701 L04 defined it;
this is where you check it).

**Refinement as the property.** Rather than enumerating properties, specify an *abstract* system —
a single-node data store with atomic operations — and prove the protocol refines it (L07). Then
every property of the abstract system holds for the protocol, and you have one theorem instead of
seven. This is TLA+'s intended idiom and the subject of the next lesson.

### 2.5 A worked structure: replicated log

The shape you will build in §3, for orientation:

- `log[n]` — a sequence of entries per node.
- `commitIndex[n]` — how much of the log is committed, per node.
- `currentTerm[n]`, `votedFor[n]` — durable election state.
- `state[n] \in {Follower, Candidate, Leader}`.
- `messages` — a set of records, each with a type, sender, recipient and payload.
- Actions: `Timeout` (become candidate, increment term), `RequestVote`, `HandleRequestVote`,
  `BecomeLeader` (on a majority), `ClientRequest`, `AppendEntries`, `HandleAppendEntries`,
  `AdvanceCommitIndex`, plus the environment actions.

The properties to check, in order of how much they teach:

1. `TypeOK`.
2. **Election safety**: at most one leader per term. Fails fast if the voting rule is wrong.
3. **Log matching**: if two logs have an entry with the same index and term, all preceding entries
   match. This is the one whose inductive strengthening is instructive.
4. **Leader completeness**: a committed entry is present in the log of every future leader. This is
   the property that makes Raft correct, and the one that requires the most strengthening.
5. **State machine safety**: no two nodes apply different entries at the same index.
6. **Liveness**: with a stable majority and fairness, a leader is eventually elected.

**Deliberately break the voting rule and the log-comparison rule** and watch which property fails
first, and with what counterexample. That exercise teaches more about why Raft is designed the way
it is than reading the paper twice.

### 2.6 Managing the state space here

Protocol specifications explode fastest, so:

- **Three nodes.** Two is often too few to exercise majorities; four is usually unaffordable. Three
  is the sweet spot and finds the great majority of bugs.
- **Two values, one or two clients.**
- **Bound the term number** with a state constraint, and report the bound. Unbounded terms means
  unbounded states.
- **Bound the log length.**
- **Symmetry** over node identifiers — a large saving, and free.
- **Bound the number of failures**: at most one crash, at most two message losses. This is a real
  restriction on what you have checked and must be reported.
- **Check safety first at three nodes; check liveness at two**, because liveness checking is much
  more expensive.

And the reporting obligation: the result is "no violation of these properties in any execution with
three nodes, two values, terms bounded by four, and at most one crash". That sentence is the
deliverable, and it is a strong claim honestly stated.

## 3. Construction: specify Raft (or your protocol) and find a bug

Build in `mpse/fm751/l06/`. This is the largest construction in the course and produces the term
artifact's specification.

**Stage 1 — the environment module.** Complete the message-passing model from L05 Stage 5 into a
reusable module: network as a set, actions for send, deliver, drop, duplicate; crash and restart
distinguishing durable from volatile state; optional partition and heal. Verify each behaviour is
genuinely possible with a temporary refuting invariant.

**Stage 2 — a simple protocol first.** Before Raft, specify something small with a real bug in it: a
naive leader election that elects on a plurality rather than a majority. Check election safety. Read
the counterexample. This calibrates your reading of counterexamples on a protocol simple enough to
hold in your head.

**Stage 3 — the Raft skeleton.** Variables, `TypeOK`, `Init`, and just two actions: `Timeout` and
`RequestVote`/`HandleRequestVote`. Check `TypeOK` and election safety at two nodes. Grow to three.

**Stage 4 — break the voting rule.** Remove the "one vote per term" restriction. Find the
counterexample to election safety and read it carefully. Then remove the majority requirement and
find that counterexample. Write both up. **These two counterexamples are the reason those rules
exist**, and having produced them yourself is different from having read them.

**Stage 5 — log replication.** Add `ClientRequest`, `AppendEntries`, `HandleAppendEntries` and
`AdvanceCommitIndex`. Check log matching. Then break the log-consistency check in `AppendEntries` —
accept an entry without verifying the previous entry's term — and find the counterexample.

**Stage 6 — leader completeness and the inductive invariant.** State leader completeness. Then do
the §2.3 workflow: attempt to make it inductive, find the states where preservation fails, and add
conjuncts until it holds. Record every conjunct you added and *why* — that list is the protocol's
design rationale, recovered.

**Stage 7 — failures.** Add crash and restart. Determine, by experiment, which state must be durable
for the safety properties to hold: make `currentTerm` volatile and find the violation; make
`votedFor` volatile and find the violation; make the log volatile and find the violation. **This
stage derives Raft's persistence requirements from first principles**, and it is the most
satisfying exercise in the lesson.

**Stage 8 — liveness.** Add fairness and check that a leader is eventually elected with a stable
majority. Then find the fairness assumption it requires, and check whether your implementation
provides it. Then remove the randomised timeout (make all nodes time out simultaneously) and observe
the split-vote livelock as a lasso counterexample.

**Stage 9 — the state space report.** Measure states against node count, term bound and log bound.
Apply symmetry. Determine your largest feasible configuration. Then write the honest results
statement in the form of §2.6.

**Stage 10 — your own protocol.** Now do the same for a protocol *you* designed: your DS-701 saga
engine, your CRDT merge, your CA-731 platform's tenant provisioning state machine, or a protocol
from work. Specify it, state its properties, check it. Report what you found — and if you found
nothing, report what writing the specification forced you to decide.

## 4. Failure modes

- **Modelling only the happy path.** The bugs are in the failure interleavings; a specification
  without them finds nothing and creates false confidence.
- **A FIFO network model when the transport is not FIFO.** Verifies a system you do not have.
- **Crash without distinguishing durable from volatile state.** Misses exactly the bugs the
  persistence rules exist to prevent.
- **Using the property directly as an invariant** and giving up when it is not preserved. It usually
  is not; strengthen it.
- **An unreported state constraint.** You checked a bounded system; the claim must say so.
- **Unbounded terms or log length.** The check never terminates.
- **Two nodes only.** Majority-based bugs need three.
- **Liveness at three nodes.** Usually too expensive; check it at two and say so.
- **Modelling time explicitly.** Almost always unnecessary and always expensive; make timeouts
  non-deterministic actions.
- **Failing to enumerate excluded failure modes.** They are assumptions, and they belong in writing.

## 5. Exercises

### Warm-up (30 min)

1. Give the anatomy of a protocol specification and explain why the network is state.
2. Explain what an auxiliary variable is for, with an example property that needs one.
3. Define an inductive invariant and explain why the property you want is usually not one.

### Core (4 h)

4. Complete Stages 1–3: the reusable environment module, the naive election with its
   counterexample, and the Raft skeleton checked at three nodes.
5. Complete Stages 4–5: four deliberate rule violations with their counterexamples, written up.
6. Complete Stage 7 and derive the persistence requirements experimentally.
7. Complete Stage 9 and write the honest results statement.

### Challenge

8. Complete Stages 6 and 8: the inductive invariant with every conjunct justified, and liveness with
   the split-vote livelock produced as a lasso.
9. Complete Stage 10, then take a **published protocol and try to find a bug in it** — not a famous
   one, but a protocol from a recent paper, an internal design document, or an RFC draft. Specify it
   faithfully, state the properties its authors claim, and check them. Report the outcome honestly:
   a bug found, an ambiguity in the specification that permits two incompatible implementations, or
   a clean check with the bounds stated. All three are publishable-quality outcomes at this level,
   and the second is the most common — Zave's Chord result and several of the AWS findings began
   exactly this way. If you find something, tell the authors; that is how this work contributes.

## 6. Self-check

1. Give the anatomy of a protocol specification with the role of each part.
2. Why does modelling the network as a set give reordering and duplication for free?
3. What is an auxiliary variable and when do you need one?
4. Why must crash modelling distinguish durable from volatile state?
5. List the failure modes to model, and say which four find most bugs.
6. Define inductive, and give the four-step strengthening workflow.
7. Why is the final invariant often more valuable than the check itself?
8. Give the six properties to check for a replicated log, in order.
9. Give six state space controls specific to protocol specifications.
10. State the form of an honest results claim after a protocol check.

## 7. Primary sources

- **Ongaro & Ousterhout, "In Search of an Understandable Consensus Algorithm" (USENIX ATC 2014)**,
  and **Ongaro's Raft TLA+ specification** — read the specification alongside the paper.
- **Newcombe et al., "How Amazon Web Services Uses Formal Methods" (CACM 2015)** — the bug
  descriptions, read now that you can produce such bugs yourself.
- **Lamport, *Specifying Systems*, Part II** — the distributed algorithm examples.
- **Zave, "Using Lightweight Modeling to Understand Chord" (SIGCOMM CCR 2012)** — a published
  protocol found incorrect; the model for Stage 10's challenge.
- Lamport, "Paxos Made Simple" (2001), and the Paxos specifications in `tlaplus/Examples`.
- Kuppe, Lamport & Ricketts, "The TLA+ Toolbox" (F-IDE 2019) — practical tooling.
- Konnov, Kukovec & Tran, "TLA+ Model Checking Made Symbolic" (OOPSLA 2019) — Apalache, for
  inductive invariant checking and larger scopes.
- DS-701 L05 and L08, re-read alongside your own specification. The paper's prose arguments will
  read differently once you have produced the counterexamples they were defending against.

---

**Previous:** [L05](L05-tla-plus.md) · **Next:** [L07 — Refinement](L07-refinement.md)
