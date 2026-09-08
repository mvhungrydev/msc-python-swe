# CS-641 · Lesson 05 — Subtyping, Variance, and Bounded Quantification

**Estimated study time:** 4.5 hours
**Prerequisites:** L03, L04; SE-511 L07

---

## 1. Orientation

SE-511 L07 gave you the variance rules and a recipe for deriving them. This lesson derives
them from a single principle, proves the ones that are theorems, and shows exactly where real
languages broke them and what it cost.

The principle is one sentence:

> **S is a subtype of T if a value of type S can be used wherever a T is expected, without
> anything going wrong.**

Everything else — covariance, contravariance, invariance, Java's array unsoundness, Python's
`list` being invariant — follows from applying that sentence carefully. Once you can derive
the rules, you stop memorizing them and start noticing when an API's variance is wrong.

## 2. Theory

### 2.1 Subtyping and subsumption

Add a relation `S <: T` and one typing rule:

```
    Γ ⊢ t : S      S <: T
   ────────────────────────  (T-Sub)
          Γ ⊢ t : T
```

**Subsumption**: a term of a subtype may be used at the supertype. This single rule is the
whole of subtyping's effect on the type system — everything else is the definition of `<:`.

Basic properties: `<:` is **reflexive** (`T <: T`) and **transitive**. Together with T-Sub
these make it a preorder, and they are what let you reason compositionally.

Note that T-Sub is **not syntax-directed** — it applies to any term, so a naive checker does
not know when to use it. Real implementations restructure the rules to make checking
algorithmic (§2.6).

### 2.2 Records: width, depth, permutation

The classic example, and the one that generates the intuition:

```
    ────────────────────────────────────────────────────  (S-RcdWidth)
     {l₁:T₁, …, lₙ:Tₙ, lₙ₊₁:Tₙ₊₁, …} <: {l₁:T₁, …, lₙ:Tₙ}


     for each i:  Sᵢ <: Tᵢ
    ────────────────────────────────  (S-RcdDepth)
     {l₁:S₁, …, lₙ:Sₙ} <: {l₁:T₁, …, lₙ:Tₙ}
```

**Width**: a record with *more* fields is a subtype. `{name: Str, age: Int} <: {name: Str}` —
extra information is harmless to someone expecting less.

**Depth**: field types may be replaced by subtypes.

**Permutation**: field order does not matter (in a structural system).

Width subtyping is exactly Python's `Protocol` matching (PY-502 L01 §2.3): a class with more
methods than the Protocol requires satisfies it.

### 2.3 Functions: the rule everyone gets backwards

```
     T₁ <: S₁        S₂ <: T₂
    ──────────────────────────  (S-Arrow)
      S₁ → S₂  <:  T₁ → T₂
```

Note the **flip** in the first premise. Derive it rather than memorizing it:

Suppose `f : S₁ → S₂` and someone expects a `T₁ → T₂`. They will:

1. **Call f with a T₁.** For that to be safe, f must accept any T₁ — so f's parameter type
   must be *at least as general*: `T₁ <: S₁`. **Contravariant in the parameter.**
2. **Use the result as a T₂.** For that to be safe, f's result must be *at least as specific*:
   `S₂ <: T₂`. **Covariant in the result.**

So: **accept more, promise more specifically.** A function that takes an `Animal` and returns
a `Dog` is usable wherever one taking a `Dog` and returning an `Animal` is expected.

This is one instance of a general principle worth stating on its own: **inputs are
contravariant, outputs are covariant.** Every variance question reduces to asking where the
type parameter appears.

### 2.4 Variance derived

For a generic `F[T]`, ask: **where does T appear?**

| T appears | Variance | Relation | Example |
|---|---|---|---|
| Output only | **covariant** | `S <: T ⟹ F[S] <: F[T]` | `Sequence[T]`, `Iterator[T]`, `frozenset[T]` |
| Input only | **contravariant** | `S <: T ⟹ F[T] <: F[S]` | `Callable[[T], R]` in T, `Comparator[T]` |
| Both | **invariant** | no relation | `list[T]`, `MutableSequence[T]`, `set[T]` |

