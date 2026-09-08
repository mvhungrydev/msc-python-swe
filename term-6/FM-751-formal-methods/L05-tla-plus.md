# FM-751 · Lesson 05 — TLA+ and PlusCal

**Estimated study time:** 5 hours
**Prerequisites:** L01–L04

---

## 1. Orientation

TLA+ is the notation this course uses, and the reason is not that it is the most elegant formalism —
it is that it is the one with the strongest record of finding real bugs in real distributed systems
designed by good engineers.

Its design decisions all follow from one commitment:

> **A specification is a mathematical formula describing a set of behaviours.** Not a program, not a
> model in a modelling language with its own semantics — a formula in a logic built on set theory
> and temporal logic.

That commitment gives TLA+ its characteristic properties: specifications at any level of
abstraction, in the same language; **refinement as implication** (L07), so "this design implements
that specification" is a theorem statement rather than a separate framework; and a very small
language, because ordinary mathematics is doing most of the work.

The practical cost is a learning curve concentrated in the first week: the notation looks alien, the
tooling is idiosyncratic, and the error messages are unhelpful. The learning curve flattens sharply
after that, and this lesson is designed to get you over it.

**PlusCal** is an algorithm language that compiles to TLA+, and it is the right on-ramp: it looks
like pseudocode, it makes concurrent algorithms much easier to express, and its output is TLA+ you
can read and eventually write directly.

## 2. Theory

### 2.1 The pieces of a specification

A TLA+ module is a set of definitions. The standard shape of a system specification:

```tla
---- MODULE Counter ----
EXTENDS Naturals

CONSTANTS MaxValue                    \* a parameter, fixed for a check
VARIABLES count, done                 \* the state

vars == << count, done >>             \* a tuple, used in the stuttering bracket

TypeOK == /\ count \in 0..MaxValue    \* an invariant that acts as a type declaration
          /\ done  \in BOOLEAN

Init == /\ count = 0                  \* the initial state predicate
        /\ done  = FALSE

Increment == /\ ~done                 \* an action: a predicate on (state, next state)
             /\ count < MaxValue
             /\ count' = count + 1
             /\ UNCHANGED done

Finish == /\ ~done
          /\ count = MaxValue
          /\ done' = TRUE
          /\ UNCHANGED count

Next == Increment \/ Finish           \* the next-state relation

Spec == Init /\ [][Next]_vars /\ WF_vars(Next)
====
```

The elements to internalise:

- **CONSTANTS** are parameters you instantiate when checking — the number of nodes, the set of
  values. Keeping them small is how you control the state space (L04 §2.2).
- **VARIABLES** are the state. Unprimed is the current state; **primed is the next state**.
- **An action** is a predicate relating current and next states. `count' = count + 1` is not an
  assignment; it is an assertion about the relationship between two states, which is why it can
  appear anywhere a formula can.
- **`UNCHANGED x`** abbreviates `x' = x`. **Forgetting it is the single most common beginner error**
  — an unconstrained primed variable can take *any* value, so the model checker explores states you
  never intended, and the resulting counterexample is baffling until you spot it.
- **`Next`** is a disjunction of actions: at each step, one of them happens.
- **`[][Next]_vars`** means every step satisfies `Next` *or leaves `vars` unchanged* — the stuttering
  allowance from L03 §2.6.
- **`WF_vars(Next)`** is weak fairness: the system cannot stall forever when a step is enabled.
- **`TypeOK`** is a convention, not a language feature: TLA+ is untyped, and `TypeOK` is an invariant
  you check first. **Always write it and always check it** — it catches more modelling errors than
  anything else, and a `TypeOK` violation usually means a missing `UNCHANGED`.

### 2.2 The mathematics

TLA+ is set theory, and the operators you need are few:

