# CS-641 · Lesson 02 — Operational Semantics and Interpreters

**Estimated study time:** 4.5 hours
**Prerequisites:** L01

---

## 1. Orientation

"What does this program mean?" has three standard answers, and the one you choose determines
what you can prove and what you can build.

- **Operational semantics** — meaning is *how it executes*: a transition relation on program
  states. This is what an interpreter implements, and it is what this course uses.
- **Denotational semantics** — meaning is a *mathematical object* the program denotes.
  Elegant, and heavy machinery for recursion and effects.
- **Axiomatic semantics** — meaning is *what you can prove* about it. Hoare logic; FM-751 L02.

The reason this matters practically: **most languages have no formal semantics at all**, so
"what does this mean?" is answered by "whatever CPython does", and disagreements between
implementations are discovered rather than decided. Python's language reference is unusually
good prose, and prose is still ambiguous — PY-501 L08's exercise on evaluation order guarantees
finds real gaps.

Writing a semantics for your own language forces you to answer questions you would otherwise
discover in production.

## 2. Theory

### 2.1 Small-step semantics

Define a relation `t → t'`: "term t steps to t' in one step." Evaluation is the reflexive
transitive closure `→*`.

For a language of booleans and conditionals:

```
   ─────────────────────────────  (E-IfTrue)
    if true then t₂ else t₃ → t₂

   ─────────────────────────────  (E-IfFalse)
    if false then t₂ else t₃ → t₃

              t₁ → t₁'
   ───────────────────────────────────────────  (E-If)
    if t₁ then t₂ else t₃ → if t₁' then t₂ else t₃
```

Read them together: E-If says evaluate the condition; the other two say what to do once it is
a value. The rules *are* the evaluation strategy.

**Values** are terms that cannot step further and are legitimate results:
`v ::= true | false | λx. t`.

**Stuck terms** are terms that are not values and cannot step: `if (λx. x) then a else b`.
Stuckness is the formal model of a runtime type error, and **the goal of a type system is to
prove that well-typed terms never get stuck** (L03).

Small-step's advantages: it makes evaluation *order* explicit, it models non-termination
naturally (an infinite reduction sequence), and it is what you need for concurrency and for
proving type soundness.

### 2.2 Big-step semantics

Define `t ⇓ v`: "t evaluates to value v."

```
   ────────────  (B-Value)        t₁ ⇓ true    t₂ ⇓ v
      v ⇓ v                    ───────────────────────────  (B-IfTrue)
                                 if t₁ then t₂ else t₃ ⇓ v


    t₁ ⇓ λx. t    t₂ ⇓ v₂    t[x := v₂] ⇓ v
   ──────────────────────────────────────────  (B-App)
                  t₁ t₂ ⇓ v
```

Closer to a recursive interpreter — each rule is a function call. Simpler for many proofs.

Its weakness: **non-termination and stuckness look the same.** A term with no derivation
either loops forever or is stuck, and big-step cannot distinguish them. That is exactly the
distinction a soundness proof needs, which is why L03 uses small-step.

Use big-step to write the interpreter; use small-step to prove things about it. Real
compilers' specifications usually do both.

### 2.3 Environments and closures

Substitution (L01 §2.3) is correct and slow: every β-step rewrites the term. Real
interpreters carry an **environment** instead.

```
ρ ⊢ t ⇓ v          "in environment ρ, t evaluates to v"

   ────────────────  (B-Var)        ──────────────────────────  (B-Abs)
    ρ ⊢ x ⇓ ρ(x)                     ρ ⊢ λx. t ⇓ ⟨λx. t, ρ⟩


    ρ ⊢ t₁ ⇓ ⟨λx. t, ρ'⟩    ρ ⊢ t₂ ⇓ v₂    ρ'[x ↦ v₂] ⊢ t ⇓ v
   ──────────────────────────────────────────────────────────────  (B-App)
                          ρ ⊢ t₁ t₂ ⇓ v
```

