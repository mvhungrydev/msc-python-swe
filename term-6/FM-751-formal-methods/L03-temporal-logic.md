# FM-751 · Lesson 03 — Temporal Logic: Safety and Liveness

**Estimated study time:** 4.5 hours
**Prerequisites:** L01, L02; DS-701 L04, CS-641 L02

---

## 1. Orientation

Hoare logic (L02) reasons about a program that starts, runs and finishes. That model does not fit
the systems this course cares about: a server does not terminate, a distributed protocol has no
single thread of control, and the interesting properties are about what happens *over time* and
*across interleavings* — "two nodes never both believe they are leader", "every request is
eventually answered".

**Temporal logic** is the notation for those properties. It extends ordinary logic with operators
about time, so that a formula is evaluated not at a state but over a **behaviour** — an infinite
sequence of states (L01 §2.3).

The lesson has two jobs. The first is the notation, which is small: four operators do almost
everything. The second, and more important, is the distinction that organises all verification:

> **Safety** properties are violated by a finite prefix — you can point at the step where it went
> wrong. **Liveness** properties are violated only by an infinite behaviour — there is no bad step,
> only an eternity in which the good thing never happens.
>
> They need different specification constructs, different verification algorithms, and different
> engineering responses. Almost every muddled requirements discussion is a failure to separate them.

## 2. Theory

### 2.1 Behaviours and the operators

A **behaviour** is an infinite sequence of states `σ = s₀, s₁, s₂, …`. A temporal formula is true or
false *of a behaviour*.

The core operators (linear temporal logic, LTL — the fragment TLA+ uses):

| Operator | Read | True of σ when |
|---|---|---|
| `P` | (a state predicate) | P is true of s₀ |
| `□P` | **always** P | P is true of every suffix (every state) |
| `◇P` | **eventually** P | P is true of some suffix (some state) |
| `P ⇝ Q` | P **leads to** Q | whenever P holds, Q holds then or later |
| `○P` | next P | P holds of s₁ (used sparingly; see §2.4) |

`◇` and `□` are duals: `◇P ≡ ¬□¬P`. And `P ⇝ Q` is defined as `□(P ⇒ ◇Q)`, which is the most
useful derived operator in practice — most liveness requirements are leads-to properties.

The combinations that carry real meaning:

- **`□◇P`** — "infinitely often P". P recurs forever; it never stops happening.
- **`◇□P`** — "eventually always P". P becomes permanently true and stays. Stabilisation.

The difference between those two is worth dwelling on because it is the source of many
specification errors. "The system is eventually always available" (`◇□available`) is a much stronger
claim than "the system is available infinitely often" (`□◇available`) — the latter permits it to
alternate between up and down forever.

### 2.2 Safety, formally

> A **safety** property is one whose violation is witnessed by a **finite prefix**. Once violated,
> no continuation can repair it.

The characteristic form is `□(invariant)`. Examples:

- Mutual exclusion: `□(¬(inCS₁ ∧ inCS₂))`
- Election safety (DS-701 L05): `□(∀ t : |{n : leader(n) ∧ term(n) = t}| ≤ 1)`
- Type safety (CS-641 L03): a well-typed program never reaches a stuck state.
- Partial correctness (L02): if it terminates, the postcondition holds.
- No lost acknowledged writes.

Why the finite-prefix property matters practically: **a safety violation has a concrete
counterexample you can print and read.** A model checker that finds one hands you a finite trace —
"here are the eleven steps that produce two leaders" — which is directly actionable. This is the
single most useful thing model checking does for engineers, and it is why L04 emphasises reading
counterexamples.

Proving safety is by **invariant**: find a predicate true initially and preserved by every step,
strong enough to imply the property. Exactly L02's technique, lifted from loops to systems. The
usual difficulty is that the property you want is not itself inductive, and you must **strengthen**
it — add clauses until it becomes preserved by every step — which is the central skill of L06.

### 2.3 Liveness, formally

> A **liveness** property is one that no finite prefix can violate — any finite behaviour can still
> be extended to satisfy it.

Characteristic forms are `◇P` and `P ⇝ Q`. Examples:

- Termination: `◇done`
- Every request answered: `□(request ⇒ ◇response)`, i.e. `request ⇝ response`
- Eventual delivery: `sent ⇝ received`
- Progress in consensus (DS-701 L05): `◇(∃ v : decided(v))`
- Starvation freedom: `waiting ⇝ inCS`

The counterexample to a liveness property is an **infinite behaviour** — usually presented as a
finite prefix plus a **lasso**: a cycle the system can loop around forever without ever satisfying
the property. Model checkers find these by looking for reachable cycles containing no "good" state,
and reading a lasso counterexample is a distinct skill from reading a safety trace.

The engineering point: **liveness properties are the ones nobody tests**, because a test that runs
for a finite time cannot distinguish "slow" from "never" — which is DS-701 L01's impossibility
result appearing as a testing problem. Model checking is one of the few practical ways to get
evidence about them.

### 2.4 Fairness

Liveness properties are almost never true without fairness assumptions, because a system that simply
stops taking steps violates every liveness property while violating no safety property.

Two standard strengths:

- **Weak fairness** (WF): if an action is *continuously* enabled, it eventually occurs. Rules out a
  process that could always proceed but never does.
- **Strong fairness** (SF): if an action is enabled *infinitely often*, it eventually occurs. Rules
  out a process that is repeatedly given a chance and repeatedly skipped.

The practical rule: **use weak fairness by default and strong fairness only where the design
genuinely requires it**, because SF is a stronger assumption about the implementation — and an
assumption about the scheduler, the network, or the operating system that must actually hold.
Assuming SF where the real system only provides WF proves a liveness property that the real system
does not have.

Fairness is where the specification's assumptions (L01 §2.2) become explicit and consequential:
"messages are eventually delivered if retried infinitely often" is a fairness assumption on the
network, and DS-701's whole treatment of failure detectors is about what happens when it does not
hold.

### 2.5 The decomposition theorem

Alpern and Schneider: **every property is the intersection of a safety property and a liveness
property.** Any requirement can be split into "nothing bad happens" and "something good eventually
happens".

This is a useful working habit rather than a piece of theory: when a requirement is confused, split
it. "The system is highly available" decomposes into a safety part (it never returns a wrong answer)
and a liveness part (it eventually returns an answer), and those two are traded against each other
by every design decision in DS-701.

The pattern recurs everywhere once you look for it:

| Domain | Safety | Liveness |
|---|---|---|
| Mutual exclusion | never two in the critical section | every waiter eventually enters |
| Consensus | never two different decisions | eventually some decision |
| Transactions | never a lost committed write | every transaction eventually commits or aborts |
| Cache | never serve inconsistent data | eventually reflects the write |
| CAP (DS-701 L04) | consistency | availability |

The last row is the point: **CAP is the statement that a certain safety property and a certain
liveness property cannot both hold under partition.** Seeing it that way makes it much clearer than
the folklore version.

### 2.6 TLA+'s formulation, and why it differs

TLA+ (L05) uses a variant of temporal logic designed so that specifications compose and refinement
works (L07). Its key ideas:

- **Actions** are predicates on *pairs* of states, relating the current state to the next. Primed
  variables denote the next state: `x' = x + 1`.
- A specification has the form `Init ∧ □[Next]_vars ∧ Fairness` — an initial predicate, and the
  requirement that every step either satisfies `Next` or leaves `vars` unchanged.
- **Stuttering steps** — steps that change nothing — are always allowed. This looks like a technical
  detail and is the crucial design decision: it makes a specification invariant under adding
  unobserved internal steps, which is exactly what refinement (L07) requires. A low-level
  implementation takes many steps where the high-level specification takes one; permitting
  stuttering is what lets the correspondence work.
- Consequently TLA+ avoids the `○` (next) operator, which is not stuttering-invariant. This is why
  §2.1 flagged it as used sparingly.

You do not need this machinery to write good properties, but you do need it to understand why TLA+
looks the way it does, and why the `[Next]_vars` bracket notation is there.

### 2.7 Writing properties that mean what you want

The common errors, each with its correction:

- **`◇□P` versus `□◇P`.** Stabilisation versus recurrence; check which you meant.
- **A vacuously true leads-to.** `P ⇝ Q` is trivially satisfied if P is never true. Always check
  that the antecedent is reachable — a model checker will happily verify a property about a state
  your system can never enter, and report success.
- **Safety stated as liveness.** "The system eventually becomes consistent" is liveness and
  unfalsifiable in finite time; if you meant "it is always consistent after a bounded delay", say
  that, with the bound.
- **Forgetting fairness.** The liveness property fails, and the counterexample is a system that
  simply stops. Add the fairness assumption — but only the one that is actually true of your
  implementation.
- **Properties that only constrain the good path.** State what must *not* happen on the error paths
  too; that is where the bugs are.
- **Confusing the property with the mechanism.** "The leader sends heartbeats" is a mechanism; "there
  is at most one leader per term" is the property. Specify the property, or you have ruled out every
  other mechanism (L01 §2.2).

## 3. Construction: express properties precisely

Build in `mpse/fm751/l03/`. Notation-heavy and tool-light — L04 and L05 bring the checker.

**Stage 1 — the operator drills.** For each of twenty English statements about a system, write the
temporal formula. Include the pairs that differ subtly: "always eventually" versus "eventually
always"; "P then Q" versus "P leads to Q"; "never P" versus "eventually never P". Then, for five of
them, write a behaviour that satisfies the formula and one that violates it.

**Stage 2 — classify.** Take twenty real requirements — from DS-701's protocols, from an RFC, from
your own systems, from an SLA — and classify each as safety, liveness, or a conjunction. For the
conjunctions, write the decomposition explicitly. Then note which of the liveness ones your existing
tests address; the answer is usually none, and that observation motivates the rest of the course.

**Stage 3 — the protocol properties.** For your DS-701 Raft implementation, write all five Raft
safety properties formally: election safety, leader append-only, log matching, leader completeness,
state machine safety. Then write its liveness property and, crucially, the fairness assumptions it
requires — and check those assumptions against what your implementation actually provides.

**Stage 4 — counterexample shapes.** For three safety properties from Stage 3, write out by hand a
finite trace that violates each. For two liveness properties, write out a lasso — a prefix plus a
cycle — that violates each. Doing this by hand before a tool does it for you is what makes the
tool's output readable in L04.

**Stage 5 — fairness, demonstrated.** Take a simple two-process mutual exclusion protocol. Write
its safety property and its starvation-freedom property. Then: show that starvation freedom fails
with no fairness; show it holds under strong fairness; and determine whether it holds under weak
fairness. Then state which fairness a real scheduler actually provides, and what that means for the
guarantee.

**Stage 6 — the vacuous property.** Write a leads-to property whose antecedent your system can never
satisfy. Convince yourself it is trivially true. Then write the reachability check that would have
caught it — an assertion that the antecedent state is reachable, checked separately. Adopt this as a
habit now; it will save you from a false sense of security in L06.

**Stage 7 — DS-701 revisited.** Express CAP formally: state the safety property (linearizability),
the liveness property (availability), and the assumption (partition), and state the impossibility as
a temporal claim. Do the same for FLP. Then write, for each of three real systems you know, which
side of each trade they chose, in this notation.

**Stage 8 — a trace checker.** Write a small program that, given a finite trace of states and a
temporal formula from a restricted grammar (`□P`, `◇P`, `P ⇝ Q` over state predicates), reports
whether the trace satisfies, violates, or leaves undetermined the formula. Note what "undetermined"
means: a finite trace can definitively violate a safety property, can definitively satisfy a
liveness property up to that point, and can never definitively establish `□P` or definitively refute
`◇P`. Getting that logic right *is* understanding §2.2 and §2.3.

**Stage 9 — apply it to real traces.** Run your Stage 8 checker against traces from your DS-701
implementation's chaos runs (L10 of that course). Report any safety violations found. Then state
what the checker's silence does and does not tell you — this is runtime verification, previewed
(L10), and its epistemics are exactly what L01 warned about.

## 4. Failure modes

