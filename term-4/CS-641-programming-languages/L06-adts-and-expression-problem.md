# CS-641 · Lesson 06 — Algebraic Data Types and the Expression Problem

**Estimated study time:** 4 hours
**Prerequisites:** L03, L05

---

## 1. Orientation

Two ways to model a shape that can be a circle, a square, or a triangle:

```python
# 1. A class hierarchy — the object-oriented way
class Shape(ABC):
    @abstractmethod
    def area(self) -> float: ...
class Circle(Shape): ...
class Square(Shape): ...

# 2. A closed union with external functions — the functional way
Shape = Circle | Square | Triangle

def area(s: Shape) -> float:
    match s:
        case Circle(r): return math.pi * r * r
        case Square(a): return a * a
        case Triangle(b, h): return b * h / 2
        case _: assert_never(s)
```

Adding a *new shape* is easy in (1) and touches every function in (2).
Adding a *new operation* is easy in (2) and touches every class in (1).

This is the **expression problem**, and it is not a matter of taste — it is a genuine
limitation that most mainstream languages cannot escape. Recognizing which axis your system
varies along is one of the most useful pieces of design judgement in this course, and it
decides a real architectural question every time you model a domain.

## 2. Theory

### 2.1 Algebraic data types

Types built from two operations, with an algebra that justifies the name:

**Product types** — "and". A tuple, a record, a struct. `(A, B)` has `|A| × |B|` values.

**Sum types** — "or". A tagged union. `A | B` has `|A| + |B|` values.

The counting is not a coincidence: types form a semiring under these operations, with
`Unit` (one value) as the multiplicative identity and `Void`/`Never` (no values) as the
additive identity. `A × 1 = A`; `A + 0 = A`; and — pleasingly — the function type behaves as
exponentiation: `|A → B| = |B|^|A|`.

That algebra gives you real reasoning: `(A + B) × C = A × C + B × C` is a *refactoring* — a
record containing a union is equivalent to a union of records, and you can pick whichever
models your domain better.

**The design consequence**: sum types let you make illegal states unrepresentable.

```python
# Bad: 2 × 2 × 2 = 8 states, of which 3 are valid
@dataclass
class Connection:
    is_connected: bool
    socket: Socket | None
    error: str | None

# Good: exactly 3 states
Connection = Disconnected | Connected | Failed
```

