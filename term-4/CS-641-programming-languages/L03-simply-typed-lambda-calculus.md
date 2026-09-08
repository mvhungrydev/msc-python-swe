# CS-641 · Lesson 03 — Simply-Typed Lambda Calculus: Progress and Preservation

**Estimated study time:** 5 hours
**Prerequisites:** L01, L02

---

## 1. Orientation

Robin Milner's slogan, 1978:

> **"Well-typed programs cannot go wrong."**

This is the central claim of type theory, and it is a *theorem*, not a hope — for languages
where it holds. This lesson states it precisely, proves it, and then examines exactly which
words in it are doing work.

"Go wrong" means *get stuck* (L02 §2.1): reaching a term that is not a value and cannot step.
It does not mean "cannot crash", "cannot loop forever", or "cannot be wrong". A well-typed
program can divide by zero, exhaust memory, or compute the wrong answer. What it cannot do is
apply a boolean as a function.

Understanding that scope precisely is what makes you accurate about what `mypy` gives you
(SE-511 L06 §2.2) — and about what a *sound* type system would give you, which is more but
still not everything.

## 2. Theory

### 2.1 The language

Simply-typed lambda calculus (STLC) with booleans:

```
Types      T ::= Bool | T → T
Terms      t ::= x | λx:T. t | t t | true | false | if t then t else t
Values     v ::= λx:T. t | true | false
Contexts   Γ ::= ∅ | Γ, x:T
```

Note `λx:T. t` — the parameter is **annotated**. Type inference (L04) removes that
requirement; for now, annotations make the rules simple.

A **typing context** Γ maps variables to types. The judgement `Γ ⊢ t : T` reads "under the
assumptions Γ, term t has type T".

### 2.2 The typing rules

```
    x:T ∈ Γ
   ──────────  (T-Var)
    Γ ⊢ x : T


    Γ, x:T₁ ⊢ t : T₂
   ─────────────────────────  (T-Abs)
    Γ ⊢ λx:T₁. t : T₁ → T₂


    Γ ⊢ t₁ : T₁ → T₂     Γ ⊢ t₂ : T₁
   ──────────────────────────────────  (T-App)
              Γ ⊢ t₁ t₂ : T₂


   ─────────────────  (T-True)      ──────────────────  (T-False)
    Γ ⊢ true : Bool                  Γ ⊢ false : Bool


    Γ ⊢ t₁ : Bool    Γ ⊢ t₂ : T    Γ ⊢ t₃ : T
   ────────────────────────────────────────────  (T-If)
        Γ ⊢ if t₁ then t₂ else t₃ : T
```

Six rules. Read each aloud; they are exactly what you would write.

T-If is worth pausing on: **both branches must have the same type**. That is why an
`if` expression has a type at all, and it is why Python — where the branches can have
different types — needs a union type to describe the result.

**A derivation** proves a typing judgement. Build one by hand for
`⊢ (λx:Bool. if x then false else true) true : Bool`; it takes ten minutes and it makes the
rules stop being symbols.

### 2.3 The two theorems

