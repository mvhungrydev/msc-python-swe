# FM-751 · Lesson 01 — Specifications as Artifacts

**Estimated study time:** 4 hours
**Prerequisites:** CS-641 L02 (operational semantics), SE-521 L10 (ADRs)

---

## 1. Orientation

Before any tool, a question: **what is a specification for?**

The naive answer — "to tell the implementer what to build" — is the least of it, and it is why most
specifications in industry are bad. The valuable answers:

1. **To find out what you actually mean.** Writing a design precisely enough to be checked forces
   you to decide things you had left vague, and the act of deciding them is where most of the value
   is. Practitioners at Amazon and elsewhere report this consistently: **a large share of the bugs
   found by formal specification are found while writing the specification, before any tool runs.**
2. **To make disagreement visible.** Two engineers with the same prose description can hold
   incompatible mental models for months. A formal statement makes the incompatibility immediate.
3. **To enable mechanical checking.** Only a precise specification can be checked by a machine.
4. **To constitute a contract.** A specification says what a component guarantees and what it
   assumes, which is the basis of modular reasoning (SE-521).
5. **To survive the team.** Two years later, the specification says what the system was supposed to
   do, which the code cannot — the code says what it does.

The claim this lesson makes:

> **A specification is what you can check an implementation against.** If nothing could contradict
> it, it is not a specification — it is documentation. And the discipline of writing something that
> *could* be contradicted is what makes the exercise worth its cost.

## 2. Theory

### 2.1 The spectrum of precision

Specifications are not binary. From weakest to strongest:

1. **Prose.** "The system should handle concurrent updates correctly." Cheap, universally
   understood, and it means nothing checkable — "correctly" is exactly the word that needed
   defining.
2. **Structured prose.** RFC 2119 keywords (MUST, SHOULD, MAY), numbered requirements, defined
   terms. A real improvement: requirements become individually referenceable and testable.
3. **Examples and scenarios.** Given/when/then, acceptance tests, worked examples. Concrete,
   checkable, and incomplete by construction — examples cannot cover a state space.
4. **Types.** A signature is a machine-checked partial specification, checked continuously, at zero
   marginal cost (CS-641 L03). The most cost-effective formal method in existence, and the one most
   engineers already use without calling it one.
5. **Contracts.** Preconditions, postconditions and invariants, checkable at runtime (L10) or
   statically.
6. **Properties.** Universally quantified statements checked over generated inputs (L08). "For all
   lists, `sort` returns a permutation in non-decreasing order."
7. **Formal models.** A mathematical description of behaviour — a state machine, a set of allowed
   traces — that a model checker or prover can reason about exhaustively (L03–L07).

The engineering judgement is **choosing the right rung for the risk**, and it varies within one
system: types everywhere, contracts on the tricky modules, properties on the data structures, and a
formal model for the one protocol whose failure would be catastrophic.

The trap at the bottom of the ladder is worth naming precisely: **prose specifications hide
ambiguity rather than resolving it**, and everyone leaves the meeting believing they agreed.

### 2.2 What a specification says

A useful specification separates three things, and conflating them is the most common structural
error:

- **Assumptions** — what the environment guarantees. "At most one leader per term." "Messages may
  be lost, duplicated and reordered, but not corrupted." (DS-701 L01's failure model *is* an
  assumptions section.)
- **Guarantees** — what the system provides *given* the assumptions.
- **The interface** — what is observable, in terms of which the guarantees are stated.

**A guarantee without its assumptions is meaningless**, and this is not pedantry: every distributed
systems result in DS-701 is a guarantee bounded by assumptions, and every real incident where a
system "violated its guarantees" is an incident where reality left the assumption set.

The other essential discipline: **specify what, not how.** A specification that describes the
algorithm has ruled out every other implementation and cannot serve as a standard against which an
implementation is checked. "The returned list is a permutation of the input in non-decreasing order"
admits any sorting algorithm; "iterate, comparing adjacent pairs" admits one.

### 2.3 Behavioural specification: states and traces

The model underlying everything from L03 onward:

- A **state** is an assignment of values to variables.
- A **behaviour** (or trace) is an infinite sequence of states.
- A **specification** is a predicate on behaviours — the set of behaviours it permits.
- An implementation **satisfies** the specification if every behaviour it can produce is permitted.

Two consequences worth internalising early:

**A specification is a set of allowed behaviours, so "more specification" means "fewer allowed
behaviours".** A specification permitting everything says nothing; one permitting nothing is
unimplementable. Real specifications sit between, and a common novice error is writing one so tight
that it forbids the implementation you intended.

**Non-determinism is expressed by permitting several behaviours.** This is how you specify "the
system may respond in any order" or "the message may or may not arrive" — you do not *implement*
non-determinism, you *allow* it, and the implementation may resolve it however it likes.