The first admits `is_connected=True, socket=None` — a state your code must defend against
everywhere. The second cannot express it. **Counting the states a type admits and comparing
with the states your domain has** is a five-minute exercise that repeatedly finds real design
errors (SE-521 L06 §2.2's value objects are the same idea).

### 2.2 In Python

```python
from dataclasses import dataclass
from typing import assert_never

@dataclass(frozen=True)
class Circle:   radius: float
@dataclass(frozen=True)
class Square:   side: float
@dataclass(frozen=True)
class Triangle: base: float; height: float

Shape = Circle | Square | Triangle          # a closed union

def area(s: Shape) -> float:
    match s:
        case Circle(r):        return math.pi * r * r
        case Square(a):        return a * a
        case Triangle(b, h):   return b * h / 2
        case _:                assert_never(s)
```

Three mechanisms make this work as a real ADT:

- **`X | Y`** (PEP 604) is the sum.
- **`match`** (PEP 634) is the eliminator, with `__match_args__` giving positional patterns
  (PY-501 L06 §2.8).
- **`assert_never`** gives **exhaustiveness checking**: add a fourth shape and every
  `assert_never` becomes a type error pointing at the function that must be updated
  (SE-511 L06 §2.5).

That last point is what makes the pattern usable at scale. Without exhaustiveness checking,
adding a variant is a grep-and-hope; with it, the type checker enumerates the work.

**The limits** of Python's version, stated honestly: the union is not *sealed* — anyone can
add a class and forget to add it to the union; there is no compiler-enforced link between the
union and its members; and matching is nominal, so `Protocol`s do not participate.

### 2.3 The expression problem, stated

Wadler's formulation (1998):

> Define a datatype by cases, where one can **add new cases** and **add new functions over the
> datatype**, without recompiling existing code, and while retaining static type safety.

The four requirements: extensibility in both dimensions, no modification of existing code, and
static safety.

| | Add a case | Add an operation |
|---|---|---|
| **Class hierarchy** (OO) | easy — a new subclass | hard — touch every class |
| **ADT + functions** (FP) | hard — touch every function | easy — a new function |

This is a genuine trade, not a failure of imagination. Neither arrangement is better; they are
optimized for different directions of change.

**The design question, therefore: which axis does your system vary along?**

- **Types stable, operations growing** → ADT + functions. An AST (the node types are fixed by
  the language; you keep adding passes: type-check, optimize, pretty-print, compile). A
  protocol message. A domain event.
- **Operations stable, types growing** → class hierarchy. A plugin system. A set of payment
  providers implementing a fixed interface. A driver model.
- **Both grow** → you have a real problem; §2.5.

Getting this backwards is the source of a specific and recognizable pain: a plugin system
where adding a plugin requires editing a central `match`, or an AST where adding a compiler
pass requires editing forty node classes.

### 2.4 Visitors: making OO grow operations

The visitor pattern converts a class hierarchy from "easy to add types" to "easy to add
operations", by inverting the dispatch:

```python
class ShapeVisitor(Protocol[R]):
    def visit_circle(self, c: Circle) -> R: ...
    def visit_square(self, s: Square) -> R: ...

class Circle:
    def accept(self, v: ShapeVisitor[R]) -> R: return v.visit_circle(self)
```

Now a new operation is a new visitor class; a new *type* requires changing the visitor
interface and every implementation. **The trade has been reversed, not eliminated** — which is
the honest summary of the visitor pattern and the thing the design-patterns literature often
obscures.

In Python, `functools.singledispatch` (PY-502 L01 §2.6) achieves the same inversion with much
less ceremony, and `match` achieves it with none. Visitors in Python are almost always a Java
idiom transplanted (SE-521 L04 §2.4).

Where visitors still earn their place: when you need the *double dispatch* explicitly (an
operation whose behaviour depends on two types), or when the traversal itself is complex and
you want to share it across operations (a generic AST walker with per-node hooks — which is
exactly `ast.NodeVisitor`, SE-511 L08 §2.3).

### 2.5 Solutions, and their costs

Several approaches genuinely solve the expression problem, and every one of them is expensive:

**Type classes / traits** (Haskell, Rust). Adding a type means implementing the class; adding
an operation means declaring a new class and instances. Both are additive, no existing code
changes. **This is the cleanest solution**, and it needs a language feature Python lacks —
though `functools.singledispatch` plus Protocols is a partial approximation.

**Open recursion with mixins.** Structure the datatype so cases are composable. Works, and
the types become elaborate.

**Object algebras** (Oliveira & Cook, 2012). Encode the datatype as an interface of
constructors, parameterized over the result type. Genuinely solves it in Java and similar
languages, and the encoding is unidiomatic enough that few use it.

**Tagless final** (Carette, Kiselyov & Shan, 2009). Represent terms as calls to an interface
rather than as data; different interpretations are different implementations. Elegant, and
common in Scala and Haskell.

**Multimethods** (CLOS, Julia, Clojure). Dispatch on all arguments; both dimensions extend
freely. Python has `singledispatch` (one argument only), and `multimethod` libraries.

The practical position: **most systems do not need the full solution.** They vary predominantly
along one axis. Identify which, pick the matching representation, and accept that the other
direction is more expensive. Reaching for object algebras when the type set has not changed in
two years is exactly the over-abstraction PY-502 L10 warns about.

### 2.6 Generalized ADTs, briefly

GADTs let each constructor specify a *more precise* result type:

```haskell
data Expr a where
  IntLit  :: Int  -> Expr Int
  BoolLit :: Bool -> Expr Bool
  Add     :: Expr Int -> Expr Int -> Expr Int
  If      :: Expr Bool -> Expr a -> Expr a -> Expr a
```

Now `Add (BoolLit True) x` is a *type error* — the AST cannot represent an ill-typed program.
The evaluator becomes total: `eval :: Expr a -> a`, with no error case at all.

This is a genuinely striking technique: the interpreter's type correctness is guaranteed by
the host language's type system, so L03's soundness theorem is discharged by the compiler
rather than by your proof.

Python cannot express this. You can approximate with generics and `TypeGuard`, but the
compiler will not enforce the constructor's refinement. Worth knowing the technique exists, so
that when you meet an interpreter in Haskell or Scala with no error cases you understand why.

### 2.7 Pattern matching as an eliminator

Every type has an **introduction** form (how to build a value) and an **elimination** form
(how to use one). For sums, the eliminator is a case analysis — and pattern matching is its
syntax.

The typing rule:

```
    Γ ⊢ t : T₁ + T₂     Γ, x:T₁ ⊢ t₁ : T     Γ, y:T₂ ⊢ t₂ : T
   ───────────────────────────────────────────────────────────  (T-Case)
      Γ ⊢ case t of inl x ⇒ t₁ | inr y ⇒ t₂ : T
```

Note what the rule requires: **every branch must be covered, and all branches must have the
same type.** Exhaustiveness is not a linting nicety — it is what makes the rule sound. A
non-exhaustive match is a partial function, and the type says it is total.

Python's `match` does not enforce exhaustiveness at runtime (a non-matching value falls
through), which is why `assert_never` exists: it moves the check to the type checker. Use it
in every match over a closed union, without exception.

