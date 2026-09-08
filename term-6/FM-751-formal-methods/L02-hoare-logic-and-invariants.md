# FM-751 · Lesson 02 — Hoare Logic, Invariants, and Reasoning About Code

**Estimated study time:** 5 hours
**Prerequisites:** L01; CS-641 L02 (operational semantics), CS-621 L01

---

## 1. Orientation

Hoare logic is the formal system for proving things about sequential programs, and it is the
foundation of every static analyser, contract system, and verification tool you will use. It is also
— and this is the practical claim of the lesson — **the source of the single most transferable
skill in this course: identifying the loop invariant.**

An engineer who can state the invariant of a loop or a data structure can reason about code they
did not write, debug by asking "which invariant is broken?" rather than by stepping, write
assertions that carry weight, and know *why* their algorithm is correct rather than that it passed
its tests. That skill costs an afternoon to acquire and pays for the rest of a career, and it is
available whether or not you ever run a verifier.

The central notation:

> **{P} S {Q}** — if precondition P holds and statement S terminates, then postcondition Q holds.

Note the "if it terminates": this is **partial correctness**. Termination is a separate obligation
(§2.5), and separating them is not pedantry — it maps exactly onto L01's safety/liveness split, and
the same tools do not establish both.

## 2. Theory

### 2.1 The rules

Hoare's axiomatic system, in the standard notation of CS-641 L01 — premises above the line,
conclusion below.

**Assignment** (read it carefully; it runs backwards from the intuition):

```
─────────────────────
{ Q[e/x] } x := e { Q }
```

To ensure Q holds *after* `x := e`, require Q-with-e-substituted-for-x *before*. So
`{y > 0} x := y {x > 0}` is derived by taking Q = `x > 0` and substituting: `y > 0`. This
backwards direction is the basis of **weakest precondition** calculation (§2.4) and is the thing to
get comfortable with first, because everything else follows.

**Sequence**:

```
{P} S1 {R}    {R} S2 {Q}
────────────────────────
    {P} S1; S2 {Q}
```

**Conditional**:

```
{P ∧ b} S1 {Q}    {P ∧ ¬b} S2 {Q}
─────────────────────────────────
{P} if b then S1 else S2 {Q}
```

**Consequence** — the rule that makes the system usable, because it lets you strengthen a
precondition and weaken a postcondition:

```
P ⇒ P'    {P'} S {Q'}    Q' ⇒ Q
────────────────────────────────
          {P} S {Q}
```

**While** — the important one:

```
      {I ∧ b} S {I}
────────────────────────────
{I} while b do S {I ∧ ¬b}
```

Read what this says. If **I** holds before the loop, and each iteration preserves it, then **I** holds
after the loop *together with the negation of the guard*. `I` is the **loop invariant**, and it is
the entire content of the rule: there is no way to reason about a loop without one, which is why
finding it is the skill.

### 2.2 Finding a loop invariant

Not magic, and there is a method. The invariant must satisfy three conditions:

1. **Initialisation**: it is true before the first iteration.
2. **Preservation**: if true at the start of an iteration with the guard true, it is true at the
   end.
3. **Termination/usefulness**: when the loop exits, `I ∧ ¬b` implies the postcondition you wanted.

The third condition is where the technique becomes constructive: **work backwards from what you want
after the loop.** The invariant is usually "the postcondition, but only about the part processed so
far", plus whatever bounds the loop variable.

Worked example — linear search:

```python
# Precondition: a is a list
i = 0
# Invariant: 0 <= i <= len(a) and target not in a[0:i]
while i < len(a):
    if a[i] == target:
        return i        # Postcondition: a[i] == target
    i = i + 1
return -1               # Postcondition: target not in a
```

Check the three conditions. Initialisation: `i = 0`, so `a[0:0]` is empty and the target is
trivially not in it. Preservation: we only increment after establishing `a[i] != target`, so the
"not in" extends by one element. Usefulness: on exit `i = len(a)` and `target not in a[0:len(a)]`,
which is the postcondition exactly.

Notice the shape: **"what I have established about the portion processed so far" plus "the bounds on
my position".** Almost every array-loop invariant has that shape, and recognising it makes the rest
mechanical.

