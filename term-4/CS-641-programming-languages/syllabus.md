# CS-641 — Programming Languages & Type Systems

**Term:** 4 · **Credits:** 15 · **Nominal hours:** 140
**Prerequisites:** PY-501, SE-511 (L06–L07 especially)
**Co-requisite:** CS-621

---

## Driving question

> What does a type actually guarantee?

You have used type annotations for a term (SE-511 L06–L07) and know the practical rules. This
course provides the theory underneath them: what a type system *is*, what soundness means,
why Python's is deliberately unsound, why type inference works, why variance is what it is,
and why some obviously-desirable checks are impossible (CS-621 L10).

The deliverable is an interpreter and type checker for a small language of your own design.
That is not an exercise for its own sake: **implementing a type checker is the only reliable
way to stop treating types as a linting convention and start treating them as a proof
system**, and the perspective transfers directly to how you use `mypy`, design APIs, and
reason about correctness.

## Learning outcomes

On completion you will be able to:

1. **Read and write** inference rules and operational semantics in the standard notation.
2. **Implement** an interpreter from a small-step or big-step semantics, and argue it matches
   the specification.
3. **State and prove** progress and preservation for a simple typed language, and explain what
   "well-typed programs do not go wrong" means precisely.
4. **Implement** Hindley–Milner type inference with unification, and explain what it can and
   cannot infer.
5. **Explain** subtyping, variance, and bounded quantification from the substitution
   principle, and derive Python's variance rules rather than memorizing them.
6. **Use** algebraic data types and exhaustiveness checking, and state the expression problem
   and the trade-off it forces.
7. **Explain** evaluation strategies and effects, and what a monad is without mysticism.
8. **Position** real type systems — Python's gradual system, TypeScript, Rust's, dependent
   types — on the axes of soundness, expressiveness, and inference.

## Lessons

| # | Title | Hours |
|---|---|---|
| L01 | Syntax, Semantics, and the Lambda Calculus | 4.5 |
| L02 | Operational Semantics and Interpreters | 4.5 |
| L03 | Simply-Typed Lambda Calculus: Progress and Preservation | 5 |
| L04 | Polymorphism and Hindley–Milner Inference | 5 |
| L05 | Subtyping, Variance, and Bounded Quantification | 4.5 |
| L06 | Algebraic Data Types and the Expression Problem | 4 |
| L07 | Effects, Evaluation Strategies, and Monads | 4.5 |
| L08 | Type Systems in Practice: Gradual, Dependent, Linear | 4 |
| L09 | Building It: The Interpreter and Type Checker | 5 |

## Assessment

| Component | Weight |
|---|---|
| Problem set 1 (L01–L03): semantics, an interpreter, and a soundness proof | 20% |
| Problem set 2 (L04–L06): inference, subtyping, and ADTs | 20% |
| Problem set 3 (L07–L09): the complete language | 20% |
| Written exam | 25% |
| Course position paper (1,500 words) | 15% |

## Term 4 build artifact

**A small programming language**, implemented end to end:

- A concrete syntax and a parser.
- An abstract syntax tree.
- A formal operational semantics, written out in inference-rule notation.
- An interpreter that implements that semantics.
- A type system, written out as typing rules.
- A type checker with inference (at least let-polymorphism).
- Algebraic data types with exhaustive pattern matching.
- A test suite including a differential test: the type checker never accepts a program that
  the interpreter gets stuck on.
- A written document with the rules, the design decisions, and an honest statement of what
  your type system does *not* guarantee.

Aim for something in the class of "typed lambda calculus with let-polymorphism, ADTs, pattern
matching, and records". That is achievable in the time and is genuinely a language.

## How to study this course

**Learn the notation early and stop being intimidated by it.** Inference rules look forbidding
and are just structured `if`-statements. L01 §2.4 teaches the notation explicitly; after an
hour it reads as easily as code.

**Implement every rule you read.** A rule you have implemented is a rule you understand. The
interpreter grows through the whole course, lesson by lesson.

**Do the proofs.** Progress and preservation are the two proofs that matter, and they are
tractable structural inductions. Writing them once changes how you read every type system
afterwards.

## Required reading

- **Pierce, *Types and Programming Languages* (MIT Press, 2002).** The spine of this course.
  Chapters 3, 5, 8–11, 15, 20, 22–23. It is the best-written textbook in computer science and
  is worth reading beyond what is assigned.
- Nystrom, *Crafting Interpreters* (free online) — for the implementation craft.
- Milner, "A Theory of Type Polymorphism in Programming" (1978).
- Wadler, "Theorems for Free!" (1989).
- Cardelli & Wegner, "On Understanding Types, Data Abstraction, and Polymorphism" (1985).

## Recommended

- Harper, *Practical Foundations for Programming Languages*, 2nd ed. — more rigorous, less
  gentle.
- Krishnamurthi, *Programming Languages: Application and Interpretation* (free) — an
  excellent alternative angle, implementation-first.
- Pierce et al., *Software Foundations* (free) — if you want the proofs mechanized in Coq.
- Abelson & Sussman, *SICP*, chs. 3–4.
- Wadler, "Propositions as Types" (CACM 2015) — read it for pleasure at some point in the
  term.
