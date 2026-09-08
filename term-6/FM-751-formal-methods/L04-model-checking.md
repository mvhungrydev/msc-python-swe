# FM-751 · Lesson 04 — Model Checking: How It Works and Where It Stops

**Estimated study time:** 4.5 hours
**Prerequisites:** L03; CS-621 L05 (graphs), CS-621 L10 (computability)

---

## 1. Orientation

Model checking is the technique that made formal methods practical for engineers, and the reason is
simple:

> **It is automatic.** You write a model and a property; the tool explores every reachable state and
> either verifies the property or hands you a concrete counterexample. No proof, no interactive
> theorem prover, no PhD.

The counterexample is the important half. A theorem prover that fails tells you it could not find a
proof, which might mean the property is false or might mean you were not clever enough. A model
checker that fails hands you eleven steps ending in two simultaneous leaders, and you read it and
say "oh — I never considered that the old leader's message could arrive after the election."

This lesson covers how it works — enough to understand its costs and its failure modes — and, more
importantly, **where it stops.** The state space explosion problem is not an implementation
weakness to be engineered away; it is the fundamental limit of the technique, and working with it
productively is the practical skill.

## 2. Theory

### 2.1 The basic algorithm

A model is a finite state machine: a set of states, an initial set, and a transition relation. The
checker explores.

**For a safety property** `□P`, the algorithm is a graph search:

```
frontier = initial_states
visited  = {}
while frontier:
    s = frontier.pop()
    if s in visited: continue
    visited.add(s)
    if not P(s): report counterexample (path from an initial state to s)
    frontier.extend(successors(s))
```

That is it. Breadth-first search gives the *shortest* counterexample, which is why checkers default
to it — a nine-step counterexample is readable and a four-hundred-step one is not.

**For a liveness property**, the machinery is more involved: negate the property, build a Büchi
automaton accepting exactly the behaviours violating it, take the product with the model, and search
for a reachable **accepting cycle** — a lasso (L03 §2.3). The nested depth-first search that finds
one is the standard algorithm. You do not need to implement it, but knowing it is a cycle search
explains why liveness checking is more expensive and why its counterexamples look different.

The essential property, and the reason this is worth doing: **the search is exhaustive over the
model.** Every reachable state, every interleaving. Testing samples; model checking enumerates.

### 2.2 State space explosion

The fundamental limit:

> The number of states is the **product** of each component's state count. Adding a variable with
> *k* values multiplies the state space by *k*. Adding a process multiplies it by that process's
> state count. **The state space is exponential in the size of the description.**

Concretely: three processes, each with five local states, communicating over channels holding up to
three messages from an alphabet of four — the arithmetic runs into the millions immediately, and a
fourth process runs into the billions.

This is not a bug in the tools. It is the reason model checking is applied to *designs* rather than
implementations, and the reason the modelling skill is fundamentally about **abstraction**.

The mitigations, roughly in order of importance to a practitioner:

1. **Model less.** The most effective technique by a wide margin, and the one requiring judgement.
   Model the protocol, not the payload. Two values instead of 2³². Three nodes instead of a hundred.
2. **Symmetry reduction.** If nodes are interchangeable, states differing only by permutation are
   equivalent. Enormous reductions for symmetric protocols, and TLA+ supports it directly with
   symmetry sets.
3. **Partial order reduction.** Independent concurrent actions can be explored in one order rather
   than all orders, because the outcome is the same. Automatic in most checkers, and often the
   difference between feasible and not.
4. **Symbolic model checking.** Represent sets of states as boolean formulas (BDDs) rather than
   enumerating them; sets that are astronomically large can have compact representations. This is
   what made hardware verification practical.
5. **Bounded model checking.** Ask only "is there a counterexample within *k* steps?", encode it as
   a SAT/SMT problem (L09), and solve. Incomplete — it cannot prove absence beyond *k* — and
   extremely good at finding shallow bugs quickly.
