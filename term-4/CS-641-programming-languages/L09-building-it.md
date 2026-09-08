# CS-641 · Lesson 09 — Building It: The Interpreter and Type Checker

**Estimated study time:** 5 hours (plus the build itself)
**Prerequisites:** all of CS-641

---

## 1. Orientation

Eight lessons have grown a language incrementally. This one is about finishing it: turning a
sequence of exercises into something you would show someone, with the engineering that
distinguishes a project from a demo.

The engineering matters as much as the theory here. A language with a beautiful type system
and unusable error messages will not be used, including by you. Nystrom's *Crafting
Interpreters* is right that the craft is most of the work, and this lesson is where the craft
gets its due.

## 2. Theory

### 2.1 The pipeline

```
source text
   │ lexer          → tokens (with spans)
   │ parser         → concrete syntax tree / AST (with spans)
   │ desugarer      → core AST (fewer constructs)
   │ name resolver  → AST with resolved bindings
   │ type checker   → typed AST, or errors
   │ (optimizer)    → simplified AST
   │ evaluator      → value
```

Two decisions shape everything downstream:

**Where is desugaring?** A **core language** with few constructs, plus surface syntax that
desugars into it, means every later pass handles a handful of node types instead of forty.
`let x = e1 in e2` desugars to `(λx. e2) e1`; `if` to a `match`; multi-argument functions to
curried ones; `and`/`or` to conditionals. **Desugar early and aggressively** — it is the single
biggest simplification available, and it is what real compilers do (GHC's Core has ~10 node
types; Haskell's surface syntax has hundreds).

The cost: error messages must be reported in *surface* terms, so desugared nodes must carry
provenance back to the source. Handle that at the desugaring step, not later.

**Where is name resolution?** Either fold it into the type checker (simpler, and fine for a
small language) or make it a separate pass producing an AST where every variable reference
points at its binder (better errors, enables later passes, and is what you want if you add
modules).

### 2.2 Lexing and parsing

**Lexer.** Regular expressions or a hand-written scanner. Emit tokens with spans. Handle:
comments, string escapes, numeric literals in several bases, and — if you want offside rule
layout — indentation tokens.

**Parser.** For a small language, **recursive descent with Pratt parsing for expressions** is
the right answer: it is readable, it gives good errors, and precedence is a table rather than a
grammar restructuring.

```python
def parse_expr(self, min_bp: int = 0) -> Expr:
    left = self.parse_prefix()
    while (op := self.peek_operator()) is not None:
        lbp, rbp = BINDING_POWER[op]
        if lbp < min_bp:
            break
        self.advance()
        right = self.parse_expr(rbp)
        left = BinOp(op, left, right, span=left.span.to(right.span))
    return left
```

Pratt parsing (Vaughan Pratt, 1973; Nystrom's "Simple but Powerful" article is the best modern
explanation) handles precedence, associativity, prefix, infix, and postfix operators in about
forty lines. Left-associative operators use `(lbp, lbp+1)`; right-associative use
`(lbp, lbp)` — one character, and getting it wrong is the classic bug.

**Parser generators** (Lark, ANTLR, tree-sitter) are worth it for large grammars and give you
worse error messages. For a language of this size, hand-written wins.

**Error recovery.** Report one error and stop, or recover and report several? Recovery means
inventing a plausible continuation — typically synchronizing at statement boundaries or
closing delimiters. Users strongly prefer multiple errors; it is real work; decide
deliberately (L02 §2.7).

### 2.3 The AST and spans

```python
@dataclass(frozen=True)
class Span:
    file: str; start: int; end: int; line: int; col: int

@dataclass(frozen=True)
class Node:
    span: Span
```

**Every node carries a span, from the lexer onward.** This is not optional and it cannot be
retrofitted cheaply. Every error message, every warning, every IDE feature depends on it.

Design questions:

- **Immutable AST** (a new tree per pass) or **mutable annotations** (fields filled in by later
  passes)? Immutable is cleaner and allocates more; a typed AST as a *separate* type from the
  untyped one gives you compile-time assurance that the checker ran (L06's illegal-states
  argument applied to your own compiler).
- **Visitor or `match`?** In Python, `match` (L06 §2.4). Use a generic traversal helper for
  passes that touch few node types.
- **Interning identifiers** — replace strings with integers after name resolution. Faster
  comparison and hashing (PY-602 L04 §2.7's ladder), and it makes shadowing explicit.

### 2.4 Error messages, properly

The single highest-value engineering investment. The target:

```
error[E0308]: type mismatch
  ┌─ example.lang:12:16
   │
12 │     if compute(x) then 1 else 2
   │        ^^^^^^^^^^ expected `Bool`, found `Int`
   │
   = note: the condition of `if` must be a `Bool`
   = note: `compute` is declared at example.lang:4:1 with type `Int -> Int`
help: did you mean to compare the result?
   │     if compute(x) > 0 then 1 else 2
```

The components, each of which is a separate piece of work:

1. **A span**, from the AST.
2. **The source line, rendered**, with the span underlined. (A `SourceMap` that maps offsets to
   line/column and retrieves lines.)
3. **Expected versus found**, in the user's vocabulary.
4. **A secondary span** for the related location — where the conflicting type came from. This
   is the thing that turns "type mismatch" into "I see what happened".
5. **A note** explaining the rule.
6. **A suggestion**, when confident.
7. **An error code**, so it is searchable.

`rustc` and Elm set the standard here and both teams have written about it; both treat error
messages as a primary product feature rather than an afterthought, and it shows.

**Type errors specifically** are hard in an inference-based system (L04 §2.8): unification
fails at a point that may be far from the mistake. Mitigations: track the *provenance* of every
constraint (which expression generated it), report both endpoints, and prefer to report at the
earlier one.

### 2.5 Testing a language implementation

Six kinds, and you want all of them:

**Unit tests per pass.** Lexer on tricky input; parser on precedence and associativity; type
checker on each rule.

**Golden/snapshot tests for errors.** Every error message, checked against a stored expected
output. Error messages regress silently otherwise, and they are the part users see most.

**Property tests** (SE-511 L04):
- Round-trip: `parse(pretty_print(ast)) == ast`. Finds precedence and parenthesization bugs
  immediately.
- Determinism: same program, same result.
- Substitution/environment agreement (L02 §2.6).
- Type preservation: if `t : T` and `t → t'` then `t' : T` — checked empirically on generated
  terms.
- **Soundness**: well-typed terms never get stuck (L03 §3 stage 4). The most valuable one.

**Differential testing.** If a subset of your language matches an existing one, compare.

**A test-program corpus.** A directory of `.lang` files, each with its expected output or
expected error, run as a suite. This is how every real language implementation is tested, and
it scales better than unit tests.

**Fuzzing.** Generate random token streams and random ASTs; the implementation must never
crash with an internal error — it may reject, but not with a traceback. `atheris` (SE-511 L03
§2.5) finds parser and checker crashes quickly.

### 2.6 Performance, briefly

A tree-walking interpreter is 10–100× slower than a bytecode VM. If you want speed:

- **Bytecode compilation.** Flatten the AST to a linear instruction stream. Removes the
  pointer chasing (PY-602 L06) and the per-node dispatch.
- **Environment as an array, not a dict.** Resolve variables to (depth, index) at name
  resolution time, so lookup is two array indexes instead of a hash. This is exactly what
  CPython's `LOAD_FAST` does (PY-501 L08 §2.2), and it is the single biggest win for a
  tree-walker too.
- **Interning and small-value caching.** As CPython does (PY-602 L04 §2.2).
- **Avoid allocation per node visit.** Reuse value representations where possible.

But: **measure first** (PY-602 L01). For a teaching language, clarity beats speed, and the
right answer is usually "it is fast enough and here is the measurement".

### 2.7 What to include, and what to leave out

Scope control is the difference between finishing and not. A defensible scope for this
artifact:

**In:**
- Integers, booleans, strings, unit.
- Functions (curried), application, `let`, `letrec`/`fix`.
- `if`, comparison, arithmetic.
- Records with field access.
- Algebraic data types with constructors and `match` (L06).
- HM type inference with let-polymorphism (L04).
- Exhaustiveness checking (L06).
- Good errors with spans.
- A REPL.

**Out — and say so explicitly in your write-up:**
- Modules and separate compilation (a large amount of work, mostly bookkeeping).
- Type classes / ad-hoc polymorphism (L04 X2 if you want it).
- Subtyping (L05 — it conflicts with full inference; pick one).
- Mutation and effects (L07 — adds a store to every rule).
- Optimization.
- A standard library beyond a handful of primitives.

**Stating what you left out, and why, is part of the deliverable.** A scoped project with a
clear boundary is a stronger artifact than a sprawling one that does nothing well — the same
argument as PY-502 L10.

### 2.8 The write-up

The artifact is the language *and the document*. The document should contain:

1. **The grammar**, in BNF or EBNF.
2. **The operational semantics**, as inference rules (L02).
3. **The typing rules**, as inference rules (L03–L06).
4. **A statement of what your type system guarantees** — and, honestly, what it does not
   (L08 §2.7's five questions, answered for *your* language).
5. **Design decisions with alternatives**: why curried functions, why let-polymorphism and not
   System F, why erased types, why this error-reporting strategy. Rejected options with reasons
   (SE-521 L10 §2.5's ADR discipline).
6. **What you left out and why.**
7. **The test strategy**, with the soundness property test called out.
8. **Known limitations**, honestly.

That document is what makes this a graduate artifact rather than an exercise, and it is the
part that will still be useful to you in five years.

## 3. Construction: finishing the language

**Stage 1 — consolidate.** You have eight lessons of accumulated code. Restructure it:
`lexer.py`, `parser.py`, `ast.py`, `desugar.py`, `resolve.py`, `types.py`, `infer.py`,
`patterns.py`, `eval.py`, `errors.py`, `repl.py`, `main.py`. Every module with a stated
responsibility (SE-521 L01 §2.2's one-sentence secret).

**Stage 2 — the core language.** Define a minimal core and desugar everything into it. Report
the node-type count before and after — a 3:1 reduction is typical and it makes every later pass
smaller. Ensure spans survive desugaring, with provenance back to surface syntax.

**Stage 3 — name resolution as its own pass.** Resolve every variable to its binder. Report
unbound variables, shadowing (as a warning, if you want one), and unused bindings. Then use
the resolution to replace name lookups with (depth, index) in the evaluator, and measure the
speedup.

**Stage 4 — errors, properly.** Implement §2.4's seven components. Build a `SourceMap`, a
diagnostic renderer with underlining, and secondary spans. Then rewrite your *twenty* most
common errors to use them. Golden-test all twenty.

**Stage 5 — the test corpus.** A directory structure:

```
tests/
  pass/       *.lang + *.expected     (program and its output)
  fail/       *.lang + *.expected     (program and its expected error)
  panic/      *.lang                  (must reject, must not crash internally)
```

with a runner that discovers and executes them. Aim for 100+ programs. This is how you will
actually develop from here.

**Stage 6 — the property suite.** All five properties of §2.5, including the type-directed
generator for well-typed terms and the soundness property. Report the number of terms tested
and anything found.

**Stage 7 — fuzz it.** Random token streams and random ASTs. Fix every internal crash. Report
the count found — there will be several, and they are all real bugs.

**Stage 8 — the REPL.** Multi-line input, `:type expr` showing the inferred type, `:ast`,
`:trace` showing small steps, `:env`, history, and — if you are ambitious — completion. A good
REPL is what makes a language pleasant, and it takes an afternoon.

**Stage 9 — performance, measured.** Benchmark the tree-walker on a few programs. Then
implement *one* optimization (the array environment from stage 3 is the best candidate) and
measure. Report the speedup and stop — this is not a performance course, and the point is to
know the number.

**Stage 10 — the write-up.** §2.8, all eight sections. This is the deliverable.

**Stage 11 — show it to someone.** Have a colleague write a program in your language, without
your help, from your documentation. Record every place they got stuck. That list is worth more
than any amount of self-review, and it is the closest thing to a real user study you can do in
an afternoon.

## 4. Failure modes

- **No spans from the start.** Retrofitting is miserable; every error is unusable until you do.
- **No core language.** Every pass handles forty node types.
- **Desugaring that loses provenance**, so errors point at generated code.
- **Error messages as an afterthought.** The most-used feature, least invested in.
- **No golden tests for errors.** They regress silently.
- **No test corpus.** Unit tests do not scale to a language.
- **Scope creep.** Modules, type classes, and effects each triple the work. Say no, in writing.
- **Optimizing before measuring.** PY-602 L01.
- **No soundness property test**, so the type system's central claim is untested.
- **A parser generator for a small grammar**, trading readability and error quality for
  nothing.
- **Left/right associativity confused in Pratt binding powers.** One character; test it.
- **Not writing the rules down.** Then the implementation *is* the specification, bugs
  included.

## 5. Exercises

### Warm-up (30 min)

**W1.** Implement Pratt parsing for arithmetic with `+ - * / ^` and unary minus, with correct
precedence and associativity. Test with expressions whose parse is ambiguous to the eye.

**W2.** Take one existing error message from your implementation and upgrade it to §2.4's seven
components. Report the before and after.

**W3.** Write a round-trip property test: `parse(pretty_print(ast)) == ast` over generated
ASTs. Report what it found.

### Core (4+ h — this is the term artifact)

**C1 — The complete language.** Complete §3 stages 1–9. Deliverable: the restructured
implementation, the core-language reduction with node counts, name resolution with the measured
speedup, twenty upgraded error messages with golden tests, the 100+ program corpus, the
property suite with the soundness test, the fuzzing results, the REPL, and the performance
measurement.

**C2 — The write-up.** Complete §3 stage 10. All eight sections. The honest statement of what
your type system does *not* guarantee (§2.8 item 4) is the assessed part.

**C3 — The user study.** Complete §3 stage 11. Deliverable: the list of everything your
colleague got stuck on, and what you changed as a result. Report at least three changes.

**C4 — The retrospective.** 800 words: what you would do differently if you started again;
which theoretical result from this course turned out to matter most in the implementation; and
which piece of theory you understood only after implementing it. That last question is the one
worth answering carefully.

### Challenge

**X1.** Add one significant feature from the "out" list of §2.7 — modules with separate
compilation, type classes with dictionary passing, subtyping with local inference, or mutable
references with the value restriction. Write the rules first, prove the soundness cases that
change, implement, and test. Report what it cost in lines and in complexity across every pass.

**X2.** Compile your language to a bytecode VM instead of tree-walking: a compiler pass to a
linear instruction stream, a stack machine with an explicit frame stack (removing the host
recursion limit, L02 X1), and proper tail calls. Benchmark against the tree-walker. Report the
speedup, the increase in implementation size, and — using PY-602 L02 — a profile showing where
the remaining time goes.

## 6. Self-check

1. Give the compiler pipeline and the two decisions that shape everything downstream.
2. What does a core language buy, and what does aggressive desugaring cost?
3. Give the seven components of a good error message.
4. Explain Pratt parsing's binding powers and the associativity encoding.
5. Give six kinds of test for a language implementation and say which is most valuable.
6. What is the single biggest performance win for a tree-walking interpreter, and which
   CPython feature is its analogue?
7. Name four things you deliberately left out of your language and give a reason for each.
8. What eight sections does the write-up need, and which is the one people omit?

## 7. Primary sources

- **Nystrom, *Crafting Interpreters* (2021, free online).** Part II for the tree-walker,
  part III for the bytecode VM. The best practical book on this subject.
- Nystrom, "Pratt Parsers: Expression Parsing Made Easy" (2011).
- Pierce, *TAPL*, ch. 4 and the implementation chapters.
- Appel, *Modern Compiler Implementation in ML* — for structure at a larger scale.
- The `rustc` dev guide's diagnostics chapter, and Elm's "Compiler Errors for Humans" (2015).
- Cooper & Torczon, *Engineering a Compiler*, 3rd ed. — if you continue into compilation.

---

**Previous:** [L08](L08-type-systems-in-practice.md) ·
**Course complete.** Next: [problem sets](problem-sets.md), [exam](exam.md), and
[Term 5](../../term-5/DS-701-distributed-systems/syllabus.md).
