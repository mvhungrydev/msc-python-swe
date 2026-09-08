# CS-641 · Lesson 01 — Syntax, Semantics, and the Lambda Calculus

**Estimated study time:** 4.5 hours
**Prerequisites:** none

---

## 1. Orientation

The lambda calculus is three lines of grammar:

```
t ::= x           variable
    | λx. t       abstraction  (a function)
    | t t         application  (calling a function)
```

That is the entire language. No numbers, no booleans, no `if`, no data structures, no
recursion construct. And it is Turing-complete (CS-621 L10 §2.1) — everything computable can
be written in it.

This matters for two reasons. First, it is the substrate on which type theory is built: every
type system you will meet is stated for a variant of this calculus, so learning it is learning
to read the literature. Second, and more practically: **Python's semantics are much closer to
the lambda calculus than to the imperative language it superficially resembles.** Closures,
first-class functions, decorators, and the scoping rules of PY-501 L07 are all lambda calculus
with syntax on top, and seeing that makes them obvious rather than arbitrary.

## 2. Theory

### 2.1 Syntax: concrete, abstract, and the difference

**Concrete syntax** is what you type: `λx. x` or `lambda x: x` or `\x -> x` or `x => x`. It
includes precedence, associativity, comments, and whitespace.

**Abstract syntax** is the tree: `Abs("x", Var("x"))`. It is what the rest of the system
operates on.

Parsing maps one to the other, and — this is the practically important point — **all the
semantics live in the abstract syntax.** Two languages with completely different concrete
syntax may have identical semantics. Most arguments about language syntax are therefore
arguments about ergonomics, and most arguments about semantics are the ones that matter.

Conventions for the lambda calculus's concrete syntax, which you must know to read anything:

- **Application associates left**: `f g h` means `(f g) h`.
- **Abstraction extends as far right as possible**: `λx. f x y` means `λx. (f x y)`, not
  `(λx. f x) y`.
- **`λx y. t` abbreviates `λx. λy. t`** (currying).

Those three conventions cause most early confusion. Write them down.

### 2.2 Bound and free variables

In `λx. t`, the `x` is **bound** in `t`. A variable occurrence not bound by any enclosing
lambda is **free**.

```
λx. x y          x is bound, y is free
λx. λy. x y      both bound
(λx. x) x        the first x is bound, the second is free
```

Formally:

```
FV(x)      = {x}
FV(λx. t)  = FV(t) \ {x}
FV(t₁ t₂)  = FV(t₁) ∪ FV(t₂)
```

A term with no free variables is **closed** (a *combinator*).

**α-equivalence.** Bound variable names do not matter: `λx. x` and `λy. y` are the same term.
This is why `def f(a): return a` and `def f(b): return b` are the same function, and it is
why renaming a local variable is always safe (PY-501 L07 — the compiler's variable
categorization does exactly this).

### 2.3 Substitution and the capture problem

**β-reduction** is the only computation rule:

```
(λx. t) v  →  t[x := v]
```

"Substitute v for the free occurrences of x in t." Everything else is machinery.

Substitution is defined by:

```
x[x := s]        = s
y[x := s]        = y                          (y ≠ x)
(t₁ t₂)[x := s]  = (t₁[x := s]) (t₂[x := s])
(λx. t)[x := s]  = λx. t                      (x is shadowed — do not go inside)
(λy. t)[x := s]  = λy. (t[x := s])            (y ≠ x, AND y ∉ FV(s))
```

That last side condition is the **capture problem**, and it is the single most common bug in a
hand-written interpreter.

```
(λx. λy. x) y     →  λy. y      ← WRONG
```

The free `y` being substituted got *captured* by the binder `λy`, silently changing the
meaning from "a constant function returning the outer y" to "the identity function". The fix
is **α-renaming** the binder first:

```
(λx. λy. x) y     →  (λx. λz. x) y  →  λz. y     ← correct
```

Three standard implementation strategies:

1. **Rename on demand** — generate a fresh name whenever the side condition fails. Simple,
   and correct if you are careful.