**Why mutable containers must be invariant** — the proof, and it is worth writing out because
it is the argument that settles every real dispute:

Suppose `list[Dog] <: list[Animal]`. Then:

```python
dogs: list[Dog] = [Dog()]
animals: list[Animal] = dogs        # allowed by the assumption
animals.append(Cat())               # type-correct: Cat is an Animal
dogs[1].bark()                      # runtime error: it is a Cat
```

The `append` — an *input* position for T — is what breaks it. A read-only view has no input
position and is safely covariant, which is exactly why `Sequence[T]` is covariant and
`list[T]` is not.

**This is the argument for the read/write interface split** (PY-502 L01 §3, SE-511 L07 §3): a
type that supports both reading and writing is necessarily invariant, so splitting it into a
covariant reader and a contravariant writer recovers the flexibility. CQRS falls out of the
same reasoning (SE-521 L06 §2.6), which is not a coincidence.

**Java's array unsoundness.** Java made arrays covariant (`String[] <: Object[]`) in 1995,
before generics existed, so that `Arrays.sort(Object[])` could sort anything. The consequence
is exactly the program above, and Java handles it with a **runtime check** on every array
store, throwing `ArrayStoreException`. Every array write in every Java program pays for this
decision.

That is the clearest available example of "a soundness hole traded for expressiveness, with
an ongoing runtime cost". C# repeated it; Scala, Kotlin, and Python did not.

### 2.5 Top, bottom, and the lattice

**Top (⊤)**: a supertype of everything. Python's `object`. You can do almost nothing with it
without narrowing, which is correct — that is what it means to know nothing.

**Bottom (⊥)**: a subtype of everything. Python's `Never`/`NoReturn`. **No values inhabit it**,
which is why a function returning `Never` cannot return normally — it must raise or loop
forever. `sys.exit()` and `assert_never()` are typed this way, and it is what makes
exhaustiveness checking work (SE-511 L06 §2.5).

Note what `Any` is *not*: it is not top. Top is "I know nothing about this value"; `Any` is
"do not check this". `Any` is compatible with everything in *both* directions, which is not a
subtyping relation at all — it is the deliberate hole of gradual typing (L08).

**Joins and meets.** With subtyping you often need a least upper bound: the type of
`if c then dog else cat` should be their join. In a lattice this exists; in Python's system,
unions serve the purpose (`Dog | Cat`), which is a different and in some ways better design —
it loses no information.

### 2.6 Algorithmic subtyping

The declarative rules (§2.1) are not directly implementable: T-Sub applies anywhere, and
transitivity requires guessing an intermediate type. The standard fix is to prove the
declarative and algorithmic systems equivalent, then implement the algorithmic one:

- Push subsumption into the specific rules where it is needed (at application, at assignment,
  at return).
- Make subtyping a *recursive function* `is_subtype(S, T)` rather than a relation with
  reflexivity and transitivity rules.

```python
def is_subtype(s: Type, t: Type) -> bool:
    if s == t: return True                                  # reflexivity
    if isinstance(t, Top): return True
    if isinstance(s, Bottom): return True
    match s, t:
        case (Arrow(sp, sr), Arrow(tp, tr)):
            return is_subtype(tp, sp) and is_subtype(sr, tr)      # ← note the flip
        case (Record(sf), Record(tf)):
            return all(l in sf and is_subtype(sf[l], tf[l]) for l in tf)
        case (Generic(n1, a1, vs), Generic(n2, a2, _)) if n1 == n2:
            return all(_check_variance(v, x, y) for v, x, y in zip(vs, a1, a2))
    return False
```

Termination is a real question once you add recursive types: `is_subtype` on
`μX. {next: X}` needs a coinductive treatment or a memo of assumed pairs. Amber's rules are
the standard answer.

### 2.7 Bounded quantification

Combining polymorphism with subtyping: `∀α <: T. σ`. The type variable ranges only over
subtypes of T.

```python
def largest[T: Comparable](xs: Sequence[T]) -> T: ...
```