Richer pattern languages add: nested patterns, guards, as-patterns, or-patterns, and
**pattern-match compilation** into an efficient decision tree (Maranget's algorithm), which
also computes exhaustiveness and redundancy as a by-product. A pattern-match compiler is a
genuinely enjoyable thing to write and is L09 material.

## 3. Construction: ADTs in your language

Continue from L05.

**Stage 1 — sums and products.** Add: tuples/records (products, from L05), and variants
(sums), with introduction (`inj_l`, or named constructors) and elimination (`case`). Write the
typing rules first.

**Stage 2 — named ADTs.** Let the programmer declare:

```
data Shape = Circle Float | Square Float | Triangle Float Float
```

which introduces a type name and constructor functions. Implement the declaration processing,
the constructor typing, and the case elimination.

**Stage 3 — pattern matching.** Nested patterns, variable binding, wildcards, and literals.
Type the patterns: a pattern for type T binds variables with the types the constructor gives
them. Get nested patterns right — that is where the work is.

**Stage 4 — exhaustiveness and redundancy.** Implement the check. Report:

- Missing cases, *naming them* (`Triangle` is not handled).
- Redundant cases (unreachable because an earlier pattern subsumes them).

Maranget's "Warnings for pattern matching" gives the algorithm. This is the most valuable
piece of the lesson to implement, because it is the feature that makes ADTs usable at scale.

**Stage 5 — the expression problem, empirically.** Implement a small AST evaluator two ways in
your own language (or in Python, if your language is not yet expressive enough):

- ADT + functions.
- Class hierarchy with methods.

Then perform four changes and record the diff for each:

| | Add a node type | Add an operation |
|---|---|---|
| ADT + functions | ? files, ? lines | ? files, ? lines |
| Class hierarchy | ? files, ? lines | ? files, ? lines |

Report the table. The numbers make the trade concrete in a way the prose does not.

**Stage 6 — illegal states.** Take three types from a real codebase of yours. For each: count
the states the type admits, count the states the domain has, and report the ratio. Redesign one
with sum types so the counts match, and report what defensive code you were able to delete.

**Stage 7 — a solution.** Implement one of §2.5's solutions (multimethods are the most
tractable in Python) and re-run stage 5's four changes. Report whether both directions are now
cheap, and what the encoding cost in readability.

## 4. Failure modes

- **Boolean flags encoding a state machine.** `is_connected`/`socket`/`error` — count the
  states.