2. **De Bruijn indices** — replace names with a number counting enclosing binders:
   `λx. λy. x` becomes `λ. λ. 1`. α-equivalence becomes syntactic equality, capture becomes
   impossible, and substitution requires index shifting (which is fiddly but mechanical).
   This is what most real implementations use.
3. **Environments and closures** — do not substitute at all; carry a mapping from names to
   values, and represent a function as (parameter, body, environment). This is what every
   practical interpreter does, including CPython (PY-501 L07 §2.4 — `__closure__` is exactly
   this), and it is what L02 builds.

### 2.4 Inference rules: the notation

You will read hundreds of these. The notation is:

```
    premise₁    premise₂    …    premiseₙ
   ────────────────────────────────────────  (RULE-NAME)
                 conclusion
```

Read it as: *if all the premises hold, then the conclusion holds*. A rule with no premises is
an **axiom**. That is all there is to it.

Example — the β-reduction rules for call-by-value:

```
         t₁ → t₁'                          t₂ → t₂'
   ──────────────────  (E-App1)      ──────────────────  (E-App2)
    t₁ t₂ → t₁' t₂                     v₁ t₂ → v₁ t₂'


   ─────────────────────────────  (E-AppAbs)
    (λx. t) v → t[x := v]
```

Reading them together tells you the whole evaluation strategy: E-App1 says evaluate the
function first; E-App2 says *once the function is a value* (`v₁`), evaluate the argument;
E-AppAbs says once both are values, substitute. **The order is encoded in which
metavariables are `t` (any term) and which are `v` (already a value).** That is the trick to
reading these fluently, and once you see it the notation stops being opaque.

A **derivation** is a tree of rule applications proving a particular judgement. Building a
derivation by hand for a small term, three or four times, is what makes the notation
comfortable.

### 2.5 Evaluation strategies

Given `(λx. t) ((λy. y) z)`, do you evaluate the argument first?

**Call-by-value (CBV).** Evaluate arguments to values before substituting. Python, Java, C,
OCaml, JavaScript. Predictable evaluation order and predictable side effects; may do
unnecessary work; may not terminate where a lazier strategy would.

**Call-by-name (CBN).** Substitute the unevaluated argument. May duplicate work if the
argument is used many times.

**Call-by-need (lazy).** Call-by-name plus memoization: evaluate at most once, on first use.
Haskell. Enables infinite data structures and avoids unnecessary work; makes reasoning about
*when* things happen — and therefore about side effects and space — much harder.

**Normal order.** Reduce the leftmost-outermost redex. Its property: **if any strategy
terminates, normal order does.** CBV does not have this property:

```
(λx. λy. y) ((λx. x x)(λx. x x))
```

CBV loops forever evaluating the argument; normal order discards it immediately and returns
`λy. y`.

Python is CBV, with two important lazinesses bolted on: `and`/`or` short-circuit, and
generators (PY-502 L05) provide explicit laziness. `if` is a *statement*, not a function,
precisely because a function would evaluate both branches under CBV.

**Church–Rosser theorem.** If a term reduces to two different terms, they can both be reduced
to a common term. Consequence: **the normal form is unique if it exists** — the answer does
not depend on the strategy, only on whether you get there. That is the theorem that makes
"the value of an expression" well-defined.

### 2.6 Encoding everything

The claim that this three-line language is enough is made concrete by encodings. Each is worth
implementing once because the *technique* — represent data by its eliminator — recurs.

**Church booleans.** A boolean is a two-argument function that picks one:

```
true  = λt. λf. t
false = λt. λf. f
if    = λb. λt. λf. b t f
and   = λp. λq. p q p
```

`if true a b` reduces to `a`. The boolean *is* the conditional.

**Church numerals.** A number n is "apply f n times":

```
0 = λf. λx. x
1 = λf. λx. f x
2 = λf. λx. f (f x)

succ = λn. λf. λx. f (n f x)
plus = λm. λn. λf. λx. m f (n f x)
mult = λm. λn. λf. m (n f)
```

**Pairs.**

```
pair = λa. λb. λs. s a b
fst  = λp. p (λa. λb. a)
snd  = λp. p (λa. λb. b)
```

