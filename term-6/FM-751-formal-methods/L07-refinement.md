# FM-751 · Lesson 07 — Refinement

**Estimated study time:** 4.5 hours
**Prerequisites:** L05, L06; DS-701 L04

---

## 1. Orientation

L06 verified a protocol against a list of properties. That is useful and it has two weaknesses: the
list might be incomplete — you check what you thought to state — and it says nothing about the
relationship between the specification and the code.

**Refinement** addresses both:

> A low-level specification **refines** a high-level one when every behaviour the low-level system
> can produce is permitted by the high-level one. Written in TLA+ as an implication:
> `Implementation ⇒ Specification`.

The consequences are what make this the intellectual centre of the course:

- **Every property of the high-level specification automatically holds of the low-level one.** One
  theorem instead of a list, and no property is missed because you did not think of it.
- **It composes.** A chain of refinements — abstract specification, then design, then detailed
  design, then implementation — is verified stepwise, each step small enough to check.
- **It gives a precise meaning to "this implements that"**, which is otherwise a phrase people use
  without agreeing on what it means.

And this is why TLA+ makes the design decisions it does. **Stuttering invariance** (L03 §2.6) exists
precisely so refinement works: an implementation takes many small steps where the specification takes
one, and permitting stuttering is what makes the correspondence expressible at all.

The same idea appears throughout the programme under other names: DS-701 L04's linearizability is
refinement of a sequential specification by a concurrent implementation; CS-641's type soundness
relates a typed language to its semantics; DI-721 L05's serializability says a concurrent schedule
refines some serial one. Seeing them as one idea is worth the lesson on its own.

## 2. Theory

### 2.1 Refinement as implication

In TLA+ both specifications are formulas over behaviours. `Impl ⇒ Spec` says: every behaviour
satisfying `Impl` also satisfies `Spec`. That is refinement, and it is just logical implication —
no separate framework, no proof obligation generator, nothing new.

Two things must be true for the statement even to typecheck:

1. **The variables must correspond.** `Impl` typically has more variables, and different ones. The
   correspondence is supplied by a **refinement mapping** (§2.2).
2. **Stuttering must be allowed**, or a specification taking one step where the implementation takes
   twenty could never be refined. `[][Next]_vars` allows steps that leave `vars` unchanged, which is
   exactly what the implementation's internal steps look like *from the specification's point of
   view*.

The practical form in TLA+, using instantiation:

```tla
\* In the implementation module:
Abstract == INSTANCE HighLevelSpec WITH x <- f(implVars), y <- g(implVars)

THEOREM Spec => Abstract!Spec
```

You define, for each of the abstract specification's variables, an expression over the
implementation's variables. TLC checks the theorem by treating `Abstract!Spec` as a property.

### 2.2 The refinement mapping

The heart of the technique, and the place where the thinking happens.

> A **refinement mapping** is a function from the implementation's state to the specification's
> state — an *abstraction function*. It answers: given this concrete state, what abstract state does
> it represent?

Examples that show the shape:

- A replicated log implementing an abstract single register: the abstract value is the last
  *committed* entry in the log. (Note "committed" — an uncommitted entry does not yet represent
  anything abstractly, and getting that right is the whole difficulty.)
- A B-tree implementing an abstract map: the abstract map is the set of key-value pairs in the
  leaves.
- A cache plus a backing store implementing an abstract store: the abstract value is the cache's
  value if present, else the store's — and if that is not what your cache does, you have found a
  bug.
- A saga engine implementing an abstract atomic transaction: the abstract state is "committed" once
  the pivot is passed, "aborted" once compensation completes, and — importantly — the mapping must
  say what the abstract state is *during* the saga, which is the question DS-701 L08 §2.6 was really
  about.

Writing the mapping is where you find out whether you understand your own design. **The most common
outcome of a first attempt is discovering that no mapping exists** — the implementation can reach
states that correspond to no abstract state, which means it can do something the specification
forbids. That is a bug, found by an activity that felt like documentation.

### 2.3 When a mapping does not exist

Sometimes the refinement is genuinely true but no mapping from the concrete state alone can express
it — the abstract state depends on information the implementation does not currently hold. Two
standard remedies:

**History variables.** Add auxiliary state recording what has happened. Adding a history variable
does not change the implementation's behaviour (it is write-only, never read by the protocol), so
it is sound. Use it when the abstract state depends on the past — "the value that was committed
first", "the set of operations acknowledged".

**Prophecy variables.** Add auxiliary state recording a *non-deterministic choice made in advance*.
Use it when the abstract state depends on the future — the classic case being an implementation
that resolves a choice later than the specification does. A concurrent queue whose linearization
point depends on which of two concurrent operations wins a race needs one.

Abadi and Lamport's theorem is the reassuring result: **with history and prophecy variables,
refinement mappings are complete** — if the implementation really does refine the specification,
some mapping exists. So a failure to find one is either a bug or a failure of imagination, and the
theorem tells you which possibilities to consider.

