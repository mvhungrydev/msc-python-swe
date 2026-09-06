# SE-511 — Software Construction: Testing, Types, and Tooling

**Term:** 1 · **Credits:** 15 · **Nominal hours:** 125
**Prerequisites:** professional Python experience
**Co-requisite:** PY-501

---

## Driving question

> What makes code trustworthy?

Not "correct" — correctness is a property of a program with respect to a specification, and
most software has no specification. *Trustworthy* is a property of a program **plus the
evidence around it**: tests, types, tooling, review, and the process that produced it. This
course is about constructing that evidence deliberately rather than by habit.

The failure this course is designed to prevent: a codebase with 85% coverage, a green CI
pipeline, full type annotations, and a defect rate nobody can explain. Every artifact is
present and none of them is doing its job. Understanding *what each instrument actually
tells you* — and what it cannot — is the whole of the course.

## Learning outcomes

On completion you will be able to:

1. **Articulate** what a test is evidence *of*, and design a test suite as a set of claims
   rather than a set of function calls.
2. **Choose** correctly between the five kinds of test double, and state what each choice
   couples the test to.
3. **Critique** coverage as an adequacy measure, and apply mutation testing to obtain a
   stronger one.
4. **Write** property-based tests: identify invariants, choose generators, and interpret
   shrunk counterexamples.
5. **Design** a test architecture — what is tested at which level and why — with a defensible
   cost/confidence argument.
6. **Explain** gradual typing: the theory of consistency, what a Python type annotation
   does and does not guarantee, and where the type system is deliberately unsound.
7. **Use** generics, variance, `Protocol`, overloads, `ParamSpec`, and `TypeVar` bounds
   correctly, and diagnose checker disagreements.
8. **Build** custom static analysis (AST-based lint rules) for project-specific invariants.
9. **Produce** reproducible builds: resolve, lock, pin, and audit dependencies; publish a
   package; reason about supply-chain risk.
10. **Argue** for a CI design as an architectural constraint rather than a chore.

## Lessons

| # | Title | Hours |
|---|---|---|
| L01 | Testing as Specification | 3 |
| L02 | Test Doubles and the Boundaries They Imply | 3.5 |
| L03 | Coverage, Adequacy, and Mutation Testing | 3.5 |
| L04 | Property-Based Testing | 4 |
| L05 | Test Architecture: Levels, Contracts, and Cost | 3.5 |
| L06 | Gradual Typing: Theory and Practice | 4 |
| L07 | Generics, Variance, and Protocols | 4 |
| L08 | Static Analysis Beyond Types | 3.5 |
| L09 | Packaging, Dependencies, and Reproducibility | 4 |
| L10 | CI as a Design Constraint | 3 |

## Assessment

| Component | Weight |
|---|---|
| Problem set 1 (L01–L05): a test suite rebuilt from first principles, with mutation score | 20% |
| Problem set 2 (L06–L08): a fully typed library plus a custom lint rule | 20% |
| Problem set 3 (L09–L10): a published, reproducible package with a CI pipeline you designed | 20% |
| Written exam | 25% |
| Course position paper (1,500 words) | 15% |

## Term 1 build artifact

A published Python package. Requirements: solves a real problem you have; full type
annotations passing `mypy --strict`; a test suite with a stated and defended mutation
score; a lockfile and a reproducible build; a CI pipeline; documented public API; a
CHANGELOG; and a README that argues for the design, not just describes the usage.

Ship it to PyPI (or a private index). "Shipped" is part of the assessment: an unpublished
package has not been through the last mile, and the last mile is where most of the learning
is.

## Required reading

- Winters, Manshreck & Wright, *Software Engineering at Google*, chs. 11–14 (testing) and
  21 (dependency management).
- Freeman & Pryce, *Growing Object-Oriented Software, Guided by Tests*, Part I and ch. 20.
- Claessen & Hughes, "QuickCheck" (ICFP 2000).
- PEPs 484, 483, 544, 561, 585, 604, 612, 646, 695, 517, 518, 621, 735.

## Recommended

- Meszaros, *xUnit Test Patterns* — the taxonomy chapters.
- Hillel Wayne, "Property-Based Testing in Python" writings; MacIver's `hypothesis`
  articles on shrinking.
- Siek & Taha, "Gradual Typing for Functional Languages" (2006) — the theory behind PEP 483.