`T: Comparable` is a bound (SE-511 L07 §2.3). Inside the function you may use `Comparable`'s
operations on T values; outside, T is preserved exactly, so `largest(dogs)` returns a `Dog`,
not a `Comparable`.

That last point is what a bound buys over just taking `Sequence[Comparable]`: the *relationship*
between input and output element type is preserved.

**F-bounded polymorphism**: `∀α <: F[α]` — the bound mentions the variable. This is how you
express "a type comparable with itself":

```python
class Comparable[T](Protocol):
    def __lt__(self, other: T) -> bool: ...

def sort[T: Comparable[T]](xs: list[T]) -> list[T]: ...
```

Java's `<T extends Comparable<T>>` is the canonical instance, and it is why that declaration
looks so strange. `Self` types (PEP 673) solve a related problem more directly.

**System F<:** (System F with bounded quantification) is well studied and its **subtyping is
undecidable** in the full version (Pierce, 1992) — another entry in CS-621 L10's catalogue.
Practical languages use restricted variants ("kernel F<:") that are decidable.

### 2.8 Subtyping meets inference

HM inference (L04) uses **unification**: symmetric equality of types. Subtyping needs
**subsumption**: an asymmetric relation. Combining them is genuinely hard.

- Unification asks "make these equal"; subtyping asks "is this acceptable here".
- Full subtyping inference requires solving *subtype constraints* (`α <: Int`, `Str <: β`),
  and the constraint sets grow large and the principal types become huge and unreadable.
- Practical languages therefore compromise: **local type inference** (Pierce & Turner) infers
  within an expression but requires annotations at function boundaries.

**That is exactly Python's design.** `mypy` infers locals and return types, and requires
parameter annotations, because Python has subtyping (nominal *and* structural), overloading,
and gradual typing — and inferring across all of that globally would be both slow and
productive of appalling error messages.

So "why must I annotate parameters in Python but not in Haskell?" has a precise answer:
**Haskell chose a system where full inference is decidable and principal; Python chose
subtyping and gradual typing, which forecloses that.** Neither is wrong; they are different
points on a real trade-off.

## 3. Construction: subtyping in your language

Continue from L04.

**Stage 1 — records.** Add record types and terms. Implement width, depth, and permutation
subtyping. Test: `{a:Int, b:Bool} <: {a:Int}`; `{a:Int} </: {a:Int, b:Bool}`;
`{a:Dog} <: {a:Animal}` given `Dog <: Animal`.

**Stage 2 — functions.** Implement S-Arrow with the flip. Then write the test that would catch
getting it backwards: construct a program that type-checks under the wrong rule and fails at
runtime.

**Stage 3 — derive the runtime failure.** Implement covariant mutable containers deliberately
(the Java choice). Write the `dogs`/`animals`/`Cat` program. Verify it type-checks and crashes.
Then either add the runtime store check (Java's answer) or make them invariant (Python's) —
implement both and compare: what does each cost, and what does each permit?

**Stage 4 — variance annotations.** Let the programmer declare variance
(`class Box[+T]`, `class Sink[-T]`) and **check it**: verify that a covariant parameter appears
only in output positions. Report a good error when it does not. This check is a small piece of
static analysis and it is exactly what a real compiler does.

**Stage 5 — algorithmic subtyping.** Implement `is_subtype` as §2.6. Verify it agrees with the
declarative rules on 1,000 random type pairs (generate types, check both ways). Then add
recursive types and discover why termination is now non-trivial; implement the assumed-pairs
memo.

**Stage 6 — bounded quantification.** Add `∀α <: T. σ`. Implement instantiation with the bound
checked. Then implement F-bounded quantification and write the `sort` example.

**Stage 7 — top and bottom.** Add `⊤` and `⊥`. Verify: everything is a subtype of ⊤; ⊥ is a
subtype of everything; no term has type ⊥ unless it diverges or raises. Then implement
exhaustiveness checking using ⊥ (as `assert_never` does) and demonstrate it catching a missing
case.

**Stage 8 — inference and subtyping.** Attempt to combine your L04 inference with subtyping.
You will find it does not work directly. Document precisely where it fails, then implement
**local type inference** instead: infer within expressions, require annotations at function
boundaries. Report what you lost. This stage is the one that makes Python's design choice make
sense.

