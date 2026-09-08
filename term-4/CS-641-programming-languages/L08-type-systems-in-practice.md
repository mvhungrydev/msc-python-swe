# CS-641 · Lesson 08 — Type Systems in Practice: Gradual, Dependent, Linear

**Estimated study time:** 4 hours
**Prerequisites:** L03–L07

---

## 1. Orientation

Seven lessons of theory. This one uses it to read real type systems — including the one you
use every day — and to place them on the axes that actually distinguish them.

The organizing insight: **every type system is a point on a trade-off surface**, and the axes
are soundness, expressiveness, inference, and annotation burden. You cannot maximize all four,
and every real language's design is legible once you know which it prioritized and what it
gave up.

By the end you should be able to answer, precisely: what does a green `mypy` run actually
license you to believe?

## 2. Theory

### 2.1 The axes

**Soundness.** Do well-typed programs never go wrong (L03 §2.3)? Sound: ML, Haskell, Rust
(modulo `unsafe`), Elm. Deliberately unsound: Java (array covariance, L05 §2.4), C# (same),
TypeScript (many), Python (`Any`, L03 §2.6). Note that "unsound" is not an insult — every
unsoundness listed was a deliberate trade, and knowing which trade is the useful knowledge.

**Expressiveness.** What properties can you state? Ranges from "this is an integer" to "this
list has length n" to "this sorting function returns a sorted permutation of its input".

**Inference.** How much can be omitted? Full and principal (HM, L04), local (Python,
TypeScript, Scala), or none (Java pre-`var`, C).

**Annotation burden.** What the programmer must write. Not the inverse of inference —
dependent types have good inference in places and enormous burden in others.

**The tension is real.** More expressiveness generally costs inference (System F is
undecidable, L04 §2.2) and burden. More soundness costs expressiveness (a sound system must
reject some correct programs, CS-621 L10 §2.6). A language picks a point.

### 2.2 Gradual typing

The theory behind Python's annotations, TypeScript, and Sorbet.

**The idea** (Siek & Taha, 2006): a type system where `Any` (written `?`) is compatible with
everything, allowing typed and untyped code to interoperate.

The key relation is **consistency** (`~`), replacing subtyping at the boundary (SE-511 L06
§2.1):

- `Any ~ T` and `T ~ Any` for every T.
- Reflexive, symmetric, **not transitive**.

Non-transitivity is what makes it work: `int ~ Any` and `Any ~ str`, but `int ~ str` is false.
Without that, `Any` would collapse the whole system.

**The gradual guarantee** (Siek et al., 2015): removing annotations from a well-typed program
keeps it well-typed, and adding annotations only makes it *more* precise — it never changes a
program's behaviour except by rejecting it. This is the formal statement of "you can migrate
incrementally", and it is what makes gradual typing viable as a migration strategy.