6. **Abstraction.** Replace the concrete model with a coarser one that over-approximates it. If the
   abstract model satisfies the property, so does the concrete one. Spurious counterexamples must be
   checked and used to refine the abstraction (CEGAR).

### 2.3 The small scope hypothesis

The empirical claim that justifies the whole practice:

> **Most bugs have small counterexamples.** A protocol flaw that requires seven nodes and forty
> steps to exhibit is rare; the overwhelming majority manifest with two or three nodes and a handful
> of steps.

This is not a theorem, and it does not have to be true. It is a repeated empirical finding — Jackson
formulated it for Alloy, and the AWS and Microsoft experience reports support it — and it is what
makes checking a three-node model useful evidence about a hundred-node deployment.

Its practical consequences:

- **Check small configurations first.** Two nodes, one client, two values. If the design is broken,
  this usually finds it, in seconds.
- **Grow until it stops being feasible**, and report the largest configuration checked. That is the
  honest statement of what you established.
- **Bugs found at small scope are real bugs.** A counterexample with two nodes is a counterexample.
- **Absence of a counterexample at small scope is evidence, not proof.** State the bound.

The honest framing to use with colleagues: "we exhaustively checked all executions with three nodes,
two terms and a network that can drop and reorder messages, and found no violation of these five
properties" is a specific and defensible claim. "We proved it correct" is not what you did.

### 2.4 Reading a counterexample

The most practically valuable skill in this lesson, and the one that is usually under-taught.

A safety counterexample is a **sequence of states**, each showing the values of all variables, with
the action taken between them. The procedure:

1. **Read the last state first.** What exactly is violated? Which conjunct of the property is false,
   and for which values?
2. **Work backwards** to find the step that made it inevitable. Often this is several steps before
   the violation.
3. **Identify the interleaving.** Which concurrent actions happened in an order you had not
   considered? This is almost always the answer, and it is almost always an order you would call
   "unlikely".
4. **Ask whether the model is wrong or the design is wrong.** A counterexample can be an artifact of
   an over-permissive model — you allowed a behaviour the real system cannot produce. Both outcomes
   are useful: the second is a bug, the first is an unstated assumption you have just discovered
   (L01 §2.2). Neither is a waste of time.
5. **Make the counterexample a regression test** in the implementation (L08).

A liveness counterexample is a **prefix plus a cycle**. The procedure is different: read the cycle
and ask *what keeps happening that prevents progress*. Common answers: a fairness assumption you
did not state; two components each waiting for the other; a retry loop that resets the very
condition it is waiting on.

The failure mode to avoid: **glazing over a long counterexample and concluding the tool is wrong.**
If the counterexample is long, the usual cause is a model with too much irrelevant detail — reduce
the model and re-run, and the counterexample shortens to something readable.

### 2.5 What model checking does not establish

Being precise here is a professional obligation, and overclaiming is how the technique gets
discredited.

- **It checks the model, not the code.** Unless you have a refinement argument (L07) or derive tests
  from the specification (L08), a verified model and a buggy implementation are entirely
  compatible. This is the largest gap in practice.
- **It checks the properties you wrote.** A property you did not think to state is not checked, and
  a vacuous property (L03 §2.7) is checked and passes meaninglessly.
- **It checks within the bounds you set.** Three nodes, not a hundred. Two values, not 2³².
- **It checks under the assumptions you modelled.** If your model assumes messages are not
  corrupted, it says nothing about a system where they are. If it assumes clocks are irrelevant, it
  says nothing about clock skew (DS-701 L02).
