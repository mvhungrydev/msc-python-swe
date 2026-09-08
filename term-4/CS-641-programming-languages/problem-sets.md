# CS-641 — Problem Sets

The three problem sets build one artifact. Each adds a layer to the same language, so do them
in order and keep the code.

**A note on proofs.** As in CS-621: handwritten first, and a proof with a missing case is an
incomplete proof, not a proof with a small error.

**A note on proofs.** As in CS-621: handwritten first, and a proof with a missing case is an
incomplete proof. Inference rules must be written in the standard notation, and a derivation tree
must be complete.

**A note on the constructions.** The three sets build one artifact — each adds a layer to the same
language — so do them in order and keep the code. Where a part names lesson stages, do those first.

**A note on breaking things.** Several parts ask you to add an unsound feature and find the exact
proof case that fails. Those parts carry the most marks. A type system you have only ever seen work
is a type system you do not understand.

---

## Problem Set 1 — Semantics, an Interpreter, and a Soundness Proof
**Covers L01–L03 · Budget: 18–22 hours**
*Builds on: L01 §3 (all stages), L02 §3 (all stages), L03 §3 stages 1–6*

**Part A — The lambda calculus (L01).** The interpreter through stage 5: parser with the three
concrete-syntax conventions, three substitution strategies cross-verified on 1,000 random
terms, the capture bug as a regression test, three evaluation strategies, and the
CBV-diverges/normal-order-terminates demonstration with reduction counts.

**Part B — Encodings and recursion (L01).** Church booleans, numerals, pairs, lists, with
`plus`, `mult`, `pred`, `is_zero`, and list `map`, all verified by round-tripping through
Python values. Y and Z combinators with factorial. Plus the 400-word explanation of fixed-point
combinators.

**Part C — Inference-rule notation (L01 C4).** The complete small-step semantics for CBV, CBN,
and normal order written out in the notation, with a full hand-built derivation tree for one
term under each strategy.

**Part D — The language (L02).** Stages 1–7: the handwritten rules for every construct, the
environment-based evaluator cross-verified against substitution, the lexical/dynamic scoping
test, integers, booleans, `let`, `letrec`/`fix`, sequencing, spans throughout, ten golden error
tests, the stuck-term catalogue, and mutable references with the count of rules that changed.

**Part E — Types (L03).** Stages 1–4: the handwritten typing rules, the checker with spans and
expected/found messages, verification against the stuck-term catalogue with a program per
entry, and the type-directed generator plus the soundness property test.

**Part F — The proofs (L03).** Progress and preservation for STLC with booleans, in full,
including canonical forms and substitution lemmas, extended to cover `if`, `let`, and
arithmetic. Every case explicit.

**Part G — Break it (L03 stage 6).** One unsound feature added; the exact failing proof case
identified; a well-typed stuck program exhibited; confirmation that the property test catches
it automatically.

**Design note (1,500–2,000 words).** What the capture bug taught you about the side condition.
Where your soundness proof nearly failed to close. What the unsoundness experiment demonstrated
that reading about `null` and array covariance did not. Your stuck-term catalogue's split
between type-preventable and not, and what that says about the reach of type systems.

### Marking emphasis

Proof completeness, and the capture bug. The marks are in the side condition, the cases you nearly
missed, and the regression test that encodes the bug you introduced deliberately.

---

## Problem Set 2 — Inference, Subtyping, and Algebraic Data Types
**Covers L04–L06 · Budget: 18–22 hours**
*Builds on: L04 §3 (all stages), L05 §3 stages 1–6, L06 §3 (all stages)*

**Part A — Hindley–Milner (L04).** Stages 1–4: types and schemes, substitution with correct
composition, unification with the occurs check, Algorithm W, and the six classic terms
inferring correctly. Cross-checked against a reference implementation or against OCaml/Haskell.

**Part B — The generalization bug (L04 stage 5).** The broken version, the unsound program it
accepts, confirmation that your PS1 soundness test catches it, the fix, and the 300-word
explanation of why the condition is necessary.

**Part C — The value restriction (L04 stage 6).** The `ref []` unsoundness reproduced, the
value restriction implemented, and a safe-but-now-rejected program with your judgement on the
trade.

**Part D — Inference error messages (L04 stage 7).** Five realistic type errors, before and
after, with both conflicting locations reported. Plus the 400-word account of why HM error
messages are hard.

**Part E — Subtyping (L05).** Records with width, depth, and permutation; functions with the
correct flip and a test that would catch getting it backwards; covariant mutable containers
implemented deliberately with the runtime failure exhibited, then both responses (runtime check
and invariance) implemented and compared; variance annotations with the position check;
algorithmic subtyping agreeing with the declarative rules on 1,000 random type pairs;
recursive types with the termination fix; top and bottom with exhaustiveness checking.