**Logic**: `/\` `\/` `~` `=>` `<=>`, and the quantifiers `\A x \in S : P` (for all) and
`\E x \in S : P` (there exists). Note the **aligned conjunction list** style used above — bullets of
`/\` at the same indentation are conjoined, and this is how real specifications are written because
it makes structure visible.

**Sets**: `\in` `\notin` `\cup` `\cap` `\subseteq` `{x \in S : P(x)}` (filter) `{f(x) : x \in S}`
(map) `SUBSET S` (powerset) `UNION S` `Cardinality(S)`.

**Functions**: `[x \in S |-> e]` defines a function; `f[x]` applies it; `DOMAIN f` is its domain;
`[f EXCEPT ![x] = e]` is `f` with one point changed — the workhorse for updating state, and worth
becoming fluent with. `[S -> T]` is the set of all functions from S to T.

**Records** are functions with string keys: `[name |-> "a", age |-> 3]`, accessed as `r.name`, and
`[S -> T]`-style sets of records are written `[name : STRING, age : Nat]`.

**Sequences** (with `EXTENDS Sequences`): `<<1, 2, 3>>`, `Len(s)`, `Head`, `Tail`, `Append`, `\o`
(concatenation), `SubSeq(s, m, n)`, `s[i]` (**1-indexed**, which trips up everyone at least once).

**`CHOOSE x \in S : P(x)`** picks *some* element satisfying P — deterministically but unspecified.
It is not non-determinism: `CHOOSE` always returns the same value for the same arguments. Modelling
"the system may pick any" requires `\E x \in S` in an action, not `CHOOSE`. Confusing these two is
the second most common beginner error.

### 2.3 PlusCal

PlusCal looks like pseudocode and compiles into TLA+ inside a comment block in the same file. It is
much easier for algorithms with control flow, and it makes the concurrency explicit:

```
(*--algorithm counter
variables count = 0, done = FALSE;

process worker \in 1..3
variables local = 0;
begin
  Loop:
    while count < MaxValue do
      Read:   local := count;
      Write:  count := local + 1;
    end while;
end process;
end algorithm; *)
```

The critical concept, and the whole reason PlusCal is useful for concurrency:

> **Labels define atomicity.** Everything between two labels executes as one atomic step. Where you
> place labels determines which interleavings the model checker explores.

In the example above, `Read` and `Write` are separate labels, so another process can run between
them — which is exactly the lost update of DI-721 L04, and the checker will find it. Combine them
into one label and the bug disappears, because you have asserted that the read-modify-write is
atomic. **Label placement is a modelling decision about what your implementation actually guarantees
atomically**, and getting it wrong is how a specification verifies something the code does not do.

PlusCal offers two idioms: `--algorithm` (procedural, with a program counter) and `--fair
algorithm` (adds weak fairness). Use `await` for blocking, `either ... or` for non-determinism, and
`with x \in S do` to choose an arbitrary element.

The workflow: write PlusCal, translate to TLA+ (the toolbox does this on save), and check the TLA+.
Read the generated TLA+ — that is how you learn to write it directly, and after a few specifications
you will start writing TLA+ for anything that is not primarily control flow.

### 2.4 Running TLC

TLC needs a **model**: constant values, what to check, and any constraints.

- **Constants**: instantiate them small. `Nodes = {n1, n2, n3}`, `Values = {v1, v2}`. This is the
  single biggest lever on check time.
- **Invariants**: state predicates checked at every reachable state. `TypeOK` first, then your
  safety properties.
- **Properties**: temporal formulas, checked over behaviours. Liveness lives here, and requires
  fairness in the spec.
- **Symmetry sets**: declare interchangeable constants and TLC applies symmetry reduction (L04
  §2.2) — often a factorial saving, for one line of configuration.
- **State constraints**: bound the exploration (`count < 10`) when the state space is otherwise
  infinite. Note the honesty obligation: a constraint means you have checked a *bounded* system, and
  it should be reported.
- **`ASSUME`** statements record assumptions on constants and are checked.

Practical operation:

- Check `TypeOK` alone first. If it fails, the model is malformed and everything else is noise.
- Then safety invariants, then liveness. Liveness checking is substantially more expensive; do it
  last and with the smallest configuration that is meaningful.
- Watch the **distinct states** count. If it is growing without bound, you need a constraint or a
  smaller model.
- **Check that your invariants are not vacuous.** TLC can verify `\A x \in {} : P(x)` all day. Add a
  deliberately false invariant temporarily and confirm the checker finds a counterexample — if it
  does not, your model is not reaching the states you think it is. This sanity check takes thirty
  seconds and catches a whole class of false confidence.

### 2.5 The modelling discipline

The skill is not the notation; it is deciding what to model. The rules:

**Model the design, not the implementation.** You are checking whether the *algorithm* is correct,
not whether your Python is. Data structures become sets and functions; a network becomes a set of
in-flight messages; a database becomes a function from keys to values.

**Abstract data ruthlessly.** Two values, not 2³². Three nodes, not a hundred. Use symmetry.

**Model the environment's misbehaviour explicitly.** This is where the bugs are. A message-passing
model should have actions for delivery, loss, duplication and reordering (DS-701 L01's failure
model, encoded), because the interesting interleavings involve them.

**Choose atomicity deliberately.** Each action is atomic. Is your real read-modify-write atomic? If
not, split it into separate actions or PlusCal labels — otherwise you have verified a system you did
not build.

**Start absurdly small and grow.** Two nodes, no failures, check `TypeOK`. Then one property. Then
failures. Then three nodes. Each step either passes quickly or hands you a counterexample about
something you just added, which is the fastest possible debugging loop.

**Write the properties before the actions**, or you will unconsciously build a model that satisfies
whatever you happen to write afterwards.

### 2.6 Common errors

The ones everyone makes, worth memorising to save the days they otherwise cost:

- **Missing `UNCHANGED`.** An unconstrained primed variable takes any value, producing bizarre
  counterexamples. If a counterexample makes no sense, check this first.
- **`CHOOSE` for non-determinism.** Use `\E x \in S` in the action, or PlusCal's `with`.
- **Sequences indexed from 1.** `s[0]` is undefined.
- **Deadlock reported as an error.** TLC treats a state with no successor as a deadlock. Often it is
  a real bug; sometimes it is a legitimate terminal state, and you either add a stuttering
  self-loop for termination or disable the deadlock check *and say that you did*.
- **An enormous state space from one unbounded variable.** Usually a counter or an unbounded
  sequence. Bound it with a constraint.
- **A specification that permits nothing.** `Next` with contradictory conjuncts means no steps are
  ever enabled, and everything passes vacuously. The deliberately-false-invariant check (§2.4)
  catches this.
- **Liveness checked with no fairness.** Fails immediately with a stuttering counterexample.
- **Modelling the happy path only.** No failures modelled means no failure bugs found, and the
  specification's value drops to nearly nothing.

## 3. Construction: learn the tool on real problems

Build in `mpse/fm751/l05/`. Install the TLA+ Toolbox or the VS Code extension plus the command-line
tools. Lamport's video course is the fastest way through the first hours and is recommended
alongside this.

**Stage 1 — the mechanics.** Write and check the counter specification from §2.1. Then break it four
ways deliberately: remove an `UNCHANGED`, make `Next` unsatisfiable, introduce an unbounded
variable, and state a vacuous invariant. Record TLC's output for each. **This stage is about
recognising the four failure signatures**, and it will save you days later.

**Stage 2 — a sequential algorithm.** Specify binary search (L02) in TLA+ and check it against the
postcondition, with the array as a sorted sequence over a small value set. Then introduce the
off-by-one from L02 Stage 3 and confirm TLC finds it. Compare the counterexample against the
invariant analysis you did in L02 — they should identify the same thing, arrived at differently.

**Stage 3 — concurrency and atomicity.** Write the PlusCal lost-update example from §2.3. Check it
and read the counterexample. Then combine the labels into one and show the bug disappearing. Then
add a lock and show it disappearing with separate labels. Write 300 words on what label placement
means about your implementation.

**Stage 4 — mutual exclusion.** Specify Peterson's algorithm in PlusCal. Check mutual exclusion
(safety) and starvation freedom (liveness, with fairness). Then break it — swap two statements — and
find the counterexample. Then determine which fairness assumption starvation freedom actually needs.

**Stage 5 — a message-passing model.** Build the reusable component you will need for L06: a network
as a set (or bag) of in-flight messages, with actions for send, deliver, drop, duplicate and
reorder. Verify that reordering and duplication are genuinely possible in your model by writing a
temporary invariant that they are not, and confirming TLC refutes it.

**Stage 6 — a real protocol.** Specify two-phase commit (DS-701 L08). Properties: all participants
decide the same thing (agreement); commit only if all voted yes (validity); and the liveness
property that everyone eventually decides. Then model the coordinator crashing and demonstrate the
blocking problem — the participant stuck in the in-doubt window — as a counterexample to liveness.
**Seeing DS-701's theoretical result appear as a concrete trace is the point of this stage.**

**Stage 7 — the state space.** For Stage 6's model, measure distinct states against the number of
participants. Apply symmetry reduction and re-measure. Then find the largest configuration you can
check in ten minutes, and write down that number as your working bound.

**Stage 8 — a design of your own.** Take a design decision you are currently uncertain about — from
your CA-731 platform, your DI-721 pipeline, or a real system at work — and specify it. Small: one
protocol, three properties. Then check it. Report what you found, including "nothing", and — more
importantly — **what writing the specification forced you to decide** that the prose design had left
vague (L01 §1). That second finding is usually the larger one.

**Stage 9 — read a real specification.** Take a published TLA+ specification — Raft, Paxos, the
DynamoDB or S3 specs where public, or one from the `tlaplus/Examples` repository — and read it end to
end. Then modify it: add a property of your own, or introduce a bug and confirm the checker catches
it. Reading good specifications is how you learn to write them, exactly as with code.

## 4. Failure modes

- **Missing `UNCHANGED`.** The first thing to check when a counterexample is incomprehensible.
- **`CHOOSE` where `\E` was meant.** Determinism where you wanted a choice.
- **A vacuous specification.** Everything passes because nothing is reachable. Run the
  deliberately-false-invariant check.
- **Modelling the implementation.** State space explodes; nothing interesting is verified.
- **Modelling only the happy path.** No failure actions means no failure bugs found.
- **Wrong atomicity.** Verifying a system you did not build.
- **Liveness without fairness.** Immediate stuttering counterexample.
- **Constants too large from the start.** Hours of checking before the first result; start with two.
- **A deadlock error dismissed without investigation.** Sometimes legitimate termination, often a
  real bug.
- **State constraints unreported.** You checked a bounded system; say so.
- **Never reading the generated TLA+ from PlusCal.** You stay dependent on PlusCal and never learn
  the language.

## 5. Exercises

### Warm-up (30 min)

1. Explain what `Spec == Init /\ [][Next]_vars /\ WF_vars(Next)` says, term by term.
2. Explain why `count' = count + 1` is not an assignment, and what `UNCHANGED` is for.
3. Explain what labels mean in PlusCal and what a label placement decision asserts about your
   implementation.