A second worked example — binary search, where the invariant is what makes correctness non-obvious:

```python
lo, hi = 0, len(a)
# Invariant: 0 <= lo <= hi <= len(a)
#            and all(a[k] < target for k in range(0, lo))
#            and all(a[k] >= target for k in range(hi, len(a)))
while lo < hi:
    mid = lo + (hi - lo) // 2
    if a[mid] < target:
        lo = mid + 1
    else:
        hi = mid
# Exit: lo == hi, so everything below lo is < target and everything from lo is >= target
```

That invariant is where binary search's off-by-one errors go to die. Bentley's observation that most
published binary searches were wrong — and the 2006 discovery that Java's had an integer overflow in
`(lo + hi) / 2` for large arrays, which is why the code above computes `lo + (hi - lo) // 2` — is the
standard cautionary tale, and the invariant is what makes the reasoning checkable.

### 2.3 Data structure invariants

The same idea, applied to a type rather than a loop:

> A **representation invariant** is a property of a data structure's internal state that holds
> whenever no operation is in progress. Every operation may assume it on entry and must re-establish
> it on exit.

Examples: a binary search tree's ordering property; a heap's parent-child ordering; a balanced
tree's height bound; "the free list contains exactly the unallocated blocks"; "the index contains
exactly the keys in the table" (DI-721 L03).

The engineering value is direct and does not require any tooling:

- **A bug is an invariant violation.** Debugging becomes "which invariant broke, and which operation
  broke it?", which is a much more tractable question than "why is the output wrong?".
- **`_check_invariant()` as a method**, called in tests and under a debug flag, converts a class of
  bugs from mysterious to immediately localised.
- **The invariant documents the design** better than prose, and it is checkable.
- **Concurrency makes it sharper** (PY-601 L03): the invariant may be violated *during* an
  operation, so the lock's job is precisely to prevent another thread observing that window. Stating
  the invariant tells you exactly what the critical section must cover.

### 2.4 Weakest preconditions

Dijkstra's reformulation, which turns proving into calculating:

> **wp(S, Q)** is the weakest precondition such that executing S is guaranteed to terminate in a
> state satisfying Q.

The rules run backwards through the program:

- `wp(x := e, Q) = Q[e/x]`
- `wp(S1; S2, Q) = wp(S1, wp(S2, Q))`
- `wp(if b then S1 else S2, Q) = (b ⇒ wp(S1,Q)) ∧ (¬b ⇒ wp(S2,Q))`
- Loops require the invariant; there is no purely mechanical rule.

The practical significance: **wp is mechanisable, and mechanising it is what verification tools
do.** Dafny, ESC/Java, Frama-C and their relatives compute a verification condition by pushing the
postcondition backwards through the program, then hand the resulting formula to an SMT solver (L09).
Understanding wp is understanding why those tools need your invariants — the loop is the one place
the calculation cannot proceed without human input, which is exactly why "the tool can't find my
invariant" is the normal experience rather than a failure.

### 2.5 Termination

`{P} S {Q}` says nothing about termination. To prove it, exhibit a **variant** (a ranking function):

- An expression over the program state, mapping into a well-founded set — usually the naturals.
- **Strictly decreasing** on every iteration.
- **Bounded below**.

Since there is no infinite strictly-decreasing sequence in a well-founded set, the loop terminates.

For linear search, the variant is `len(a) - i`. For binary search, `hi - lo`. For Euclid's
algorithm, the second argument. For a recursion over a tree, the subtree's height.

Note the correspondence to L01: **partial correctness is a safety property; termination is a
liveness property.** They are proved by different means — invariants for safety, variants for
liveness — and this pairing recurs throughout the course. In TLA+ (L05) the same split appears as
`□[Next]` versus fairness conditions.

For concurrent systems, termination generalises to *progress*, and the variant argument generalises
to a well-founded measure that decreases whenever the system takes a useful step — which is exactly
how one proves that a consensus protocol eventually decides (DS-701 L05).

### 2.6 Total correctness, and where practice sits

**Total correctness** = partial correctness + termination. Full verification of an implementation
establishes it, and for most software it is out of proportion.

Where the ideas do pay for practitioners, in order of value per hour:

1. **Invariants as thinking tools.** Free. State the invariant, and the code often writes itself
   correctly.
2. **Assertions in code.** `assert` the invariant at the top and bottom of an operation. Cheap,
   catches violations at the moment they occur rather than three functions later, and doubles as
   documentation (L10).
3. **Contracts** — preconditions and postconditions enforced at runtime, at least in testing
   (L10).
4. **Property-based tests** derived from postconditions (L08). The postcondition *is* the property.
5. **Static verification** where the risk justifies it: cryptographic code, a memory allocator, a
   parser handling hostile input, an OS kernel.

The industrial evidence for the top of that list is seL4 and CompCert at the extreme end — full
functional verification of a kernel and a C compiler, at costs of tens of person-years — and, at the
practical end, the observation that most engineers who learn to state invariants write better code
immediately, with no tools at all.

## 3. Construction: prove things about real code

Build in `mpse/fm751/l02/`. Proofs by hand first — a proof you have not written out is a proof you
have not done.

**Stage 1 — the rules by hand.** Prove five small programs correct using the Hoare rules explicitly,
writing every step: swap two variables; integer division by repeated subtraction; maximum of an
array; reversing a list in place; and summing an array. Every application of consequence must be
justified.

**Stage 2 — the invariant catalogue.** For ten standard algorithms — linear search, binary search,
insertion sort, selection sort, partition (from quicksort), heapify, Euclid's GCD, exponentiation by
squaring, the two-pointer merge, and Dutch national flag — state the loop invariant, verify all
three conditions, and give the variant. The Dutch national flag one is the hardest and the most
instructive.

**Stage 3 — break it.** Take three of Stage 2's algorithms and introduce a subtle off-by-one. For
each: identify precisely which of the three invariant conditions fails, and construct the smallest
input that exhibits the bug. Then check whether your existing test suite would have caught it —
often it would not, and the invariant analysis found it in seconds.

**Stage 4 — binary search, properly.** Write binary search, prove it correct against the §2.2
invariant, and prove termination with the variant. Then examine the overflow bug in
`(lo + hi) // 2`: state precisely which precondition it violates, and in which languages it can
occur (and why Python's arbitrary-precision integers make it a non-issue there while it remains real
in C, Java and Rust).

**Stage 5 — representation invariants.** Take three data structures you have built in this
programme — the LSM memtable and SSTable index (DI-721 L02), the Raft log (DS-701 L05), and a CRDT
(DS-701 L07). For each: state the representation invariant precisely, implement `_check_invariant()`,
and call it in tests after every operation. Then find a sequence of operations that violates one —
or convince yourself none exists, and say what your argument rests on.

**Stage 6 — concurrency.** Take a concurrent data structure from PY-601. State its invariant, and
identify the exact window during each operation in which it is violated. Then show that the locking
covers exactly that window — and find whether it covers more than necessary (a performance cost) or
less (a bug). Then deliberately narrow the critical section to expose the bug and demonstrate it
with a stress test.

**Stage 7 — weakest preconditions.** Implement a `wp` calculator for a small imperative language:
assignment, sequence, conditional, and assert, with loops requiring an annotated invariant. Given a
program and a postcondition, output the verification condition. Then check three of Stage 1's
programs with it and compare against your hand proofs.

**Stage 8 — an SMT-backed verifier.** Extend Stage 7: discharge the verification conditions with Z3
(anticipating L09). Now you have a small program verifier. Run it on Stage 1's programs, then on
Stage 3's broken ones, and confirm it rejects them with a counterexample. Then find a correct program
your verifier cannot prove, and explain why — the answer will be about the invariant it needed and
could not infer, which is §2.4's point made concrete.

**Stage 9 — assertions in production code.** Take a real component of your term artifact and add
invariant assertions at operation boundaries, controlled by a flag. Measure the performance cost.
Then run your chaos experiments (DS-701 L10) with assertions enabled and report whether any fired.
Then decide: would you ship with them on? State the reasoning, including what an assertion failure
should *do* in production (L10 develops this).

## 4. Failure modes

- **Reasoning about loops without an invariant.** There is no other way; the alternative is hoping.
- **An invariant that is preserved but useless.** `true` is preserved by everything and implies
  nothing. Check the third condition.
