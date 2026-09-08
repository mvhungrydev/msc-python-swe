# CS-641 · Lesson 04 — Polymorphism and Hindley–Milner Inference

**Estimated study time:** 5 hours
**Prerequisites:** L03

---

## 1. Orientation

In STLC, the identity function must be written once per type: `λx:Bool. x`, `λx:Int. x`,
`λx:Bool→Bool. x`. That is unusable, and fixing it is the origin of generics.

Two things are needed, and they are separable:

1. **Polymorphism** — a type language that can express "for all types T, T → T".
2. **Inference** — an algorithm that finds those types without annotations.

Hindley–Milner gives you both, with a remarkable property: **it infers the most general type
of any expression, with no annotations at all, and it always terminates.** That is why ML,
OCaml, Haskell, F#, Elm, and (partially) Rust and Swift can omit type annotations while
remaining fully statically typed.

Understanding HM tells you exactly why Python's inference is weaker, why `mypy` sometimes
needs an annotation, and why some type systems are undecidable.

## 2. Theory

### 2.1 Kinds of polymorphism

Cardelli & Wegner's taxonomy, which is the standard vocabulary:

- **Parametric.** One implementation works uniformly for all types. `len` on a list works
  regardless of element type *because it does not look at the elements*. This is generics.
- **Ad-hoc (overloading).** Different implementations chosen by type. `+` on ints and on
  strings. Python's `__add__` dispatch; Haskell's type classes; C++'s overloading.
- **Subtype (inclusion).** A value of a subtype is usable where a supertype is expected (L05).
- **Row/structural.** Any record with the required fields. Python's Protocols
  (PY-502 L01 §2.3).

Parametric polymorphism has a property the others lack, and it is the deep one — §2.7.

### 2.2 System F: explicit polymorphism

Add type abstraction and type application:

```
Types  T ::= X | T → T | ∀X. T
Terms  t ::= x | λx:T. t | t t | ΛX. t | t [T]
```

`ΛX. t` abstracts over a *type*; `t [T]` applies a type.

```
id = ΛX. λx:X. x           :  ∀X. X → X
id [Bool] true             :  Bool
id [Int] 3                 :  Int
```

System F is enormously expressive — you can encode all the algebraic data types in it. But:

> **Theorem (Wells, 1994).** Type inference for System F is *undecidable*.

You must write the type applications. That is unusable in practice (imagine writing
`id [Int] 3` everywhere), which is why real languages restrict to something inferable.

CS-621 L10 §2.4 listed this among the undecidable problems; here is where it bites.

### 2.3 Hindley–Milner: the restriction that makes inference work

HM (also called *let-polymorphism* or prenex polymorphism) restricts System F in two ways:

1. **Quantifiers only at the outermost level.** `∀X. X → X` is allowed; `(∀X. X → X) → Int` is
   not. These are called **type schemes** (σ) as distinct from **types** (τ):

   ```
   τ ::= α | τ → τ | Int | Bool | List τ        (monotypes)
   σ ::= τ | ∀α. σ                              (type schemes / polytypes)
   ```

2. **Generalization only at `let`.** A `let`-bound variable can be used at several types; a
   lambda-bound parameter cannot.

That second restriction is why `let` and `(λx. …) …` differ, which L02 §2.4 flagged:

```
let id = λx. x in (id true, id 3)          ✓  id : ∀α. α → α, used at Bool and at Int
(λid. (id true, id 3)) (λx. x)             ✗  id's type is a monotype; it cannot be both
```

Every ML-family programmer meets this and it is the price of decidable inference.

The result:

> **Theorem (Hindley 1969, Milner 1978, Damas–Milner 1982).** Type inference for HM is
> decidable, and every typable term has a **principal type** — a most general type such that
> every other valid type is an instance of it.

Principality is the property that makes inference *useful*: the algorithm does not merely find
*a* type, it finds *the best* type, so no information is lost.

### 2.4 Unification

The engine. Given a set of type equations, find a substitution making both sides equal.

```python
def unify(a: Type, b: Type, s: Subst) -> Subst:
    a, b = apply(s, a), apply(s, b)
    match a, b:
        case (TVar(x), _):
            if a == b: return s
            if occurs(x, b): raise InfiniteType(x, b)        # ← the occurs check
            return s | {x: b}
        case (_, TVar(_)):
            return unify(b, a, s)
        case (Arrow(p1, r1), Arrow(p2, r2)):
            s = unify(p1, p2, s)
            return unify(r1, r2, s)
        case (TCon(n1, args1), TCon(n2, args2)) if n1 == n2 and len(args1) == len(args2):
            for x, y in zip(args1, args2):
                s = unify(x, y, s)
            return s
        case _:
            raise TypeMismatch(a, b)
```