Prophecy variables are conceptually uncomfortable ("the implementation guesses the future") and
sound, because the guess is constrained by the requirement that the behaviour remain consistent —
wrong guesses simply produce behaviours the specification already permits.

### 2.4 The layered approach

The intended methodology, and the one that makes large systems tractable:

```
Abstract specification    "there is one value; reads return the last write"
        ↑ refines
High-level design         "a leader holds the value and replicates to followers"
        ↑ refines
Detailed design           "Raft with terms, logs, and commit indices"
        ↑ refines
Implementation            code
```

Each arrow is checked separately. Each step is small enough that its mapping is comprehensible and
its check is feasible. Refinement is transitive, so the composition establishes that the code
satisfies the abstract specification.

In practice the last arrow — design to code — is the one that is not mechanically checked, and §2.6
is about what to do instead. The first three arrows are entirely feasible and are where most of the
value is.

### 2.5 Linearizability as refinement

Worth stating explicitly, because it unifies DS-701 L04 with this course:

> A concurrent object is **linearizable** with respect to a sequential specification if every
> concurrent history is equivalent to some sequential history that respects the real-time ordering
> of non-overlapping operations.

In refinement terms: **the concurrent implementation refines the abstract sequential specification**,
where the refinement mapping maps each concurrent state to the sequential state at its linearization
point.

Which explains several things at once: why linearizability composes (refinement composes, and
composition of refinements is refinement); why sequential consistency does not (it is not a
refinement of a specification with real-time ordering); and why finding the linearization point of a
lock-free algorithm is the hard part of proving it correct (it is the refinement mapping, and for
some algorithms it needs a prophecy variable).

Checking linearizability in TLA+ is therefore just checking a refinement, at small scope — three
clients, two keys, two values. Expensive, and exactly the property you claim.

### 2.6 Closing the gap to code

Refinement relates two specifications. Relating a specification to code is the remaining gap, and
the honest options, in decreasing order of rigour:

1. **Verify the code directly.** Software model checkers (CBMC, Kani, Java Pathfinder) or a
   verification-aware language (Dafny, Verus, Frama-C). Real, and expensive; correct for a memory
   allocator or a kernel.
2. **Generate the code from the specification.** Removes the gap by construction, and is only
   available for restricted domains.
3. **Derive tests from the specification** (L08). Use the specification as a model-based test
   oracle: run the implementation and the model on the same operation sequences and compare. **This
   is the best value for effort in ordinary engineering** and it is the technique to reach for.
4. **Trace validation.** Instrument the implementation to emit a trace of its state at each step,
   and check that trace against the specification — TLC can be driven to validate whether a real
   trace is a behaviour the specification permits. Elastic and others have used this in production,
   and it catches divergences that tests miss because it uses real executions.
5. **Structural correspondence by discipline.** Name the code's functions after the specification's
   actions, keep the structure parallel, and review the correspondence. Weak, cheap, and much better
   than nothing — and it makes options 3 and 4 easier later.

The professional statement to be able to make: **"the design is model-checked; the implementation is
tested against the model with derived properties; here is where the correspondence is weakest."**
That last clause is the honest part, and it is what distinguishes an engineer who understands the
technique from one who is selling it.

## 3. Construction: refine

Build in `mpse/fm751/l07/`, on the L06 protocol specification.

**Stage 1 — the abstract specification.** Write the simplest possible specification of what your
system provides: a single variable, atomic read and write, no nodes, no messages. Ten lines. Check
that it satisfies the properties you care about — trivially, which is the point: **the abstract
specification is where the properties are obviously true, and refinement transports them
downward.**

**Stage 2 — the mapping, attempted.** Write the refinement mapping from your L06 Raft specification
to the Stage 1 abstract one. Then check `Spec => Abstract!Spec` with TLC. Expect it to fail on the
first attempt. Read the counterexample: it is showing you a concrete state that maps to an abstract
state the abstract specification did not permit.

**Stage 3 — fix it.** The failure is one of three things: the mapping is wrong (most likely), the
abstract specification is too strong, or the implementation has a bug. Diagnose which. Then fix and
re-check. Write up which it was, because the diagnosis is the skill.

**Stage 4 — the intermediate layer.** Write a middle specification between the two: a leader holds
the value and replicates it, with no terms, no elections, no logs. Then check both arrows:
`Raft => Middle` and `Middle => Abstract`. Compare the effort and the check time against the single
direct refinement. Report which was easier and why.

**Stage 5 — a history variable.** Construct a case where the mapping needs one: an abstract property
about "the first value committed", or "the set of acknowledged operations". Add the history
variable, verify it does not constrain the protocol (it is written but never read by any guard), and
complete the mapping.

**Stage 6 — a prophecy variable.** Take a concurrent data structure whose linearization point is
determined by a later event — a lock-free stack or a queue with a helping mechanism. Show that no
mapping from the current state alone works. Add a prophecy variable and complete the mapping. This
stage is difficult and is the most conceptually interesting exercise in the course.

**Stage 7 — linearizability as refinement.** Specify an abstract sequential key-value store. Then
specify a concurrent implementation with overlapping operations. Check refinement at very small
scope. Then break it — allow a read to return a value from before a completed write — and confirm
the check fails, and read the counterexample as a linearizability violation.