### Core (3.5 h)

4. Complete Stages 1–3: the four failure signatures recorded, binary search checked and broken, and
   the atomicity demonstration with its write-up.
5. Complete Stage 4: Peterson's algorithm with safety and liveness, broken and diagnosed, with the
   fairness requirement identified.
6. Complete Stage 5 and verify your network model genuinely permits reordering and duplication.
7. Complete Stage 6: two-phase commit with the blocking problem produced as a counterexample.

### Challenge

8. Complete Stages 7–9: the state space measurement with your working bound, your own design
   specified with both findings reported, and a real specification read and modified.
9. Specify **a protocol you use but did not design** — a piece of a real system's behaviour, from
   its documentation or RFC. Write the properties you believe it guarantees. Check them. Then do the
   part that makes this valuable: where the documentation was too vague to specify, record the
   ambiguity and resolve it by experiment against the real system, then encode the resolution as an
   `ASSUME`. Produce, as the deliverable, a specification plus a list of the ambiguities you had to
   resolve. Published protocols contain more of these than their authors would like, and the list
   is frequently the most useful artifact you can hand a team that depends on the protocol.

## 6. Self-check

1. Give the standard shape of a TLA+ specification and explain each conjunct.
2. What is an action, and why is a primed variable not an assignment?
3. What does `UNCHANGED` do and what happens without it?
4. What is `TypeOK` and why is it a convention rather than a feature?
5. Distinguish `CHOOSE` from `\E` and say which expresses non-determinism.
6. What do PlusCal labels determine, and what does a placement decision assert?
7. Give five things you configure in a TLC model.
8. How do you check that your invariants are not vacuous?
9. Give the six modelling discipline rules.
10. Give six common TLA+ errors and the signature of each.

## 7. Primary sources

- **Lamport, *Specifying Systems* (2002)** — free from the author. Parts I and II are the course; the
  rest is reference.
- **Lamport, *The TLA+ Video Course*** — the fastest route through the first hours; watch it
  alongside Stage 1.
- **Wayne, *Practical TLA+* and learntla.com** — the practitioner's on-ramp, especially for PlusCal.
- **Newcombe et al., "How Amazon Web Services Uses Formal Methods" (CACM 2015)** — read again now
  that you can read the specifications.
- Lamport, "The PlusCal Algorithm Language" (ICTAC 2009).
- The `tlaplus/Examples` repository — specifications of Paxos, Raft, two-phase commit, the
  Byzantine generals, and many others. Read three.
- Ongaro's Raft TLA+ specification, alongside the Raft paper (DS-701 L05).
- The TLA+ Google Group and the community Discord, which are unusually helpful for a formal methods
  community and are the fastest route past a stuck model.

---

**Previous:** [L04](L04-model-checking.md) · **Next:**
[L06 — Specifying a Distributed Protocol](L06-specifying-a-protocol.md)