Note the pattern: a data structure is represented by a function that takes a *handler* and
applies it to the contents. This is the Church encoding, and it is the same idea as the
visitor pattern (SE-521 L04 §2.2) and as continuation-passing style.

**Recursion, via the Y combinator.** The lambda calculus has no `def`, so a function cannot
refer to itself by name. Fixed-point combinators solve this:

```
Y = λf. (λx. f (x x)) (λx. f (x x))
```

`Y g` reduces to `g (Y g)` — the function receives itself as an argument. For CBV you need
the Z combinator (a η-expanded Y) or `Y` diverges.

Implement a factorial with it. It takes an hour, it is genuinely mind-bending the first time,
and afterwards recursion is demystified: **recursion is not primitive; it is derivable.**

### 2.7 What Python inherits

Direct correspondences, worth listing because they make Python's design look inevitable
rather than arbitrary:

| Lambda calculus | Python |
|---|---|
| `λx. t` | `lambda x: t`, `def f(x): return t` |
| Application | `f(x)` |
| Free/bound variables | PY-501 L07's local/free/global categorization |
| Closures (environment strategy) | `__closure__` cells |
| α-equivalence | renaming a local is safe |
| Capture avoidance | why `exec` in a function does not bind (PY-501 L07 §2.8) |
| CBV | Python's evaluation order |
| Currying | `functools.partial`, decorators |
| Church encoding | the visitor pattern; callbacks |
| Y combinator | why you can write recursion without a name |

And one non-correspondence worth noting: Python's `lambda` is *not* the lambda calculus's
abstraction in one respect — Python's is limited to a single expression, which is a syntactic
restriction with no semantic content. The `def` form is the real abstraction.

## 3. Construction: a lambda calculus interpreter

Build it. This becomes the skeleton of the language you build across the course.

**Stage 1 — the AST.**

```python
from dataclasses import dataclass

@dataclass(frozen=True)
class Var:  name: str
@dataclass(frozen=True)
class Abs:  param: str; body: "Term"
@dataclass(frozen=True)
class App:  fn: "Term"; arg: "Term"

Term = Var | Abs | App
```

**Stage 2 — a parser.** Concrete syntax `\x. x` or `λx. x`, with the three conventions of
§2.1. A recursive-descent parser is about 60 lines. Get the associativity right and test it
with terms whose parse is ambiguous to the eye.

**Stage 3 — free variables and substitution, naively.** Implement `fv(t)` and
`subst(t, x, s)` *without* the capture check. Then find a term that demonstrates the capture
bug and write it as a failing test. **Do this deliberately** — meeting the bug is what makes
the side condition memorable.

**Stage 4 — fix it, three ways.**

- Capture-avoiding substitution with fresh-name generation.
- De Bruijn indices, with the shift/substitute operations.
- Environments and closures.

Test all three against each other on 1,000 random terms. They must agree, and getting them to
agree will find bugs in at least two of them.

**Stage 5 — evaluation.** Implement call-by-value, call-by-name, and normal order. Then find a
term where CBV diverges and normal order does not, and demonstrate it. Report the reduction
count for each strategy on a term that uses its argument three times — this is where CBN's
work duplication becomes visible.

**Stage 6 — the encodings.** Implement Church booleans, numerals, pairs, and lists. Write
`plus`, `mult`, `pred` (harder than it looks — look it up only after an honest attempt),
`is_zero`, and list `map`. Verify by converting Church numerals back to Python ints.

**Stage 7 — recursion.** Implement the Y combinator (and Z for CBV). Write factorial and
Fibonacci with it. This is the stage that produces the "oh" moment.

**Stage 8 — a REPL and a tracer.** A REPL that shows each reduction step. Watching
`(λx. x x)(λx. x x)` loop, and `Y g` unfold, is worth more than any amount of reading.

## 4. Failure modes

- **Variable capture.** The classic. Every naive interpreter has it.
- **Getting the association conventions wrong** in the parser: `λx. f x y` misparsed.
- **Substituting under a shadowing binder.** `(λx. t)[x := s]` must not recurse.
- **Confusing α-equivalence with syntactic equality.** `λx. x` and `λy. y` are the same term;
  a naive `==` says otherwise.