The key is B-Abs: a lambda evaluates to a **closure** `⟨λx. t, ρ⟩` — the code *and* the
environment it was created in. And B-App extends `ρ'` (the closure's environment), **not**
`ρ` (the caller's).

That one detail is the difference between **lexical** and **dynamic** scoping. Using the
caller's environment gives dynamic scope, where a function sees the caller's variables. Almost
every modern language chose lexical, because dynamic scope makes a function's meaning depend
on its caller — you cannot read a function and know what it does.

This is exactly PY-501 L07: `__closure__` holds the captured cells; the LEGB rule is lexical
scoping; and Python's oddities (class bodies not being closure scopes, `exec` not binding) are
deviations you can now name precisely.

Python has one surviving piece of dynamic scoping: `contextvars` and thread-locals
(PY-502 L07 §2.7) are *deliberately* dynamically scoped, which is why they are convenient for
ambient context and dangerous as a substitute for parameters.

### 2.4 Adding features, one rule at a time

The method: for every construct, give its **evaluation rules** (now) and its **typing rules**
(L03). The discipline of writing both is what stops a language from acquiring features whose
interactions nobody has thought about.

**Let.**

```
   ─────────────────────────────  (E-LetV)         t₁ → t₁'
    let x = v in t → t[x := v]         ──────────────────────────────  (E-Let)
                                        let x = t₁ in t → let x = t₁' in t
```

`let x = t₁ in t₂` is *almost* `(λx. t₂) t₁` — and the difference will matter enormously in
L04, where `let` gets polymorphic typing and the application does not.

**Recursion.** Either a `letrec` construct with a rule that binds the name in its own body, or
a `fix` primitive:

```
   ──────────────────────────────  (E-Fix)
    fix (λx. t) → t[x := fix (λx. t)]
```

`fix` is the Y combinator (L01 §2.6) as a primitive — which is what real languages do, because
the encoding is expensive and untypeable in a simply-typed system (L03 §2.6).

**Mutable references.** These require a **store**, and adding them changes the shape of every
rule:

```
t | μ → t' | μ'        a term and a store step together

   ─────────────────────────────────────  (E-Ref)     ℓ ∉ dom(μ)
    ref v | μ → ℓ | μ[ℓ ↦ v]

   ─────────────────────────  (E-Deref)               ─────────────────────────────  (E-Assign)
    !ℓ | μ → μ(ℓ) | μ                                  ℓ := v | μ → unit | μ[ℓ ↦ v]
```

Notice what mutation costs: **every rule in the language must now thread the store**, because
any subterm might modify it. That is the formal statement of why side effects complicate
reasoning — it is not a matter of taste, it is that the semantics gets an extra component
that every rule must carry. L07 returns to this.

**Exceptions.** Similar structural cost: you need rules propagating an exception out through
every construct, or a separate "abnormal" configuration.

### 2.5 Writing the interpreter from the rules

The correspondence is direct and mechanical, which is the point:

```python
def eval(t: Term, env: Env) -> Value:
    match t:
        case Var(name):
            try:
                return env[name]
            except KeyError:
                raise UnboundVariable(name)              # a stuck term

        case Abs(param, body):
            return Closure(param, body, env)             # B-Abs: capture the env

        case App(fn, arg):
            f = eval(fn, env)                            # B-App premise 1
            a = eval(arg, env)                           # premise 2 (CBV)
            if not isinstance(f, Closure):
                raise NotAFunction(f)                    # stuck
            return eval(f.body, f.env | {f.param: a})    # premise 3: the CLOSURE's env

        case If(cond, then, els):
            c = eval(cond, env)
            if c is TRUE:  return eval(then, env)        # B-IfTrue
            if c is FALSE: return eval(els, env)         # B-IfFalse
            raise NotABoolean(c)                         # stuck

        case Let(name, value, body):
            return eval(body, env | {name: eval(value, env)})
```

**Every `raise` corresponds to a stuck term** — a case the rules do not cover. Listing them is
listing exactly what a type system would have to rule out, and doing that list *before*
writing L03's type system makes the type system's job obvious.

Note `f.env`, not `env`, in the App case. Change it and you have dynamic scoping; write a test
that distinguishes them so you cannot regress.

**A tail-call caveat.** This interpreter uses the host language's stack, so deep recursion in
the *interpreted* language hits Python's recursion limit (PY-501 L07). Real interpreters use
an explicit stack — a CEK machine or a bytecode VM — which also enables tail-call
optimization. L09 addresses this.

### 2.6 Testing an interpreter

An interpreter is a program with an unusually good testing story, and you should exploit it:

- **Property: determinism.** Same term, same environment ⟹ same value. Trivially true if you
  have no mutation, and a useful regression test once you add it.
- **Property: values are irreducible.** `eval(v) == v` for every value.
- **Property: substitution matches environments.** Run the substitution-based and
  environment-based interpreters on the same terms and require agreement. This finds capture
  bugs and scoping bugs immediately.
- **Property: strategies agree where they should.** For strongly normalizing terms, CBV and
  normal order give the same answer (Church–Rosser, L01 §2.5).
- **Differential testing against a reference.** If your language is a subset of Python or
  Scheme, compare against the real thing.
- **Property-based generation of terms.** `hypothesis` generating well-formed ASTs (SE-511
  L04) finds the cases you would not write. Generating *well-typed* terms (after L03) is
  harder and more valuable.
- **Golden tests** for error messages. Error message quality is a real feature and it
  regresses silently.

### 2.7 Errors and messages

The part that separates a toy from something usable. Design decisions, each of which you
should make consciously:

**Source locations everywhere.** Every AST node carries the span it was parsed from. Without
this, every error says "type error" with no position, and the interpreter is unusable. Add it
at the AST stage, not later — retrofitting is miserable.

**Error values versus exceptions.** Does a runtime error abort, or produce an error value that
propagates? Both are defensible; decide and be consistent.

**Message quality.** `TypeError` versus:

```
type error at example.lang:12:8
    if (λx. x) then 1 else 2
        ^^^^^^^^ expected Bool, found a function
    the condition of `if` must be a Bool
```

The second requires: spans, the expected and actual types, and a rendering of the source line.
That is perhaps 100 lines of infrastructure and it changes the experience entirely — the same
argument as PY-502 L09 §2.4 for DSLs.

**Multiple errors.** Report one and stop, or attempt recovery and report several? Recovery is
substantially harder (you must invent a plausible type or AST to continue with) and users
strongly prefer it. Decide, and say why.

## 3. Construction: the interpreter grows

Continue from L01's interpreter. This is the artifact you build through the whole course.

**Stage 1 — write the semantics first.** Before any code, write out the small-step and
big-step rules for: variables, abstraction, application, `if`, `let`, and integer arithmetic.
In the notation. On paper. This is the exercise.

**Stage 2 — environments and closures.** Rewrite L01's substitution-based evaluator to use
environments. Cross-verify against the substitution version on 1,000 random terms — they must
agree, and any disagreement is a real bug in one of them.

**Stage 3 — lexical versus dynamic scope.** Implement both (the one-word change in B-App).
Write a term that distinguishes them and add it as a test. Then write 200 words on why lexical
won, referring to the "you cannot read a function and know what it does" argument.

**Stage 4 — extend the language.** Add, each with its rules written first:

- Integers and arithmetic.
- Booleans and comparison.
- `let`, and `letrec` or `fix`.
- Multi-argument functions (as sugar for currying — write the desugaring rule).
- Sequencing and `unit`.

**Stage 5 — source locations and errors.** Add spans to every AST node, thread them through,
and produce errors in the format of §2.7 with the source line rendered and the span
underlined. Write golden tests for ten distinct error cases.

**Stage 6 — the stuck-term catalogue.** List every way your evaluator can raise. For each: is
it a *type* error (a type system could prevent it) or a *runtime* error that no reasonable
type system prevents (division by zero, array bounds)? That partition is the specification for
L03, and writing it now is the single most useful thing you can do to prepare.

**Stage 7 — mutable references.** Add a store. Notice how many rules change. Report the count
— it is the concrete cost of adding effects to a language, and it is the L07 argument made
tangible.

**Stage 8 — testing.** Implement all six properties of §2.6, with `hypothesis` generating
random well-formed terms.

**Stage 9 — a REPL.** With multi-line input, a `:trace` mode showing each small step, `:ast`
showing the parse, and `:env` showing bindings. A good REPL makes the rest of the course
faster.

## 4. Failure modes

- **Extending the closure's environment with the caller's.** Silent dynamic scoping.
- **Big-step semantics used for a soundness proof.** Cannot distinguish stuck from diverging.
- **No source locations.** Every error is unusable; retrofitting is expensive.
- **Rules written after the code.** Then the rules document the bugs.
- **Forgetting to thread the store** through one rule after adding references. A subtle,
  order-dependent bug.
- **The host stack for the interpreted stack.** Deep interpreted recursion crashes the
  interpreter, and no tail calls.
- **Not distinguishing type errors from runtime errors** in the stuck-term catalogue, so the
  type system's goal is unclear.
- **No differential testing** between the substitution and environment evaluators.
- **Environment as a mutable dict shared between closures.** Every closure sees later
  bindings; this is PY-501 L07 §4's loop-variable capture bug, reproduced in your own
  interpreter.

## 5. Exercises

### Warm-up (30 min)

**W1.** Write the small-step rules for `if` and application, then build the full derivation
tree for one reduction of `if ((λx. x) true) then 1 else 2`.

**W2.** Implement dynamic scoping by changing one word, and write the term that distinguishes
it from lexical.

**W3.** Give three stuck terms in your language, and for each say whether a type system could
prevent it.

### Core (3 h)

**C1 — Semantics first.** Complete §3 stages 1–4. Deliverable: the handwritten rules for every
construct, the environment-based evaluator, the cross-verification against the
substitution-based one, the scoping test, and the desugaring rules for the sugar you added.

**C2 — Errors.** Complete §3 stage 5. Deliverable: spans throughout, ten golden error tests,
and a before/after comparison of a message with and without the infrastructure. Then write
300 words on whether multiple-error recovery is worth implementing for your language.

**C3 — The stuck-term catalogue.** Complete §3 stage 6. Deliverable: the exhaustive list,
partitioned into type-preventable and not, with a one-line justification each. This is your
specification for L03.

**C4 — References.** Complete §3 stage 7. Deliverable: the store-threaded rules, the
implementation, the count of rules that changed, and 400 words on what mutation cost you in
the semantics — and by extension, in reasoning.

### Challenge

**X1.** Implement a CEK machine (Control, Environment, Kontinuation) for your language: an
abstract machine with an explicit continuation stack instead of the host stack. Then add
proper tail calls, and demonstrate that a tail-recursive interpreted loop runs in constant
space where the recursive-descent interpreter overflows. Report the maximum recursion depth
for each.

**X2.** Write the complete formal semantics for a *subset of Python* — say, expressions,
assignment, `if`, `while`, and function definition and call with lexical scoping. Then find at
least two places where the Python Language Reference is ambiguous or where CPython's behaviour
does not obviously follow from the prose (PY-501 L08's evaluation-order exercise is a good
hunting ground). Write up the discrepancies as you would a specification bug report.

## 6. Self-check

1. Distinguish operational, denotational, and axiomatic semantics, and say what each is for.
2. Distinguish small-step from big-step, and say why soundness proofs need small-step.
3. What is a stuck term, and what is the relationship between stuckness and type systems?
4. Write the B-App rule with environments and identify the word that determines lexical versus
   dynamic scoping.
5. Why does adding mutable references change every rule in the language?
6. Give six properties you can test about an interpreter.
7. What infrastructure does a good error message require, and when must you add it?
8. Why does a recursive-descent interpreter limit interpreted recursion depth, and what fixes
   it?

## 7. Primary sources

- Pierce, *TAPL*, chs. 3, 4 (an ML implementation), and 13 (references).
- Nystrom, *Crafting Interpreters*, part II — the best practical treatment of building one,
  including error reporting.
- Felleisen, Findler & Flatt, *Semantics Engineering with PLT Redex* — for going further.
- Krishnamurthi, *PLAI* — environments, closures, and scope, done carefully.
- The Python Language Reference §§4 and 6–8 — read as a semantics document, critically.

---

**Previous:** [L01](L01-lambda-calculus.md) · **Next:**
[L03 — Simply-Typed Lambda Calculus: Progress and Preservation](L03-simply-typed-lambda-calculus.md)