- **An invariant that is useful but not preserved.** Check every path through the body, including
  the early return.
- **Forgetting initialisation.** The invariant is often false before the loop starts, and the fix is
  usually in the setup rather than the invariant.
- **Proving partial correctness and calling it correct.** A non-terminating program satisfies every
  partial correctness specification vacuously.
- **A variant that decreases but is not bounded below.** Not a termination proof.
- **Representation invariants unstated.** Debugging becomes archaeology.
- **Invariants checked only in tests.** The interesting violations happen under production
  concurrency and load.
- **Assertions with side effects.** They vanish when assertions are disabled, and the behaviour
  changes.
- **Assertions used for input validation.** Assertions check *your* invariants; input validation
  checks the *caller's* obligations, and disabling assertions must not disable input validation.

## 5. Exercises

### Warm-up (30 min)

1. Give the Hoare rules for assignment, sequence, conditional, consequence and while. Explain why
   the assignment rule runs backwards.
2. State the three conditions a loop invariant must satisfy, and explain how the third makes finding
   it constructive.
3. Give the variant for linear search, binary search, Euclid's GCD and a tree recursion.

### Core (3.5 h)

4. Complete Stages 1–2: five hand proofs with every consequence justified, and ten invariants with
   their three conditions and variants.
5. Complete Stages 3–4: three subtle bugs localised to a specific failing condition, with a test
   suite comparison; and binary search proved with the overflow precondition analysed.
6. Complete Stage 5 for all three data structures.
7. Complete Stage 6, including the deliberately-narrowed critical section and its stress test.

### Challenge

8. Complete Stages 7–8: the wp calculator, the Z3-backed verifier, and the correct program it
   cannot prove with the explanation.
9. Complete Stage 9, then take a **published algorithm with a known subtle bug** — the Java binary
   search overflow, the TimSort merge invariant bug that de Gouw et al. found by verification in
   2015, or a CVE arising from a broken invariant — and write the analysis: state the intended
   invariant, show precisely where it fails, construct the input that exhibits it, and explain why
   testing did not find it. The TimSort case is the best one to work through, because the bug was
   found by an attempt at verification, the invariant involved is genuinely subtle, and the code had
   been in every Java and Python installation for years.

## 6. Self-check

1. State the Hoare triple's meaning, including what "partial" excludes.
2. Give the while rule and explain why the invariant is its entire content.
3. Give the three conditions on a loop invariant and the method for finding one.
4. Give the invariant of binary search and explain what it makes checkable.
5. What is a representation invariant, and what does it turn debugging into?
6. In a concurrent structure, what is the relationship between the invariant and the critical
   section?
7. Define wp and explain why loops are the one place it needs human input.
8. What is a variant, and what property of the ordering makes the argument work?
9. Map partial correctness and termination onto safety and liveness.
10. Give the five ways these ideas pay for a practitioner, in order of value per hour.

## 7. Primary sources

- **Hoare, "An Axiomatic Basis for Computer Programming" (CACM 1969)** — six pages; read the
  original.
- **Dijkstra, *A Discipline of Programming* (1976)** — weakest preconditions, and the argument for
  deriving programs from specifications rather than debugging them into correctness.
- Gries, *The Science of Programming* (1981) — the best textbook treatment of invariant discovery;
  the worked examples are excellent.
- Bentley, *Programming Pearls*, the binary search column — and the 2006 Google Research blog post
  on the overflow bug.
- de Gouw, Rot, de Boer, Bubel & Hähnle, "OpenJDK's `java.utils.Collection.sort()` Is Broken"
  (CAV 2015) — the TimSort result; a verification effort finding a real bug in ubiquitous code.
- Klein et al., "seL4: Formal Verification of an OS Kernel" (SOSP 2009) — the extreme end, with an
  honest account of the cost.
- Leroy, "Formal Verification of a Realistic Compiler" (CACM 2009) — CompCert, and the argument for
  where full verification pays.
- The Dafny tutorial — the most accessible way to see wp-based verification working, and it will
  make L09 easier.

---

**Previous:** [L01](L01-specifications-as-artifacts.md) · **Next:**
[L03 — Temporal Logic: Safety and Liveness](L03-temporal-logic.md)