- **Implementing CBV and calling it normal order.** They differ on exactly the terms that
  matter.
- **The Y combinator diverging under CBV.** You need Z.
- **Assuming a normal form exists.** `(λx. x x)(λx. x x)` has none.
- **Confusing the calculus's `λ` with Python's `lambda`.** The restriction to one expression
  is Python's, not the calculus's.
- **Thinking the encodings are practical.** Church numerals are Θ(n) to represent n. They are
  an existence proof, not an implementation strategy.

## 5. Exercises

### Warm-up (30 min)

**W1.** For each of these, list the free and bound variables:
`λx. x y`, `(λx. x) x`, `λx. λy. z (λz. z x)`, `(λx. λy. x y) y`.

**W2.** Reduce by hand, showing each step: `(λx. λy. x) a b`, `(λf. f (f a)) (λx. x)`,
`(λx. x x) (λy. y)`.

**W3.** Demonstrate variable capture with the smallest term you can construct, and show the
α-renamed correct reduction.

### Core (3 h)

**C1 — The interpreter.** Complete §3, stages 1–5. Deliverable: parser, three substitution
strategies cross-verified on 1,000 random terms, three evaluation strategies, the capture bug
as a regression test, and the CBV-diverges/normal-order-terminates demonstration with the
reduction counts.

**C2 — The encodings.** Complete §3 stage 6. Deliverable: booleans, numerals, pairs, lists,
and the arithmetic, all verified by round-tripping through Python values. `pred` is the hard
one; attempt it for at least thirty minutes before looking it up, and write down what made it
hard.

**C3 — Recursion.** Complete §3 stage 7. Implement Y and Z, and factorial with both.
Demonstrate why Y diverges under CBV, by tracing. Then write 400 words explaining fixed-point
combinators to a colleague who knows Python but not this.

**C4 — Inference rules.** Write out, in the notation of §2.4, the complete small-step
semantics for call-by-value, call-by-name, and normal order. Then, for one term, build the
full derivation tree by hand for one reduction step under each strategy. This is the exercise
that makes the notation fluent.

### Challenge

**X1.** Implement de Bruijn indices completely, including a converter both ways from named
terms, and the shifting logic for substitution. Prove (informally, in writing) that your shift
and substitute operations are correct, and verify by round-tripping 10,000 random terms
through named → de Bruijn → named and checking α-equivalence. The shifting is genuinely
subtle; getting it right is the exercise.

**X2.** Read Pierce ch. 5 in full and implement the untyped lambda calculus exactly as
specified there, including his treatment of substitution. Then extend it with `let`, numbers,
and booleans as *primitives* rather than encodings, and write 600 words on what changes — in
particular, what the primitives buy that the encodings do not (the answer involves efficiency,
but also error messages and, in L03, types).

## 6. Self-check

1. Give the three-line grammar of the lambda calculus and the three concrete-syntax
   conventions.
2. Define free and bound variables formally.
3. State the substitution rules, including the side condition, and explain what it prevents.
4. Give three implementation strategies for avoiding capture, and say which real interpreters
   use.
5. Read an inference rule aloud and explain how evaluation order is encoded in the
   metavariables.
6. Distinguish call-by-value, call-by-name, call-by-need, and normal order, with a term
   separating CBV from normal order.
7. State the Church–Rosser theorem and its consequence.
8. Explain the Church encoding pattern and connect it to a design pattern you know.

## 7. Primary sources

- Pierce, *TAPL*, chs. 3 and 5. Chapter 5 is this lesson.
- Church, "An Unsolvable Problem of Elementary Number Theory" (1936).
- Barendregt, *The Lambda Calculus: Its Syntax and Semantics* — the reference, if you want
  depth.
- Krishnamurthi, *PLAI* — an implementation-first alternative treatment.
- Wadler, "Propositions as Types" (CACM 2015) — for the wider picture; read it once this term
  for pleasure.

---

**Next:** [L02 — Operational Semantics and Interpreters](L02-operational-semantics.md)