**Part F — Inference meets subtyping (L05 stage 8).** The documented failure of naive
combination, local type inference implemented, and the 600-word explanation of Python's
annotation requirement — written to convince a colleague who has not taken this course.

**Part G — ADTs (L06).** Sums, products, named data declarations, nested pattern matching with
typed patterns, and exhaustiveness and redundancy checking reporting missing cases *by name*.

**Part H — The expression problem, measured (L06 stage 5).** Both implementations, the four
changes, the diff table, and the 400-word recommendation for three different scenarios.

**Design note (1,500–2,000 words).** The generalization condition and the value restriction:
two places where a small side condition is load-bearing for soundness — what they have in
common. The variance derivation, done from the substitution principle rather than memorized.
What the expression-problem measurement showed, and which representation you would default to.

### Marking emphasis

The unsoundness experiment. Adding an unsound feature and identifying the exact proof case that
fails carries more marks than the inference algorithm working.

---

## Problem Set 3 — The Complete Language
**Covers L07–L09 · Budget: 20–24 hours**
*Builds on: L07 §3 stages 1–6, L08 §3 (all stages), L09 §3 (all stages)*

**Part A — Effects (L07).** The purity audit of your interpreter with the pure/impure ratio;
`Result` in your language with the type checker rewritten to report multiple errors; the State
monad threading the store, compared with explicit threading; the monad laws property-tested for
four monads with a deliberate break demonstrated; generic `sequence`/`traverse`/`map_m` over a
monad Protocol.

**Part B — Effect types (L07 stage 6).** A simple effect system: annotations, checking, five
programs correctly rejected, and an honest report of the annotation burden.

**Part C — Generators as effect handlers (L07 stage 7).** The mechanism with handlers for
Reader, Writer, exceptions, and non-determinism — including the analysis of why Python's
generators cannot resume a continuation multiple times and what would be needed.

**Part D — Type system assessment (L08).** Five languages assessed against the five questions,
with a demonstrating program for every claim. Your own codebase measured: `Any` surface,
`type: ignore` count by code, untyped dependencies, unvalidated boundaries. The one-sentence
statement of what your green run licenses.

**Part E — Gradual typing, both ways (L08 stage 3).** Erased and sound implementations, the
crashing program, the boundary catch, the measured overhead, and the comparison with Takikawa
et al.'s published result.

**Part F — Refinement types (L08 stage 4).** The syntax, verification-condition generation, Z3
discharge, five caught programs, and the annotation-burden report.

**Part G — The comparison table (L08 stage 6).** One buggy program in five type systems, the
table of what each catches at what cost, and the 600-word recommendation for three different
project types.

**Part H — Finish the language (L09).** All eleven stages: restructured implementation, core
language with node counts, name resolution with the measured speedup, twenty upgraded error
messages with golden tests, the 100+ program test corpus, the full property suite, fuzzing
results, the REPL, the performance measurement, **the write-up**, and the user study.

**Design note (1,500–2,000 words).** Which theoretical result mattered most in the
implementation. Which piece of theory you understood only after implementing it. What your
colleague got stuck on in the user study and what you changed. What you left out and why. And
the honest statement of what your type system does not guarantee.

### Marking emphasis

The write-up. A language implementation is judged by its error messages and by the four things you
deliberately left out, each with a reason.

---

## Term 4 build artifact

**The language.** As specified in the syllabus, plus CS-621's contribution: the complexity
analysis of its core algorithms, one component chosen on the basis of a proved bound, and the
statement of which of its analyses are decidable and which are conservative approximations.

Publish it — a repository with a README that lets a stranger install it, run the REPL, and
write a program in ten minutes. The write-up as `DESIGN.md`.

---

## Course position paper (1,500 words)

**Driving question: what does a type actually guarantee?**

Claim, grounds, rebuttal, limits. Grounds from your own work: the soundness proof and the
deliberate unsoundness of PS1 Part G, the `Any`-surface measurement of PS3 Part D, and the
five-system comparison of PS3 Part G.

The rebuttal must state fairly the position that static types are largely a productivity and
tooling feature rather than a correctness one — that the defects they catch are the cheap ones,
that the expensive defects are logical and no mainstream type system catches them, and that the
industry's enthusiasm outruns the evidence — and then answer it. The empirical literature on
types and defect rates is genuinely thin and mixed; an answer that acknowledges that will score
above one that overclaims.

---

## Submission checklist (per set)

- [ ] **Proofs and rules handwritten first**, scanned and committed.
- [ ] Every typing and evaluation rule written *before* the code that implements it.
- [ ] The soundness property test passing, with the number of generated terms reported.
- [ ] Every error message golden-tested.
- [ ] Code in `courses/cs641/` — one growing project, not three.
- [ ] `NOTE.md` — the design note. `DESIGN.md` — the language write-up.
- [ ] `MARK.md` — self-marked, one line per criterion.
- [ ] `log/failures.md` — every proof case that would not close and every rule you got wrong
      the first time.