- **A non-exhaustive match with no `assert_never`.** Adding a variant silently produces wrong
  behaviour rather than a type error.
- **`case _:` catching everything**, which defeats exhaustiveness checking. Use `assert_never`
  in the default arm, or omit the default.
- **Choosing the representation without asking which axis varies.**
- **A visitor pattern in Python** where `match` or `singledispatch` would do.
- **`Optional` everywhere** instead of a proper sum. `User | None` is a sum with one
  informative case and one uninformative one; often the domain has three states, not two.
- **An open union pretending to be closed.** Anyone can add a class; nothing links it to the
  union.
- **Nested patterns implemented naively**, producing exponential code or wrong bindings.
- **Reaching for object algebras** when the type set is stable.

## 5. Exercises

### Warm-up (30 min)

**W1.** For three types in a real codebase, count admitted states versus domain states. Report
the worst ratio.

**W2.** Write a closed union with `assert_never`, add a variant, and observe every affected
function become a type error. Then remove the `assert_never` and observe the silence.

**W3.** Use the type algebra to show `(A + B) × C ≡ A × C + B × C`, and give a domain example
where each side reads better.

### Core (3 h)

**C1 — ADTs in your language.** Complete §3 stages 1–3. Deliverable: typing rules for sums,
products, and patterns; the implementation; and a test suite covering nested patterns, guards,
and binding.

**C2 — Exhaustiveness.** Complete §3 stage 4. Deliverable: the checker, reporting missing cases
by name and redundant cases by position, with tests including nested and overlapping patterns.
This is the assessed centrepiece.

**C3 — The expression problem, measured.** Complete §3 stage 5. Deliverable: both
implementations, the four changes, the diff table, and 400 words on which representation you
would choose for an AST, for a plugin system, and for a domain-event type — with reasons.

**C4 — Illegal states.** Complete §3 stage 6. Deliverable: the state counts for three types,
one redesign, and a list of the defensive checks you deleted. Report the line count removed.

### Challenge

**X1.** Implement Maranget's pattern-match compilation algorithm: compile a set of patterns
into a decision tree that tests each scrutinee position at most once, and derive exhaustiveness
and redundancy warnings from the same structure. Compare the generated tree's efficiency
against naive sequential matching on a realistic pattern set.

**X2.** Read Wadler's expression-problem email (1998), Oliveira & Cook on object algebras
(ECOOP 2012), and Carette, Kiselyov & Shan on tagless final (2009). Implement one solution in
Python and write 1,200 words assessing it: does it actually satisfy all four of Wadler's
requirements in Python, what does it cost in readability and tooling support, and when — if
ever — would you use it in production code? An honest "never, and here is why" is a full-credit
answer.

## 6. Self-check

1. Define product and sum types, and give the counting algebra including the identities.
2. Show how sum types make illegal states unrepresentable, with the state count argument.
3. State the expression problem with all four of Wadler's requirements.
4. Give the two-by-two table and say which representation suits which direction of change.
5. What does the visitor pattern do to the trade-off, and what does it not do?
6. Name three genuine solutions and their costs.
7. Why is exhaustiveness required by the typing rule, not merely desirable?
8. What do GADTs buy for an interpreter, and why can Python not express them?

## 7. Primary sources

- Wadler, "The Expression Problem" (java-genericity mailing list, 1998). One page.
- Pierce, *TAPL*, chs. 11 (sums, variants) and 20 (recursive types).
- Maranget, "Warnings for Pattern Matching" (JFP 2007) — exhaustiveness and redundancy.
- Oliveira & Cook, "Extensibility for the Masses: Practical Extensibility with Object
  Algebras" (ECOOP 2012).
- Carette, Kiselyov & Shan, "Finally Tagless, Partially Evaluated" (JFP 2009).
- PEPs 604 (union syntax), 634–636 (structural pattern matching — 635 is the rationale and is
  the interesting one).

---

**Previous:** [L05](L05-subtyping-and-variance.md) · **Next:**
[L07 — Effects, Evaluation Strategies, and Monads](L07-effects-and-monads.md)