- **It does not check performance, cost, usability or operability** (CA-731 L10's other dimensions).

The corresponding statement of what it *does* establish is short and strong: **within these bounds,
under these assumptions, no execution violates these properties — and the search was exhaustive.**
Testing cannot make that claim at any scale.

### 2.6 The tools

- **TLA+ with TLC** (L05): explicit-state, excellent for distributed algorithms and concurrent
  designs, with a good counterexample display. The industrial standard for this kind of work, and
  what this course uses. TLAPS provides theorem proving for unbounded results when needed.
- **Alloy**: relational logic with a SAT backend, bounded by construction, superb for structural and
  data-model properties and for exploratory "show me an instance" work. Its visualiser is
  outstanding, and it is often faster to get started with than TLA+.
- **SPIN / Promela**: the classic explicit-state checker for communicating processes; heavily used
  in protocol verification.
- **NuSMV / nuXmv**: symbolic, BDD- and SAT-based; strong for finite-state systems including
  hardware.
- **P**: a state-machine language with a checker, designed for practitioners writing distributed
  systems, and used in production at Amazon.
- **Software model checkers** (CBMC, Java Pathfinder, Kani): check the actual code rather than a
  model, usually bounded. They close the model/code gap and pay for it in scale.

The pragmatic choice for this course: **TLA+ for behavioural and protocol properties, Alloy for
structural ones, a bounded software checker when you need to check real code.**

## 3. Construction: build a model checker, then use one

Build in `mpse/fm751/l04/`. Building a small checker before using a real one makes the real one's
behaviour — and its limits — legible.

**Stage 1 — an explicit-state safety checker.** Implement it: a model as `(init_states,
next_states(s), property(s))`, BFS with a visited set, and on violation a printed path from an
initial state to the bad one. Test it on the dining philosophers and on a two-process mutual
exclusion protocol with a deliberate bug.

**Stage 2 — measure the explosion.** Model a simple protocol with a parameterisable number of
processes and message-buffer depth. Plot reachable states against each parameter. Fit the curve.
Then find, on your machine, the configuration at which the check stops completing in a minute, in an
hour. Those two numbers are your practical budget and are worth knowing.

**Stage 3 — symmetry reduction.** Add it to your checker: canonicalise states by sorting the
interchangeable components. Re-run Stage 2's measurement and plot the reduction factor against
process count. It should be roughly *n!*, and seeing that empirically explains why symmetry is the
first reduction to reach for.

**Stage 4 — liveness.** Extend the checker to find lassos: search for a reachable cycle containing
no state satisfying the goal. Test it on a protocol with a starvation bug. Then add weak fairness as
a constraint on which cycles count, and show the same protocol now passing — or not.

**Stage 5 — read counterexamples.** Take five buggy protocol models (write them, or use classic ones
— Peterson's algorithm with a reordered assignment, a two-phase commit missing a durability step, a
naive leader election). For each, run your checker and follow §2.4's five-step procedure in writing.
The written analysis, not the counterexample, is the deliverable.

**Stage 6 — the model that is wrong.** Deliberately build a model that is *too permissive* — allow
an interleaving the real system cannot produce. Get a counterexample. Then identify it as a modelling
artifact, and state the assumption you had failed to encode. Write it into the model as an explicit
assumption. This is §2.4 step 4, and it is the step that turns a frustrating false positive into a
discovered unstated assumption.

**Stage 7 — bounded model checking.** Implement a bounded checker that unrolls the transition
relation *k* steps and encodes the question as an SMT query (previewing L09). Compare against your
explicit-state checker on the same models: time to find a shallow bug, and behaviour when there is
no bug. Report where each wins.

**Stage 8 — a real tool.** Install TLC and Alloy. Re-model two of Stage 5's protocols in each.
Compare: expressiveness for this problem, the time to a first result, the readability of the
counterexample, and the effort to reach a useful configuration size. Write 500 words on which you
would reach for and when.

**Stage 9 — abstraction.** Take a model too large to check. Abstract it — replace a data domain with
a small one, collapse states, or over-approximate a component — until it checks. Verify the property
on the abstraction. Then answer carefully: **what does this establish about the concrete system?**
Identify one spurious counterexample the abstraction produces, and refine to eliminate it.

## 4. Failure modes

- **Modelling too much detail.** The state space explodes and the counterexamples become unreadable.
- **Modelling the implementation instead of the design.** Same problem, plus it verifies nothing
  interesting.
- **Reporting "verified" without the bounds.** Overclaiming; and it discredits the technique when
  the bug appears at four nodes.
- **Ignoring a counterexample as unrealistic.** Sometimes correct, and then the assumption must be
  written into the model. Dismissing it without doing that means you will meet it in production.
- **Checking only safety.** Liveness bugs are real and are exactly the ones testing cannot find.
- **Liveness without fairness.** Everything fails trivially.
- **Vacuous properties** (L03 §2.7) that pass because their antecedent is unreachable.
- **Believing the model checks the code.** It does not (§2.5).
- **Giving up at the first state explosion** instead of abstracting.
- **Not making the counterexample a regression test.** The bug returns.

## 5. Exercises

### Warm-up (30 min)

1. Give the safety checking algorithm and explain why BFS rather than DFS.
2. Explain state space explosion quantitatively for a system with n processes of k states each and a
   channel of depth d over an alphabet of size a.
3. State the small scope hypothesis and say precisely what it justifies and what it does not.

### Core (3.5 h)

4. Complete Stages 1–3: the checker, the explosion measurement with your two time budgets, and
   symmetry reduction with the measured reduction factor.
5. Complete Stage 5 — five counterexamples with the written five-step analysis for each.
6. Complete Stage 6 and state the assumption you had failed to encode.
7. Complete Stage 8 and deliver the TLC-versus-Alloy comparison.

### Challenge

8. Complete Stages 4, 7 and 9: liveness with lasso detection and fairness, bounded model checking
   compared against explicit-state, and the abstraction with its spurious counterexample refined
   away.
9. Take a **real bug from your own systems** — one from your DS-701 chaos experiments, or a
   production incident you know the details of — and try to reproduce it by model checking. Build the
   smallest model that could exhibit it, state the properties, and see whether the checker finds it.
   Then write the honest report: did the checker find it? If not, why — was the bug outside the
   model's abstraction, in the implementation rather than the design, or dependent on an assumption
   you did not model? **The cases where model checking would not have caught a real bug are as
   instructive as the cases where it would**, and collecting a few of each is what calibrates your
   judgement about when to reach for it — which is this course's tenth learning outcome.

## 6. Self-check

1. Give the safety checking algorithm and say what the exhaustiveness buys over testing.
2. How is liveness checked, and what shape is the counterexample?
3. State the state space explosion problem quantitatively.
4. Give six mitigations and say which requires judgement rather than tooling.
5. State the small scope hypothesis and its four practical consequences.
6. Give the five-step procedure for reading a safety counterexample.
7. What are the two possible causes of a counterexample, and why is each useful?
8. Give five things model checking does not establish.
9. State precisely what it does establish, in the form you would use with a colleague.
10. Compare TLC, Alloy and a software model checker on what each is for.

## 7. Primary sources

- **Clarke, Grumberg, Kroening, Peled & Veith, *Model Checking*, 2nd ed. (2018)** — the reference.
- **Baier & Katoen, *Principles of Model Checking* (2008)** — the best textbook; chapters 3–6 cover
  this lesson properly.
- **Newcombe et al., "How Amazon Web Services Uses Formal Methods" (CACM 2015)** — read the
  descriptions of the bugs found; every one is a counterexample nobody would have imagined.
- Jackson, *Software Abstractions*, chapter 5 — the small scope hypothesis, stated and defended.
- Clarke, Emerson & Sifakis's Turing Award lecture, "Model Checking: Algorithmic Verification and
  Debugging" (CACM 2009) — the history and the ideas, from the inventors.
- Holzmann, *The SPIN Model Checker* — partial order reduction and explicit-state techniques.
- Biere et al., "Bounded Model Checking" (Advances in Computers, 2003).
- Clarke, Grumberg, Jha, Lu & Veith, "Counterexample-Guided Abstraction Refinement" (CAV 2000).
- Desai et al., "P: Safe Asynchronous Event-Driven Programming" (PLDI 2013) — model checking aimed
  at working distributed systems engineers.

---

**Previous:** [L03](L03-temporal-logic.md) · **Next:** [L05 — TLA+ and PlusCal](L05-tla-plus.md)