**Sound gradual typing** inserts *runtime casts* at the boundary between typed and untyped
code, so that a type error is caught at the boundary rather than propagating. This preserves
soundness — and the performance cost was found to be severe (Takikawa et al., "Is Sound
Gradual Typing Dead?", POPL 2016, measured slowdowns over 10× in some configurations).

**So Python, TypeScript, and most practical gradual systems are *unsound*: they erase and do
not check.** That is a deliberate choice, and the cost is exactly what SE-511 L06 §2.2
enumerated: a green check is evidence proportional to how much of your code and your
dependencies are actually typed.

The practical consequence you should carry: **measure your `Any` surface** (`mypy
--any-exprs-report`), because that number, not "percent of functions annotated", is what your
green check is worth.

### 2.3 Dependent types

Types that depend on *values*:

```agda
Vec : Type → ℕ → Type
head : ∀ {A n} → Vec A (suc n) → A          -- cannot be called on an empty vector
append : ∀ {A m n} → Vec A m → Vec A n → Vec A (m + n)
```

`append`'s type states that the result's length is the sum. That is not documentation; it is
checked, and a wrong implementation does not compile.

Taken to its conclusion — **propositions as types** (the Curry–Howard correspondence) — a type
*is* a proposition and a program *is* its proof:

| Logic | Types |
|---|---|
| proposition | type |
| proof | program |
| implication `A → B` | function type |
| conjunction `A ∧ B` | product |
| disjunction `A ∨ B` | sum |
| `∀x. P(x)` | dependent function type |
| `∃x. P(x)` | dependent pair |
| normalization | evaluation |

This is one of the deepest results in the subject: **logic and computation are the same
thing**, discovered independently and repeatedly. Wadler's "Propositions as Types" (CACM 2015)
is the readable account and is worth an evening.

Practical systems: Agda, Idris, Coq, Lean, F*. What they buy: proofs of correctness, verified
compilers (CompCert), verified operating-system kernels (seL4), verified cryptography (used in
Firefox and Linux).

What they cost: **enormous** annotation burden; type checking that may not terminate (so
totality checking is required, and the language must forbid general recursion, CS-621 L10);
a steep learning curve; and small ecosystems.

The realistic assessment: dependent types are production-viable for *small, critical* systems
— a crypto primitive, a compiler's optimizer, a kernel's scheduler — and not for application
code. Their ideas leak into mainstream languages steadily: refinement types, const generics,
literal types, and exhaustiveness checking are all diluted versions.

### 2.4 Refinement types

The pragmatic middle: types plus a decidable predicate, checked by an SMT solver (FM-751 L06).

```
type Nat        = {v: Int | v >= 0}
type NonEmpty a = {v: [a] | len v > 0}

head :: NonEmpty a -> a
divide :: Int -> {v: Int | v /= 0} -> Int
```

The checker generates verification conditions and discharges them with Z3. Far less burden
than full dependent types, and it catches division by zero, index-out-of-bounds, and null
dereference *statically*.

Systems: Liquid Haskell, F*, Dafny, and — closest to home — `refinement` type experiments in
Python. It is the technique most likely to reach mainstream languages next, because the
solver does the work.

### 2.5 Linear and affine types

Types that constrain *how many times* a value is used.

- **Linear**: exactly once.
- **Affine**: at most once.
- **Relevant**: at least once.

**Rust's ownership system is affine types.** A value has one owner; moving it invalidates the
source; the borrow checker enforces that references do not outlive their referent.

What this buys — and it is remarkable — is **memory safety and data-race freedom with no
garbage collector and no runtime cost**:

- Use-after-free is impossible: the value's lifetime is known statically.
- Double-free is impossible: one owner.
- Data races are impossible: you may have many readers *or* one writer, never both, and that
  is exactly the invariant PY-601 L03 asks you to maintain by discipline.

Rust encodes in the type system what PY-601's design hierarchy asks you to maintain by
convention. That is the same relationship as between a type checker and a coding standard, one
level up.

The cost: a genuinely hard learning curve, and some correct programs are rejected (the
conservative side, CS-621 L10 §2.6). The `unsafe` escape hatch exists for those, and confining
it is how real Rust codebases work.

Linear types are appearing elsewhere: Haskell's `LinearTypes`, Clean's uniqueness types, and —
in a diluted form — Python's context managers, which enforce "used within this scope" by
convention rather than by type (PY-502 L07).

### 2.6 Reading real type systems

Placing them on the axes:

| Language | Sound | Inference | Expressiveness | Notable |
|---|---|---|---|---|
| Haskell | yes | full HM+ | high (type classes, GADTs, higher-rank) | purity in the type system |
| OCaml/ML | yes | full HM | medium-high | the value restriction |
| Rust | yes* | local | high (affine, traits, lifetimes) | memory safety without GC |
| Java | **no** | local (`var`) | medium | array covariance; erasure |
| C# | **no** | local | medium-high | reified generics |
| TypeScript | **no** | local, strong | high (structural, conditional, mapped) | deliberately unsound |
| Python | **no** | local, weak | medium (gradual, structural + nominal) | erased, not enforced |
| Go | yes | local | low (deliberately) | simplicity as a goal |
| Kotlin/Swift | mostly | local | medium-high | null safety in the type system |
| Agda/Idris/Lean | yes | partial | maximal | dependent; proofs |

`*` modulo `unsafe`.

**Read a system by asking:**

1. What can it *prove*? (Null safety? Exhaustiveness? Memory safety? Termination?)
2. Where is it *deliberately* unsound, and why was that traded?
3. How much can it infer, and where must you annotate?
4. What is the escape hatch, and how visible is it? (`Any`, `unsafe`, `as`, `# type: ignore`.)
5. Is it erased?

Those five questions, asked of any language, tell you what its types mean.

### 2.7 Python, assessed precisely

Applying the framework to the system you use:

**What it can prove**: nothing, formally. It is unsound in at least the ways SE-511 L06 §2.2
lists.

**What it reliably catches**: `None` handling with strict optional; wrong argument order for
same-arity calls of different types; misspelled attributes on annotated objects; unhandled
union members with exhaustiveness checking; and — the big one — the consequences of a
refactor, mechanically.

**What it cannot catch**: anything at a trust boundary (data arrives as `Any`); anything
depending on values; anything in an untyped dependency; anything mediated by `getattr`,
`eval`, or dynamic class creation (PY-502 L03); logical errors.

**Its escape hatches**: `Any` (invisible, and the important one), `cast` (visible),
`# type: ignore` (visible, and should require an error code).

**Its inference**: local and weak. Parameters must be annotated, because full inference with
subtyping and gradual typing is not available (L05 §2.8).

**The honest summary**, and it is the answer to this lesson's orienting question:

> A green `mypy --strict` run means: *within the fraction of your program where types are
> actually known* (measure it), a specific class of errors — mostly about names, arities,
> `None`, and union members — is absent. It is a strong statement about refactoring safety and
> a weak statement about correctness.

That is a genuinely valuable guarantee. It is not the guarantee people assume, and knowing the
difference is what this course was for.

### 2.8 Where things are going

Reasonable predictions, stated as such:

- **Gradual typing has won for dynamic languages.** Python, Ruby (Sorbet/RBS), PHP, and
  JavaScript all went this way, and the unsound/erased variant won on performance grounds.
- **Refinement types are the likely next mainstream step**, because SMT solvers do the work
  and the annotation burden is tolerable.
- **Affine/ownership types are spreading** from Rust — Swift's ownership, Mojo, and various
  proposals.
- **Effect systems** (L07 §2.7) are moving from research to practice, with OCaml 5 as the
  first mainstream instance.
- **Dependent types remain niche** for application code and increasingly standard for
  critical infrastructure.
- **Proof assistants are becoming practical for verification** of the small, high-value core:
  CompCert, seL4, and verified cryptography libraries are in production.

The meta-point worth carrying: **type systems are a compression of specification into
something machine-checkable**, and the field's direction is steadily toward compressing more
of the specification without proportionally more annotation. Every technique in this lesson is
a point on that trajectory.

## 3. Construction: positioning and extending

**Stage 1 — assess five type systems.** For Python, TypeScript, Java, Rust, and one ML-family
language, answer §2.6's five questions with *evidence*: write a program that demonstrates each
unsoundness you claim, and one that demonstrates each inference limit. Produce the table with
your programs as citations.

**Stage 2 — measure your Python.** On a real codebase:

- `mypy --any-exprs-report` — the `Any` surface as a percentage of expressions.
- The count of `# type: ignore`, by error code.
- The count of untyped dependencies (`--disallow-any-unimported`).
- The proportion of your program's data that enters from a trust boundary without validation.

Report the numbers, and then state — in one sentence — what a green run on that codebase
actually licenses.

**Stage 3 — add gradual typing to your language.** Add `Any` (`?`) with consistency instead of
equality at the boundaries. Then implement it **twice**:

- **Erased** (Python's choice): no runtime checks.
- **Sound** (with casts): insert a runtime check whenever a value crosses from `Any` to a
  known type.

Write a program that the erased version accepts and that crashes, and confirm the sound
version catches it at the boundary. Then **measure the overhead** of the sound version on a
benchmark. Report the slowdown, and connect it to Takikawa et al.'s result.

**Stage 4 — refinement types.** Add a simple refinement: `{v: Int | v > 0}`. Generate
verification conditions and discharge them with Z3 (`z3-solver` is already in your lab
environment). Support: non-negative integers, non-empty lists, and a divisor that is non-zero.
Report what it catches and what the annotation burden feels like.

**Stage 5 — affine types.** Add a `linear`/`affine` qualifier: a value so marked may be used at
most once. Implement the check (a usage count per binding, with care around branches: a value
used in both arms of an `if` is used once, not twice — working that out is the exercise).
Then use it to model a file handle that cannot be used after closing, and demonstrate the
checker rejecting a use-after-close.

**Stage 6 — the comparison.** For one small program with a real bug (a null dereference, an
index out of bounds, a use-after-close), write it in your language with each of: no types,
STLC types, gradual types, refinement types, and affine types. Report which systems catch the
bug, at what annotation cost, and with what error message quality.

That table is the deliverable of the lesson, and it is the concrete form of the trade-off
surface.

## 4. Failure modes

- **Believing a green type check means correctness.**
- **Not measuring the `Any` surface**, so the check's strength is unknown.
- **`# type: ignore` without an error code**, silencing future unrelated errors.
- **Assuming soundness.** Python, TypeScript, Java, and C# are all unsound in specific,
  documented ways.
- **Assuming unsoundness means uselessness.** The refactoring guarantee alone justifies the
  annotations.
- **Adopting dependent types for application code.** The burden is not proportionate.
- **Comparing type systems on one axis.** "Haskell's types are better than Python's" is
  underspecified; better at what, and at what cost?
- **Ignoring runtime validation** because the types "cover it" (SE-511 L06 §2.7).
- **Rejecting a type system because it rejects a correct program.** That is the conservative
  side of an unavoidable trade (CS-621 L10 §2.6), not a defect.

## 5. Exercises

### Warm-up (30 min)

**W1.** Write a Python program that passes `mypy --strict` and crashes at runtime. Then write
one that is correct and rejected. Name the mechanism in each case.

**W2.** Measure the `Any` surface of a real codebase and report the number.

**W3.** For three languages you know, answer §2.6's five questions, with one demonstrating
program each.

### Core (3 h)

**C1 — The assessment.** Complete §3 stages 1–2. Deliverable: the five-language table with
demonstrating programs for every claim, the measurements on your own codebase, and the
one-sentence statement of what your green run licenses.

**C2 — Gradual, both ways.** Complete §3 stage 3. Deliverable: both implementations, the
crashing program, the boundary catch, and the measured overhead of the sound version with the
comparison to the published result.

**C3 — Refinements.** Complete §3 stage 4. Deliverable: the refinement syntax, VC generation,
Z3 discharge, five programs it catches, and an honest report of the annotation burden.

**C4 — The comparison table.** Complete §3 stage 6. Deliverable: one program with a real bug
in five type systems, the table of what each catches at what cost, and 600 words on which
point on the trade-off surface you would choose for: a web application, a cryptographic
library, and a data pipeline — with different answers and reasons.

### Challenge

**X1.** Complete §3 stage 5 (affine types) fully, including the branch analysis, and use it to
enforce a real protocol: a file that must be opened, used, and closed exactly once; or a
connection that cannot be used after being returned to a pool. Then write 800 words on what
Python's context managers achieve by convention that affine types achieve by construction, and
whether a Python-level approximation (a linter check, PY-502 L07) is worth building.

**X2.** Read Wadler's "Propositions as Types" (CACM 2015), then prove three simple
propositions by writing programs of the corresponding types in a dependently typed language
(Idris, Agda, or Lean). Start with `A → (B → A)` and `(A → B) → (B → C) → (A → C)`. Then write
1,200 words on the correspondence: what it means that logic and computation are the same
thing, and what it implies about the relationship between testing and proving.

## 6. Self-check

1. Name the four axes and explain the tensions between them.
2. Define consistency and the gradual guarantee, and say why non-transitivity is essential.
3. Why are practical gradual systems unsound, and what was the measured cost of soundness?
4. State the Curry–Howard correspondence with five rows of the table.
5. What do refinement types buy over full dependent types, and what makes them feasible?
6. What do affine types give Rust, and what PY-601 discipline do they encode?
7. Give the five questions for reading any type system.
8. State precisely what a green `mypy --strict` run licenses you to believe.

## 7. Primary sources

- Siek & Taha, "Gradual Typing for Functional Languages" (2006); Siek et al., "Refined
  Criteria for Gradual Typing" (SNAPL 2015) — the gradual guarantee.
- Takikawa et al., "Is Sound Gradual Typing Dead?" (POPL 2016).
- **Wadler, "Propositions as Types" (CACM 2015).** Read it.
- Rondon, Kawaguchi & Jhala, "Liquid Types" (PLDI 2008).
- Wadler, "Linear Types Can Change the World!" (1990); the Rust Book's ownership chapters.
- Pierce, *TAPL*, ch. 30 and *Advanced Topics in Types and Programming Languages* for depth.
- The Python Typing Specification, read as a design document.

---

**Previous:** [L07](L07-effects-and-monads.md) · **Next:**
[L09 — Building It: The Interpreter and Type Checker](L09-building-it.md)