**Progress.** *If `⊢ t : T` (t is closed and well-typed), then either t is a value, or there
exists t' with `t → t'`.*

In words: a well-typed term is never stuck.

**Preservation (subject reduction).** *If `Γ ⊢ t : T` and `t → t'`, then `Γ ⊢ t' : T`.*

In words: evaluation does not change the type.

**Together: type soundness.** By induction on the number of steps: a well-typed term either
runs forever or reaches a value of the expected type. It never gets stuck.

That is Milner's slogan, made precise. Note what it does not say: nothing about termination
(STLC happens to be strongly normalizing, but that is a separate theorem, and adding recursion
destroys it while preserving soundness).

### 2.4 Proving progress

By induction on the derivation of `⊢ t : T`. The technique is worth internalizing because
every soundness proof has this shape.

First you need **canonical forms** — what values of each type look like:

> **Lemma (canonical forms).**
> (a) If `⊢ v : Bool` then v is `true` or `false`.
> (b) If `⊢ v : T₁ → T₂` then v is `λx:T₁. t` for some t.
>
> *Proof.* By inspection of the typing rules: the only rules concluding `Bool` for a value are
> T-True and T-False; the only rule concluding an arrow type for a value is T-Abs. ∎

Then:

> **Theorem (progress).** If `⊢ t : T` then t is a value or `∃t'. t → t'`.
>
> *Proof.* By induction on the typing derivation.
>
> - **T-Var**: impossible — t is closed, so Γ is empty and `x:T ∈ ∅` cannot hold.
> - **T-Abs, T-True, T-False**: t is a value. Done.
> - **T-App** (`t = t₁ t₂` with `⊢ t₁ : T₁→T₂` and `⊢ t₂ : T₁`):
>   By the IH on t₁, either t₁ steps — then E-App1 applies and t steps — or t₁ is a value.
>   If t₁ is a value, by the IH on t₂, either t₂ steps (E-App2 applies) or t₂ is a value.
>   If both are values, then by canonical forms (b), `t₁ = λx:T₁. t₁₂`, so E-AppAbs applies
>   and t steps. ∎ (this case)
> - **T-If** (`t = if t₁ then t₂ else t₃` with `⊢ t₁ : Bool`):
>   By the IH, t₁ steps (E-If applies) or t₁ is a value. If a value, by canonical forms (a) it
>   is `true` or `false`, so E-IfTrue or E-IfFalse applies. ∎

Notice **where the canonical forms lemma is used**: exactly at the point where the proof needs
to know that a value of function type really is a lambda. That is the step that fails if the
type system is unsound — and it is why adding a rule that lets a non-lambda have arrow type
breaks everything.

### 2.5 Proving preservation

Needs a substitution lemma, which is the technical heart.

> **Lemma (substitution).** If `Γ, x:S ⊢ t : T` and `Γ ⊢ s : S`, then `Γ ⊢ t[x := s] : T`.
>
> *Proof.* By induction on the derivation of `Γ, x:S ⊢ t : T`. The interesting cases:
> - **T-Var**, `t = x`: then `T = S` and `t[x := s] = s`, and `Γ ⊢ s : S` by assumption.
> - **T-Var**, `t = y ≠ x`: `t[x := s] = y`, and `y:T ∈ Γ` since `y ≠ x`.
> - **T-Abs**, `t = λy:T₁. t₁`: by α-renaming assume `y ≠ x` and `y ∉ FV(s)` (L01 §2.3 — the
>   capture condition, now doing work in a *proof*). Then apply the IH to t₁ under
>   `Γ, y:T₁, x:S` and reassemble with T-Abs. ∎

> **Theorem (preservation).** If `Γ ⊢ t : T` and `t → t'`, then `Γ ⊢ t' : T`.
>
> *Proof.* By induction on the typing derivation, with case analysis on the evaluation step.
> The key case is **E-AppAbs**: `t = (λx:T₁. t₁₂) v₂ → t₁₂[x := v₂]`. From T-App we have
> `Γ ⊢ λx:T₁. t₁₂ : T₁ → T₂` and `Γ ⊢ v₂ : T₁`. Inverting T-Abs gives
> `Γ, x:T₁ ⊢ t₁₂ : T₂`. The substitution lemma then gives `Γ ⊢ t₁₂[x := v₂] : T₂`. ∎

Write both proofs out yourself. They are two pages, they are the most transferable thing in
the course, and having done them once you will read every "our type system is sound" claim
knowing exactly what was proved.

### 2.6 What breaks the theorems, and what that teaches

Each of these is a real design decision in a real language:

**Adding `fix` (general recursion).** Progress and preservation still hold — soundness is
preserved. But **strong normalization is lost**: `fix (λx:T. x)` is well-typed and diverges.
So a sound type system does *not* imply termination, and languages that want termination
(Agda, Idris, Coq) must restrict recursion instead.

Conversely, **`fix` cannot be typed in pure STLC.** The Y combinator requires `x x`, which
requires x to have a type that is its own argument type — impossible with finite types. That is
why real languages add `fix`/`letrec` as a primitive (L02 §2.4).

**Adding a downcast.** `(t as T)` that succeeds if the runtime value matches. Preservation
holds; progress *fails*, unless you also add a rule for the failure case. So you either accept
stuckness (unsound) or add a runtime error as a legitimate outcome (sound, with a weaker
guarantee). Java's `ClassCastException` is the second choice.

**Adding null.** If `null : T` for every T, canonical forms breaks immediately: a value of
type `T₁ → T₂` need not be a lambda. Progress fails. This is Tony Hoare's "billion dollar
mistake", stated formally — and it is exactly why `Optional`/`Option`/`T | None` with
mandatory narrowing (SE-511 L06 §2.5) restores soundness.

**Adding `Any`.** If `Any` is compatible with everything in both directions, canonical forms
breaks. Python's gradual system is *deliberately* unsound here (L08 §2.2), and the payoff is
interoperability with untyped code.

**Adding unchecked mutable arrays with covariant subtyping.** Java's array covariance:
`String[]` is a subtype of `Object[]`, so you can store an `Integer` into a `String[]` — which
Java catches at *runtime* with `ArrayStoreException`. This is a known unsoundness, deliberately
accepted in 1995 for expressiveness before generics existed. L05 shows why mutable containers
must be invariant.

**The general lesson**: each unsoundness is a *trade*, made for a reason. Being able to name
the trade — and which theorem it breaks — is the difference between using a type system and
understanding it.

### 2.7 Type checking as an algorithm

The rules are a *specification*; a checker is an algorithm. For STLC the translation is direct
because the rules are **syntax-directed**: exactly one rule applies to each term form, so the
algorithm is structural recursion.

```python
def typecheck(ctx: dict[str, Type], t: Term) -> Type:
    match t:
        case Var(x):
            if x not in ctx: raise UnboundVariable(x, t.span)
            return ctx[x]                                     # T-Var

        case Abs(x, ty, body):                                # T-Abs
            return Arrow(ty, typecheck(ctx | {x: ty}, body))

        case App(f, a):                                       # T-App
            ft = typecheck(ctx, f)
            at = typecheck(ctx, a)
            match ft:
                case Arrow(p, r) if p == at: return r
                case Arrow(p, _): raise TypeMismatch(expected=p, found=at, span=a.span)
                case _:           raise NotAFunction(ft, span=f.span)

        case If(c, th, el):                                   # T-If
            ct = typecheck(ctx, c)
            if ct != Bool: raise TypeMismatch(Bool, ct, c.span)
            tt, et = typecheck(ctx, th), typecheck(ctx, el)
            if tt != et: raise BranchMismatch(tt, et, t.span)
            return tt

        case TrueLit() | FalseLit(): return Bool              # T-True / T-False