## 4. Failure modes

- **Getting S-Arrow backwards.** Contravariant in the parameter, covariant in the result.
- **Covariant mutable containers.** Unsound, as Java demonstrates at runtime cost.
- **Declaring a type parameter covariant when it appears in an input position.** The checker
  must catch this; if yours does not, it is unsound.
- **Confusing `Any` with top.** `Any` is not a type in the lattice.
- **Assuming `<:` is antisymmetric.** With structural types, two differently-written types can
  be mutual subtypes and thus equivalent; equality of types is a separate question.
- **Non-terminating `is_subtype`** on recursive types.
- **Expecting HM-style full inference with subtyping.** It does not exist in a usable form.
- **Protocols with mutable attributes** declared covariant (PY-502 L01 §2.3).
- **A `Never`-returning function that returns.** Your checker should reject it; the type
  asserts non-return.

## 5. Exercises

### Warm-up (30 min)

**W1.** Derive S-Arrow from the substitution principle, in writing, without looking at it.

**W2.** Write the covariant-mutable-container unsoundness in Python's type system by declaring
a covariant TypeVar in a mutable Protocol, and see whether `mypy` catches it.

**W3.** For six standard library generics, determine the variance and justify it from where the
parameter appears.

### Core (3 h)

**C1 — Records and functions.** Complete §3 stages 1–3. Deliverable: the subtyping rules
implemented, the wrong-flip test, and both responses to covariant mutable containers with the
comparison.

**C2 — Variance checking.** Complete §3 stage 4. Deliverable: variance annotations, the
position check, good error messages, and a test suite of five correct and five incorrect
variance declarations.

**C3 — Algorithmic subtyping with recursion.** Complete §3 stage 5. Deliverable: `is_subtype`,
the agreement test against the declarative rules on random type pairs, recursive types, and the
termination fix.

**C4 — Inference meets subtyping.** Complete §3 stage 8. Deliverable: the documented failure of
naive combination, local type inference implemented, and a 600-word note explaining Python's
annotation requirement in these terms — written so a colleague who has not taken this course
would find it convincing.

### Challenge

**X1.** Implement bounded and F-bounded quantification (§3 stage 6) with a decidable subtyping
algorithm, and test it on the `sort`/`Comparable` example. Then read Pierce's "Bounded
Quantification is Undecidable" (1992) and construct an example that makes your checker diverge
— or explain why your restriction prevents it.

**X2.** Take Python's typing specification's variance rules and *prove* three of them from the
substitution principle, then find one place where the specification's rule is more permissive
than the principle strictly allows (there are a few, for usability). Write 800 words on why the
deviation was accepted.

## 6. Self-check

1. State the substitution principle and the subsumption rule.
2. Derive S-Arrow, explaining both premises.
3. Give the input/output rule for variance and apply it to five generics.
4. Prove that mutable containers must be invariant.
5. Explain Java's array covariance, its cost, and why Python did not repeat it.
6. Distinguish top, bottom, and `Any`.
7. What does a bound in `∀α <: T` buy over taking `T` directly?
8. Why is subtyping hard to combine with HM inference, and what is Python's answer?

## 7. Primary sources

- Pierce, *TAPL*, chs. 15 (subtyping), 16 (algorithmic subtyping), 26–28 (bounded
  quantification and F<:).
- Cardelli & Wegner (1985), §§3–4.
- Liskov & Wing, "A Behavioral Notion of Subtyping" (TOPLAS 1994) — subtyping as a
  *behavioural* rather than merely structural notion.
- Pierce, "Bounded Quantification is Undecidable" (POPL 1992).
- Pierce & Turner, "Local Type Inference" (TOPLAS 2000) — the approach Python and TypeScript
  use.
- The Python Typing Specification, the variance and protocol chapters.

---

**Previous:** [L04](L04-polymorphism-and-inference.md) · **Next:**
[L06 — Algebraic Data Types and the Expression Problem](L06-adts-and-expression-problem.md)