In TLA+'s formulation, which L05 develops: a specification is written as `Init ∧ □[Next]_vars ∧
Fairness` — an initial state predicate, a next-state relation that every step must satisfy (or leave
the variables unchanged), and fairness conditions that rule out the implementation doing nothing
forever.

### 2.4 Safety and liveness

The fundamental classification (L03 develops it formally):

- **Safety**: "something bad never happens." Violated by a *finite* prefix — you can point at the
  exact moment. Mutual exclusion, no lost writes, no two leaders in a term, the balance never goes
  negative.
- **Liveness**: "something good eventually happens." Violated only by an *infinite* behaviour —
  there is no moment at which it has failed, only an eternity in which it never succeeded.
  Termination, eventual delivery, every request eventually answered.

Why this matters immediately: **safety and liveness require different reasoning techniques,
different specification constructs and different verification approaches**, and most engineers
conflate them. "The system is available" is liveness; "the system does not lose data" is safety.
DS-701's entire structure is a series of results about which of the two you must give up (CAP is
"you cannot have this safety property and this liveness property under this assumption").

Every property is the conjunction of a safety property and a liveness property (Alpern and
Schneider), which is a nice theorem and a useful decomposition habit: when a requirement is
unclear, split it.

### 2.5 What makes a specification good

- **Checkable.** Something could contradict it.
- **Abstract.** It says what, not how. It should admit implementations you have not thought of.
- **Complete about what matters** and silent about what does not. A specification that fixes
  irrelevant details over-constrains; one that omits a needed guarantee is useless.
- **Assumptions explicit.** Every one of them.
- **Small.** A specification that is longer than the implementation will not be maintained and
  probably will not be read. Specify the part where the difficulty is.
- **Executable or checkable mechanically**, if the risk justifies the cost.

The realistic scope for a working engineer: **specify the hard part.** Not the whole system — the
consensus protocol, the cache invalidation scheme, the permission model, the state machine that
everyone argues about. Newcombe's account of the AWS experience makes exactly this point: the
specifications were of the tricky components, not of the services.

### 2.6 The objections, answered honestly

**"It takes too long."** Sometimes true. Newcombe reports two to four weeks for a substantial
specification by an engineer who already knew the technique, and days for smaller ones. Against a
consensus protocol's development cost, that is small; against a CRUD endpoint's, it is absurd.
Judge per component.

**"The spec won't match the code."** True, and it is why L07 (refinement) and L08 (property-based
testing derived from the specification) exist. The gap is real and it is manageable, and the honest
position is that a checked design plus tested code is better evidence than tested code alone, not
that the gap disappears.

**"We'd have to learn the maths."** Less than people fear. TLA+ needs set theory, logic and
functions — no more than a discrete mathematics course, and CS-621 and CS-641 have already supplied
it.

**"Our system changes too fast."** A specification of a *stable core* — the protocol, the invariant,
the state machine — outlives the code that implements it, and is usually the part that changes least.

**"It only finds bugs in the model."** Correct, and the bugs it finds in the model are the ones
that would otherwise have been found in production, at a much higher price. It also finds bugs in
your *understanding*, which is where they started.

The genuine limitation, stated plainly: **verification is relative to the model and its
assumptions.** A specification that assumes messages are not corrupted says nothing about a system
where they are. Being precise about what has and has not been established is a professional
obligation, and overclaiming is the fastest way to discredit the technique.

## 3. Construction: write specifications and find the ambiguity

Build in `mpse/fm751/l01/`. This lesson's construction is deliberately notation-free: the skill is
precision, and the tools come later.

**Stage 1 — the ambiguity hunt.** Take a prose specification from real life: an RFC, an API
document, your own team's design doc, or the requirements for a feature you have built. Find five
ambiguities — places where two reasonable engineers would implement differently. For each: state
both readings, and construct a scenario in which they differ observably.

**Stage 2 — climb the ladder.** Take one requirement from Stage 1 and express it at every rung of
§2.1: prose, structured prose, examples, a type signature, a contract, a property, and a state
machine. Note at each rung what became decided that was previously vague. This exercise is the
lesson in miniature.

**Stage 3 — assumptions and guarantees.** For a component you have built in this programme — your
DS-701 replicated store, your DI-721 storage engine, your CA-731 platform interface — write the
assumption/guarantee/interface document. Then find the guarantee whose assumptions you had not
written down, because there will be one.

**Stage 4 — safety or liveness.** Take fifteen requirements from real systems (mix your own with
some from RFCs and SLAs) and classify each as safety, liveness, or a conjunction requiring
decomposition. For the ambiguous ones, explain what makes them ambiguous. Then check: does your
system's testing actually address the liveness ones? Usually it does not, and noticing that is the
point.

**Stage 5 — the state machine.** Specify a non-trivial component as an explicit state machine:
states, initial state, transitions with guards, and invariants. A saga engine (DS-701 L08), a
circuit breaker (DS-701 L09), or a TCP-like connection protocol are good choices. Then find a state
your implementation can reach that your state machine does not describe — or prove to yourself that
none exists.

**Stage 6 — the trace view.** For your Stage 5 machine, write out (by hand) five behaviours as
sequences of states: a normal one, two error paths, and two interleavings you had not considered.
Then write one behaviour that your specification permits but that you do *not* want, and tighten the
specification to exclude it. Then check that your tightening did not exclude something you do want.

**Stage 7 — specify by example, then generalise.** Take an operation and write ten example-based
tests. Then write the general property they were instances of. Then find a case the property covers
and the examples did not — and check whether your implementation handles it. This is the bridge to
L08, and it usually finds something.

**Stage 8 — the review.** Give your Stage 3 assumption/guarantee document to a colleague with one
instruction: *find a case where the guarantee does not hold.* Record what they find. Then repeat
with the instruction: *find an assumption I did not state.* The second question is more productive
than the first, in most cases.

## 4. Failure modes

- **Prose that hides ambiguity.** Everyone agrees; nobody agrees.
- **Guarantees without assumptions.** Meaningless, and the source of every "it violated its
  guarantees" incident.
- **Specifying how rather than what.** Rules out every implementation but the one you thought of.
- **Over-constraining.** Fixing irrelevant details forbids valid implementations, including the
  faster one you will want later.
- **Under-specifying the hard part** while over-specifying the easy part. The usual distribution
  of effort, and backwards.
- **A specification longer than the implementation.** Will not be maintained.
- **Confusing safety and liveness.** They need different techniques, and liveness is the one nobody
  tests.
- **Specifying everything.** Unaffordable, so it does not happen; specify the difficult component.
- **A specification that nothing could contradict.** Documentation wearing a specification's
  clothes.
- **Never re-reading it.** A specification that diverges from the system silently is worse than none,
  because people trust it.

## 5. Exercises

### Warm-up (30 min)

1. Give five purposes of a specification and say which is most often the largest source of value.
2. Give the seven rungs of the precision ladder with an example of each.
3. Define safety and liveness precisely in terms of how each is violated, and classify five
   requirements.

### Core (3 h)

4. Complete Stages 1–2: five ambiguities with observable-difference scenarios, and one requirement
   expressed at all seven rungs with what each rung decided.
5. Complete Stage 3 and identify the guarantee whose assumptions you had not written.
6. Complete Stages 5–6: the state machine, five hand-written behaviours, and the
   tighten-then-check-you-did-not-over-tighten exercise.
7. Complete Stage 8 and report what the two review questions produced.

### Challenge

8. Complete Stages 4 and 7, then take a **published protocol specification** — an RFC, a consensus
   protocol paper, or a well-known API — and write a precision review: what is stated formally, what
   is stated in prose that should be formal, what assumptions are implicit, and what two
   implementations could differ on while both conforming. Then check whether any known
   interoperability problems with that protocol correspond to the ambiguities you found. For widely
   deployed protocols they often do, and the correspondence is the best available evidence that this
   exercise is worth doing.
9. Write a **specification-first design** for a component you have not built yet: assumptions,
   guarantees, interface, state machine, invariants, and both safety and liveness properties —
   before any code. Then implement it, and record every place where the implementation forced you to
   change the specification. Classify each change: was the specification wrong, incomplete, or
   over-constrained? That classification is the most instructive output, because it tells you which
   kind of specification error you personally tend to make.

## 6. Self-check

1. Give five purposes of a specification and the claim about what makes something a specification
   at all.
2. Give the seven rungs of the precision ladder and say which is the most cost-effective formal
   method in existence.
3. What three things must a specification separate, and why is a guarantee without assumptions
   meaningless?
4. Why specify what rather than how?
5. Define state, behaviour, specification and satisfaction in the trace model.
6. Why does "more specification" mean "fewer allowed behaviours", and what is the novice error that
   follows?
7. How is non-determinism expressed in this model?
8. Define safety and liveness by how each is violated, and give two examples of each.
9. Give six properties of a good specification.
10. Give the five common objections and the honest answer to each, including the one that is a
    genuine limitation.

## 7. Primary sources

- **Lamport, *Specifying Systems* (2002)**, Part I — free from the author; chapters 1–3 are this
  lesson's material done properly.
- **Newcombe et al., "How Amazon Web Services Uses Formal Methods" (CACM 2015)** — read it now, and
  note how much of the reported value came from writing the specification rather than from checking
  it.
- Hoare, "An Axiomatic Basis for Computer Programming" (CACM 1969) — six pages; L02's foundation and
  worth reading before it.
- Alpern & Schneider, "Defining Liveness" (Information Processing Letters, 1985) — the safety /
  liveness decomposition theorem.
- Lamport, "Who Builds a House Without Drawing Blueprints?" (CACM 2015) — the argument for
  specification, in two pages, aimed at practitioners.
- Jackson, *Software Abstractions*, chapters 1–2 — the case for lightweight modelling, from a
  different tradition.
- Zave, "Using Lightweight Modeling to Understand Chord" (SIGCOMM CCR 2012) — a published,
  widely-cited protocol shown to be incorrect. Read it as evidence for Stage 8.
- Meyer, *Object-Oriented Software Construction*, chapters on Design by Contract — the contracts rung
  of the ladder, argued at length.

---

**Next:** [L02 — Hoare Logic, Invariants, and Reasoning About Code](L02-hoare-logic-and-invariants.md)