**The occurs check** is the interesting part. Unifying `α` with `α → Int` would produce the
infinite type `((… → Int) → Int) → Int`. The check rejects it, and this is exactly what makes
`λx. x x` untypable:

```
x : α, and x applied to x needs α = α → β.  Occurs check fails.
```

Which means the Y combinator (L01 §2.6) is untypable in HM — hence `fix`/`letrec` as a
primitive with the type `∀α. (α → α) → α` (L03 §2.6). Everything connects.

Omitting the occurs check gives you **equirecursive types**, which some systems use
deliberately; without care it gives you an infinite loop in the checker.

Unification is a general algorithm (Robinson, 1965) used far beyond type inference: Prolog
resolution, pattern matching, and SMT solving all use it.

### 2.5 Algorithm W

Milner's inference algorithm. Structural recursion producing a substitution and a type.

```python
def infer(env: TypeEnv, t: Term, s: Subst) -> tuple[Subst, Type]:
    match t:
        case Var(x):
            if x not in env: raise UnboundVariable(x)
            return s, instantiate(env[x])            # fresh vars for each ∀

        case Abs(x, body):
            tv = fresh()
            s, tb = infer(env | {x: Scheme([], tv)}, body, s)
            return s, Arrow(apply(s, tv), tb)

        case App(f, a):
            s, tf = infer(env, f, s)
            s, ta = infer(env, a, s)
            tv = fresh()
            s = unify(apply(s, tf), Arrow(ta, tv), s)
            return s, apply(s, tv)

        case Let(x, val, body):
            s, tv = infer(env, val, s)
            scheme = generalize(apply(s, env), apply(s, tv))    # ← generalization
            return infer(env | {x: scheme}, body, s)
```

Two operations carry the polymorphism:

**`instantiate(σ)`** — replace each quantified variable with a *fresh* type variable. So each
*use* of `id` gets its own variables and can unify with a different type.

**`generalize(env, τ)`** — quantify over the type variables free in τ but **not free in the
environment**:

```python
def generalize(env: TypeEnv, t: Type) -> Scheme:
    return Scheme(sorted(free_vars(t) - free_vars(env)), t)
```

The exclusion is essential. A variable free in the environment is *constrained* by something
outside — perhaps a lambda parameter whose type is still being determined — and quantifying
over it would let you use it at two incompatible types and produce an unsound result. Getting
this condition wrong is the classic Algorithm W bug, and it makes the checker *unsound* rather
than merely wrong.

### 2.6 The value restriction

HM as stated is unsound in the presence of mutable references. The classic counterexample:

```ml
let r = ref [] in          (* r : ∀α. α list ref  — generalized! *)
r := [1];                  (* used at int list ref *)
List.hd !r ^ "boom"        (* used at string list ref — type-checks, crashes *)
```

The problem: `ref []` was generalized, so the *same* cell can be used at two types.

**The value restriction** (Wright, 1995): only generalize when the right-hand side is a
**syntactic value** — a literal, a variable, or a lambda; not an application. `ref []` is an
application, so it is not generalized, and the program is rejected.