- **`◇□` where `□◇` was meant, or vice versa.** Very different claims.
- **Vacuous leads-to.** Verified successfully, about a state that cannot occur.
- **Liveness without fairness.** Every liveness property fails against a system that does nothing.
- **Assuming strong fairness the implementation does not provide.** Proves a property the real
  system lacks.
- **Testing for liveness.** A finite run cannot distinguish slow from never.
- **Specifying the mechanism instead of the property.** Rules out valid implementations and does not
  check the thing you cared about.
- **Only constraining the happy path.** The bugs are on the error paths.
- **Safety properties that are not inductive**, used directly as invariants without strengthening —
  L06's central difficulty.
- **Using `○` (next).** Not stuttering-invariant; breaks refinement.
- **Believing a checked property covers unmodelled assumptions.** It covers the model.

## 5. Exercises

### Warm-up (30 min)

1. Give the four core temporal operators and the two important combinations, with the difference
   between the combinations explained.
2. Define safety and liveness by how each is violated, and say what shape the counterexample takes
   in each case.
3. Distinguish weak and strong fairness, and say which you should assume by default and why.

### Core (3.5 h)

4. Complete Stages 1–2: twenty formulas with satisfying and violating behaviours for five, and
   twenty requirements classified with the conjunctions decomposed.
5. Complete Stage 3: all five Raft safety properties formally, plus liveness and the fairness
   assumptions checked against your implementation.
6. Complete Stage 4: three safety counterexamples and two lassos, written by hand.
7. Complete Stage 5, including which fairness a real scheduler provides.

### Challenge

8. Complete Stages 6–7: the vacuity check adopted as a habit, and CAP and FLP expressed formally
   with three real systems classified.
9. Complete Stages 8–9, then extend the trace checker into a **runtime monitor** that evaluates
   temporal properties incrementally as a distributed system runs — maintaining just enough state to
   report a safety violation the moment it occurs, and reporting liveness properties as
   "unsatisfied so far" with the elapsed time. Deploy it alongside your DS-701 implementation during
   chaos experiments. Then write the honest assessment: which properties can be monitored at
   runtime, which cannot, what the monitoring costs, and what a clean run establishes. The answer to
   that last question — "that these particular executions did not violate these particular safety
   properties" — is much weaker than a model check, and knowing exactly how much weaker is the
   professional skill this course is building.

## 6. Self-check

1. Define a behaviour, and say what it means for a temporal formula to be true.
2. Give the four operators and define `⇝` in terms of them.
3. Distinguish `□◇P` from `◇□P` with an example where the difference matters.
4. Define safety and liveness by their violating witnesses.
5. What shape does a liveness counterexample take, and why?
6. Why are liveness properties the ones nobody tests?
7. Distinguish weak and strong fairness, and state the hazard of assuming the wrong one.
8. State the decomposition theorem and apply it to consensus and to CAP.
9. What is a stuttering step, why does TLA+ permit it, and what does that enable?
10. Give five ways a temporal property can fail to mean what you intended.

## 7. Primary sources

- **Lamport, *Specifying Systems*, chapters 2–3 and 8** — temporal logic as TLA+ uses it.
- **Lamport, "The Temporal Logic of Actions" (TOPLAS 1994)** — the source; difficult, and worth
  attempting after L05.
- **Alpern & Schneider, "Defining Liveness" (IPL 1985)** and "Recognizing Safety and Liveness"
  (Distributed Computing, 1987) — the formal definitions and the decomposition theorem.
- Pnueli, "The Temporal Logic of Programs" (FOCS 1977) — where LTL entered computer science.
- Manna & Pnueli, *The Temporal Logic of Reactive and Concurrent Systems* — the comprehensive
  treatment.
- Baier & Katoen, *Principles of Model Checking*, chapter 5 — LTL, clearly presented, with the
  automata connection L04 needs.
- Owicki & Lamport, "Proving Liveness Properties of Concurrent Programs" (TOPLAS 1982) — where
  fairness is developed carefully.
- DS-701 L04 and L05, re-read with this notation available. The properties in those lessons were
  stated in English; write them formally and see what the exercise exposes.

---

**Previous:** [L02](L02-hoare-logic-and-invariants.md) · **Next:**
[L04 — Model Checking: How It Works and Where It Stops](L04-model-checking.md)