```

**Every `raise` corresponds to a missing derivation**, and the *span* is what makes the error
usable (L02 §2.7). Compare this function against your L02 stuck-term catalogue: each
type-preventable stuck case should now be impossible.

Termination: the recursion is on strictly smaller subterms, so it terminates — which is
worth noting because for richer systems (L04's inference, L08's dependent types) termination
of *type checking* is a real question, and for System F with full inference it is undecidable
(CS-621 L10 §2.4).

### 2.8 Erasure and the phase distinction

Types are checked at compile time and — in most languages — **erased** before execution. The
typed and untyped evaluators produce the same results on well-typed programs:

> **Theorem (erasure).** If `t → t'` in the typed semantics, then `erase(t) → erase(t')` in
> the untyped one, and conversely for well-typed terms.

Consequences:

- **Types cost nothing at runtime.** Python's annotations are not erased (they are stored in
  `__annotations__`) but they are equally not *used* at runtime — CPython ignores them
  entirely (SE-511 L06 §2.3).
- **Types cannot be inspected at runtime** in an erased system. Java's generics are erased,
  which is why `List<String>.class` does not exist. Python's are not erased but are unreliable
  (a `list[int]` annotation says nothing about the actual contents).
- **Runtime type information must be added deliberately** if you need it — reified generics,
  tags, or runtime validation (SE-511 L06 §2.7).

The **phase distinction** — compile time versus run time — is a genuine conceptual boundary,
and languages that blur it (dependent types, L08) pay for it with complexity.

## 3. Construction: adding types to your interpreter

Continue the language from L02.

**Stage 1 — write the typing rules.** For every construct you added in L02, by hand, in the
notation. Booleans, integers, arithmetic, comparison, `if`, `let`, functions, sequencing. Do
this *before* writing the checker.

**Stage 2 — the checker.** Implement §2.7 for your full language. Every error carries a span
and reports expected versus found.

**Stage 3 — check against the catalogue.** Take your L02 stuck-term list. For each
type-preventable entry, write a program exhibiting it and verify the checker rejects it with a
good message. For each *non*-preventable entry (division by zero, etc.), verify it still
type-checks and fails at runtime — and be able to say why your type system does not catch it.

**Stage 4 — the soundness property test.** The most valuable test in the whole course:

```python
@given(well_typed_terms())
def test_well_typed_terms_never_get_stuck(t):
    assert typecheck({}, t) is not None
    try:
        eval(t, {})                      # must not raise a *stuck* error
    except (Divergence, RuntimeLimitExceeded):
        pass                             # divergence is allowed
    except StuckError as e:
        pytest.fail(f"well-typed term got stuck: {e}")