This is a *syntactic* approximation of a semantic property ("does this allocate mutable
state?"), which is undecidable (CS-621 L10 §2.5). It rejects some safe programs — the
conservative side of the choice (CS-621 L10 §2.6, strategy 1) — and it is what every
ML-family language uses.

**The generalizable lesson**: mutation and polymorphism interact badly, and the tension is
fundamental rather than an implementation artifact. Every language with both has some rule
about it, and Python's is "there is no generalization, so the question does not arise".

### 2.7 Parametricity: theorems for free

Wadler's result, and the most beautiful thing in this course.

A parametrically polymorphic function **cannot inspect the values it is generic over**. From
the *type alone*, you can derive theorems about the function's behaviour.

`f : ∀α. α → α` — the only total, terminating function with this type is the identity.
The function cannot construct an α (it does not know what α is) so it must return its
argument. **The type is a proof.**

`f : ∀α. [α] → [α]` — whatever f does, it can only permute, drop, or duplicate elements; it
cannot invent or examine them. It follows that:

```
map g (f xs) = f (map g xs)          for every g
```

f commutes with map. That theorem holds for `reverse`, `tail`, `sort`… wait, not `sort` —
`sort` needs `∀α. Ord α ⇒ [α] → [α]`, and the constraint is exactly what lets it inspect
elements. **The constraint's presence in the type is what breaks the free theorem**, which is
a lovely illustration of types carrying real information.

`f : ∀α. [α] → Int` — can only depend on the *length*.

The engineering consequences are practical:

- **A more general type is a stronger specification.** If a function can be typed
  `∀α. [α] → [α]`, giving it type `[Int] → [Int]` throws information away and permits
  implementations the general type forbids.
- **Constraints tell you what a function can look at.** In Python, `Sequence[T]` versus
  `Sequence[SupportsLessThan]` is the same distinction.
- **Type-directed reasoning is real.** "What can a function of this type possibly do?" is
  often answerable and often narrows the possibilities to one.

Python's type system is far too weak for parametricity to hold formally — `Any`, `isinstance`,
and reflection all break it. But the *reasoning* transfers: a function annotated
`Callable[[Sequence[T]], Sequence[T]]` that inspects elements is doing something its type says
it should not, and that is a design smell you can now name.

### 2.8 What HM cannot do, and how real languages cope

The limits, each of which explains a real language feature:

- **No higher-rank types.** `(∀α. α → α) → Int` is inexpressible. Haskell adds `RankNTypes`
  with mandatory annotations; Python has no equivalent, which is why you occasionally cannot
  type a higher-order function precisely.
- **No ad-hoc polymorphism.** Plain HM cannot type `+` for both Int and Float. Haskell adds
  type classes; OCaml uses modules; Python uses runtime dispatch (PY-502 L01 §2.6).
- **No subtyping.** HM's unification is symmetric equality; subtyping needs *subsumption*, and
  combining the two makes inference much harder (L05).
- **The value restriction** rejects some safe programs.
- **Error messages are poor.** Unification fails at a point that may be far from the actual
  mistake — a wrong type in one function surfaces as a mismatch three functions away. This is
  a genuine, well-known usability problem and the subject of ongoing research.
- **Exponential worst case.** HM inference is DEXPTIME-complete in theory (nested lets
  compounding type size); in practice it is linear on real programs.

**Python's inference** is much weaker: local, no generalization, no principal types. `mypy`
infers a variable's type from its initializer and a function's return from its body, but it
requires parameter annotations because it does not solve constraints globally. That is a
deliberate choice — global inference across a large dynamically-loaded codebase would be slow
and would produce the poor error messages above, attributed to the wrong file.

Knowing this, "why do I have to annotate parameters?" has an answer: **because Python did not
choose HM, and the reasons are principled.**

## 3. Construction: implementing inference

Continue your language from L03. This is the largest single piece of implementation in the
course.

**Stage 1 — types and schemes.** Represent monotypes (variables, constructors, arrows) and
schemes (`∀ᾱ. τ`). Implement `free_vars` for types, schemes, and environments.

**Stage 2 — substitution.** `apply(s, τ)`, composition of substitutions. Get composition
right: `(s₂ ∘ s₁)` applies s₁ then s₂, and confusing the order produces bugs that manifest far
away.

**Stage 3 — unification.** §2.4, with the occurs check. Test: unifying `α` with `α → Int` must
raise; unifying `α → β` with `Int → Bool` must give `{α↦Int, β↦Bool}`; unification must be
symmetric.

**Stage 4 — Algorithm W.** §2.5. Then verify the classics:

```
λx. x                        ⟹  ∀α. α → α
λf. λx. f (f x)              ⟹  ∀α. (α → α) → α → α
λx. λy. x                    ⟹  ∀α β. α → β → α
let id = λx. x in id id      ⟹  ∀α. α → α          (works)
λid. id id                   ⟹  occurs check fails  (correctly)
λx. x x                      ⟹  occurs check fails
```

**Stage 5 — the generalization condition.** Deliberately implement `generalize` *without* the
`− free_vars(env)` exclusion. Find a program that now type-checks but is unsound, and verify
your L03 soundness property test catches it. Then fix it. **This exercise is the point of the
stage**: it makes the condition memorable and demonstrates that it is load-bearing.

**Stage 6 — the value restriction.** Add mutable references to your language. Reproduce the
`ref []` unsoundness. Then implement the value restriction and confirm it is rejected. Report
a safe program your restriction now also rejects, and say whether you would accept that.

**Stage 7 — error messages.** HM's weakness. Improve on "cannot unify Int with Bool" by:
tracking the source span of each constraint, reporting *both* locations that forced the
conflicting types, and showing the inferred types of the surrounding expressions. Compare
before and after on five realistic errors. This is genuinely hard and genuinely valuable, and
partial progress is a good outcome.

**Stage 8 — principality.** For twenty terms, verify that the inferred type is the most
general: check that any other type you can hand-write for the term is an *instance* of the
inferred one. Write the instance check.

**Stage 9 — annotations.** Allow optional type annotations, and check them for consistency
with the inferred type rather than replacing it. Report a case where the annotation is *less*
general than the inferred type and decide whether to accept it (most languages do, and it is
worth knowing why).

## 4. Failure modes

- **Generalizing over variables free in the environment.** Unsound. The classic bug.
- **Omitting the occurs check.** Infinite types, and often an infinite loop.
- **Wrong substitution composition order.** Bugs that appear far from the cause.
- **Not applying the accumulated substitution before unifying.** Stale types.
- **Generalizing at lambda binders**, not just `let`. Breaks decidability.
- **No value restriction with mutable state.** Unsound (§2.6).
- **Fresh variable collisions.** Use a global counter or a proper supply; reusing names is a
  silent disaster.
- **Errors reported at the unification site**, far from the real mistake.
- **Expecting HM to handle subtyping or overloading.** It cannot; that is a different system.
- **Assuming inference implies no annotations are ever useful.** Annotations are documentation
  and error-locality even when inferable — which is why Haskell programmers write top-level
  signatures by convention.

## 5. Exercises

### Warm-up (30 min)

**W1.** Infer by hand, showing the constraints and the substitution:
`λf. λg. λx. f (g x)`.

**W2.** Show why `λx. x x` fails, naming the check.

**W3.** Show the `let`/lambda asymmetry with a concrete pair of programs, and explain it in one
sentence.

### Core (3.5 h)

**C1 — The inference engine.** Complete §3 stages 1–4. Deliverable: unification with tests
(including symmetry and the occurs check), Algorithm W, and the six classic terms inferring
correctly. Cross-check with a reference implementation or with OCaml/Haskell's inferred types
for equivalent terms.

**C2 — The generalization bug.** Complete §3 stage 5. Deliverable: the broken version, the
unsound program it accepts, confirmation that the soundness property test catches it, the fix,
and 300 words explaining precisely why the condition is necessary.

**C3 — The value restriction.** Complete §3 stage 6. Deliverable: the `ref []` unsoundness
demonstrated, the value restriction implemented, and a safe-but-rejected program with your
judgement on the trade.

**C4 — Error messages.** Complete §3 stage 7. Deliverable: five realistic type errors, before
and after, with the spans and the two conflicting locations reported. Then write 400 words on
why HM error messages are hard, referring to the fact that unification loses the *order* in
which constraints arose.

### Challenge

**X1.** Read Wadler's "Theorems for Free!" and derive the free theorem for three polymorphic
types of your choosing. Then test them: for `∀α. [α] → [α]`, write ten implementations and
verify empirically that all satisfy `map g ∘ f = f ∘ map g`. Then write a Python function
annotated with an equivalent generic type that *violates* the theorem (by using `isinstance`),
and write 600 words on what Python's type system therefore does not guarantee.

**X2.** Implement type classes (or a comparable mechanism for ad-hoc polymorphism) on top of
your HM inference: qualified types `∀α. C α ⇒ τ`, constraint collection during inference,
constraint solving, and dictionary-passing elaboration. Then explain how this relates to
Python's Protocols and to Rust's traits. This is a substantial project and it is the natural
next step after HM.

## 6. Self-check

1. Name four kinds of polymorphism with an example of each.
2. Why is System F inference undecidable, and what does HM restrict to make it decidable?
3. Explain the `let`/lambda asymmetry and why it exists.
4. What is a principal type and why does it matter?
5. What does the occurs check prevent, and which famous term does it reject?
6. State the generalization condition and explain what goes wrong without it.
7. State the value restriction and the general lesson about mutation and polymorphism.
8. State a free theorem and explain what a type-class constraint does to it.

## 7. Primary sources

- **Milner, "A Theory of Type Polymorphism in Programming" (JCSS 1978).** Algorithm W.
- Damas & Milner, "Principal Type-Schemes for Functional Programs" (POPL 1982).
- Pierce, *TAPL*, chs. 22 (reconstruction) and 23 (System F).
- **Wadler, "Theorems for Free!" (FPCA 1989).** Read it; it is short and it is the best paper
  in this course.
- Cardelli & Wegner, "On Understanding Types, Data Abstraction, and Polymorphism" (1985).
- Wright, "Simple Imperative Polymorphism" (1995) — the value restriction.
- Robinson, "A Machine-Oriented Logic Based on the Resolution Principle" (1965) —
  unification.
- Wells, "Typability and type checking in System F are equivalent and undecidable" (1999).

---

**Previous:** [L03](L03-simply-typed-lambda-calculus.md) · **Next:**
[L05 — Subtyping, Variance, and Bounded Quantification](L05-subtyping-and-variance.md)