**Stage 8 — trace validation.** Instrument your DS-701 implementation to emit a state trace: after
each protocol step, the values of the variables that appear in your specification. Then write a
checker that validates a recorded trace against the specification — each consecutive pair of states
must satisfy `Next` (or be a stuttering step). Run it against traces from your chaos experiments.
Report every divergence. **Divergences are either implementation bugs or specification
inaccuracies, and both are findings**; expect several, and expect most to be the second kind, which
is itself informative about how well your specification describes your code.

**Stage 9 — the correspondence document.** Write, for your system: the abstract specification, the
refinement chain, the mappings, what is mechanically checked and what is not, and — the required
section — **where the correspondence between specification and code is weakest, and what could
diverge there without being detected.** This is the term artifact's refinement deliverable, and the
last section is the one that makes it honest.

## 4. Failure modes

- **No abstract specification.** Checking a long list of properties and hoping it is complete.
- **A mapping that does not exist, treated as a tooling problem.** It is usually a bug.
- **A history variable that the protocol reads.** It now constrains behaviour and the refinement is
  unsound.
- **Skipping intermediate layers.** One enormous refinement step, incomprehensible when it fails.
- **Checking refinement at too large a scope.** It is expensive; small scope, honestly reported.
- **Claiming the code is verified because the design is.** The largest overclaim available in this
  field.
- **Trace validation without recording enough state.** The specification's variables must all be
  observable, which may require adding instrumentation the implementation did not need.
- **Ignoring trace divergences as "just a modelling difference".** Sometimes true, and it must be
  investigated and written down each time.
- **Using `○` (next) anywhere.** Breaks stuttering invariance and therefore refinement.

## 5. Exercises

### Warm-up (30 min)

1. Define refinement as implication and state the two things that must hold for it to typecheck.
2. Explain why stuttering invariance is necessary for refinement, with an example.
3. Explain what a refinement mapping is and what its non-existence usually means.

### Core (3.5 h)

4. Complete Stages 1–3: the abstract specification, the failing mapping, and the diagnosis of which
   of the three causes it was.
5. Complete Stage 4 and compare the layered against the direct refinement.
6. Complete Stage 5 with the history variable and the verification that it does not constrain.
7. Complete Stage 9 — the correspondence document with its weakest-link section.

### Challenge

8. Complete Stages 6–7: the prophecy variable for a lock-free structure, and linearizability checked
   as refinement with a deliberate violation.
9. Complete Stage 8, then extend trace validation into a **continuous conformance check**: run your
   implementation under load and fault injection, stream its state trace to the validator, and
   report divergences in real time. Then run it for an extended period across your full chaos suite
   (DS-701 L10) and produce a report: how many divergences, of what kinds, how many were
   implementation bugs versus specification inaccuracies, and what the instrumentation cost in
   performance. Finally, assess the technique honestly: what class of bug does trace validation
   catch that property-based testing does not, and what does it miss that a model check catches?
   Answering that precisely is the whole point of this course's final third.

## 6. Self-check

1. Define refinement and state its two consequences.
2. Why is stuttering invariance a precondition for refinement?
3. What is a refinement mapping, and what does a failure to find one usually indicate?
4. When do you need a history variable, and what makes adding one sound?
5. When do you need a prophecy variable, and why is it not circular?
6. State the Abadi–Lamport completeness result and what it tells you when a mapping eludes you.
7. Why is a layered refinement chain easier than one big step?
8. Express linearizability as a refinement, and explain why it composes and sequential consistency
   does not.
9. Give the five options for closing the specification-to-code gap, ranked.
10. State the honest professional claim about a model-checked design with a tested implementation.

## 7. Primary sources

- **Abadi & Lamport, "The Existence of Refinement Mappings" (TCS 1991)** — history and prophecy
  variables and the completeness theorem.
- **Lamport, *Specifying Systems*, chapters 5 and 10–11** — refinement and instantiation in TLA+.
- **Herlihy & Wing, "Linearizability: A Correctness Condition for Concurrent Objects" (TOPLAS 1990)**
  — read §2.5's claim against the original.
- Lamport & Merz, "Auxiliary Variables in TLA+" (2017) — the modern treatment.
- Abrial, *Modeling in Event-B* — refinement as the central methodology, in a different formalism;
  worth seeing for contrast.
- Davis, Beckett et al. and the Elastic and MongoDB write-ups on **trace validation** of TLA+
  specifications against production systems — the practical form of Stage 8, and the most
  interesting recent development in applying this material.
- Hawblitzel et al., "IronFleet: Proving Practical Distributed Systems Correct" (SOSP 2015) — a full
  refinement chain from specification to executable code; read it for what that costs.
- DS-701 L04, re-read with refinement available. Linearizability's definition should now look like
  what it is.

---

**Previous:** [L06](L06-specifying-a-protocol.md) · **Next:**
[L08 — Property-Based Testing as Lightweight Verification](L08-property-based-testing.md)