```

Generating well-typed terms is the hard part: generate *type-directed*, building a term of a
requested type from the inside out. This is a genuinely instructive exercise and it is how you
gain confidence that your soundness argument is not merely on paper.

**Stage 5 — the proofs.** Write progress and preservation for *your* language, by hand.
Include the canonical forms lemma and the substitution lemma. When a case does not close,
you have found either a bug in your rules or a genuine unsoundness — and either is a valuable
outcome. Record it.

**Stage 6 — break it deliberately.** Add one unsound feature — an unchecked cast, or `null`
with type T for all T. Then:

- Find the case of the progress proof that now fails, precisely.
- Write a well-typed program that gets stuck.
- Watch your stage 4 property test find it automatically.

This is the exercise that makes soundness real rather than ceremonial.

**Stage 7 — erasure.** Verify that your typed and untyped evaluators agree on all well-typed
terms, by differential testing.

## 4. Failure modes

- **Believing "well-typed cannot go wrong" means "cannot crash".** It means "cannot get
  stuck", which is much narrower.
- **Confusing soundness with termination.** Adding `fix` keeps soundness and loses
  termination.
- **Adding a feature without checking the proofs.** `null`, unchecked casts, and covariant
  mutable arrays each break a specific case, and the case is findable.
- **Type rules written after the checker.** Then the rules document the implementation's bugs.
- **No spans in type errors.** Unusable checker.
- **A checker that is not syntax-directed** without realizing it — then you need a search, not
  a recursion, and you have accidentally made checking expensive or undecidable.
- **Assuming type checking terminates.** True for STLC; not automatic.
- **Not testing soundness empirically.** The stage 4 property test is worth more than any
  amount of confidence.
- **Trusting runtime type information in an erased system.**

## 5. Exercises

### Warm-up (30 min)

**W1.** Build the full typing derivation for
`⊢ (λf:Bool→Bool. λx:Bool. f (f x)) (λy:Bool. if y then false else true) true : Bool`.

**W2.** Prove the canonical forms lemma for a language with `Bool`, `Int`, and arrows.

**W3.** Add `null : T` (for all T) to STLC. Identify the exact proof case that fails, and
exhibit a well-typed stuck term.

### Core (3 h)

**C1 — The proofs.** Write progress and preservation for STLC with booleans, in full,
including both lemmas, by hand. Then extend both proofs to cover `if`, `let`, and integer
arithmetic. Note every case explicitly; a proof with a case omitted is not a proof.

**C2 — Types for your language.** Complete §3 stages 1–3. Deliverable: the handwritten typing
rules for every construct, the checker with spans and good messages, and the verification
against your stuck-term catalogue with a program per entry.

**C3 — The soundness property test.** Complete §3 stage 4, including the type-directed term
generator. Deliverable: the generator, the property test, and a report of anything it found.
If it found nothing, report the number of terms tested and the type/term-shape coverage.

**C4 — Break it.** Complete §3 stage 6. Deliverable: the unsound feature, the identified
failing proof case, the well-typed stuck program, and confirmation that the property test
catches it. Then write 400 words on the analogous unsoundness in a real language and why its
designers accepted it.

### Challenge

**X1.** Extend your language with records and record subtyping, and *prove* preservation still
holds. The record case of the substitution lemma and the width/depth subtyping rules are
where the work is. Then add mutable record fields and discover why depth subtyping becomes
unsound (this is L05's variance argument, met by proof rather than by rule).

**X2.** Mechanize the progress and preservation proofs for STLC in Coq or Lean, following
*Software Foundations* (Volume 2, `Stlc.v` and `StlcProp.v`). Report how long it took, what
the mechanization forced you to make precise that the paper proof glossed over, and whether
you would do it again. The answer to the middle question is usually "several things", and
that is the point.

## 6. Self-check

1. State progress and preservation, and say what "go wrong" means precisely.
2. What does type soundness *not* guarantee? Give four things.
3. State the canonical forms lemma and say exactly where the progress proof uses it.
4. Give the key case of the preservation proof and the lemma it needs.
5. Why can `fix` not be typed in STLC, and what happens to the theorems when you add it as a
   primitive?
6. Explain, in terms of canonical forms, why `null : T` breaks progress.
7. What makes the STLC typing rules implementable as a simple recursion?
8. What is erasure, and what does it imply about runtime type information?

## 7. Primary sources

- Pierce, *TAPL*, chs. 8–11. Chapter 8 (typed arithmetic), 9 (STLC), 11 (simple extensions).
  This is the core of the course and the proofs above are his.
- Milner, "A Theory of Type Polymorphism in Programming" (JCSS 1978) — the source of the
  slogan.
- Wright & Felleisen, "A Syntactic Approach to Type Soundness" (1994) — the
  progress/preservation formulation now universally used.
- Pierce et al., *Software Foundations*, Volume 2 — the mechanized versions.
- Hoare, "Null References: The Billion Dollar Mistake" (2009 talk).

---

**Previous:** [L02](L02-operational-semantics.md) · **Next:**
[L04 — Polymorphism and Hindley–Milner Inference](L04-polymorphism-and-inference.md)
