# CS-621 · Lesson 10 — Computability, Halting, and Rice's Theorem

**Estimated study time:** 4 hours
**Prerequisites:** L08

---

## 1. Orientation

There is a boundary before complexity: problems that **no algorithm can solve at all**, at
any cost, on any hardware, ever.

This is not a limitation of current technique. Turing proved in 1936 — before there were
computers — that the boundary exists and where it is. The practical consequences are
everywhere in the tools you use daily:

- Why your type checker is *conservative* — it rejects some correct programs, because it
  must (SE-511 L06 §2.2).
- Why static analysis has false positives and false negatives, necessarily (SE-511 L08).
- Why mutation testing cannot detect all equivalent mutants (SE-511 L03 §2.3).
- Why a perfect dead-code eliminator, a perfect linter, a perfect memory-leak detector, and a
  perfect virus scanner are all impossible, not merely difficult.
- Why model checkers bound the state space and theorem provers need human help (FM-751).

Each of these is a direct corollary of Rice's theorem. Knowing that saves you from asking for
tools that cannot exist — and, more usefully, tells you which *approximations* are available.

## 2. Theory

### 2.1 Models of computation and the Church–Turing thesis

**Turing machines.** A tape, a head, a finite state machine. Simple enough to reason about,
powerful enough to compute anything computable.

**Lambda calculus** (Church, 1936). Variables, abstraction, application. Nothing else — no
numbers, no booleans, no data structures; all of those are encoded. CS-641 builds on this.

**μ-recursive functions, register machines, cellular automata, tag systems**, and — this is
the striking part — a great many accidentally-Turing-complete systems: C++ templates,
Magic: The Gathering, PowerPoint animations, Conway's Game of Life, x86 `mov` instructions
alone, and sendmail configuration.

**Church–Turing thesis.** All of these compute exactly the same class of functions, and that
class is what "effectively computable" means. This is not a theorem — it is a claim about the
informal notion of computability — but ninety years of equivalent formalisms have made it
about as well-supported as an unprovable claim can be.

The engineering consequence: **the boundary is not about your language or your hardware.** A
problem undecidable for Turing machines is undecidable for Python, for a quantum computer,
and for anything else.

### 2.2 Countability and the existence of uncomputable functions

Before any specific undecidable problem, a counting argument shows most functions are
uncomputable:

- Programs are finite strings over a finite alphabet, so **the set of programs is countable**.
- The set of functions ℕ → {0,1} is uncountable (Cantor's diagonal argument).
- Therefore **almost all functions have no program.**

This is worth pausing on. Computability is the exception, not the rule. The functions we can
compute are a measure-zero subset of the functions that exist.

### 2.3 The halting problem

**Theorem (Turing, 1936).** There is no program `H(P, x)` that decides, for every program P
and input x, whether P halts on x.

> **Proof.** Suppose H exists. Define:
>
> ```python
> def D(P):
>     if H(P, P):     # does P halt when given its own source?
>         loop_forever()
>     else:
>         return
> ```
>
> Now ask: does `D(D)` halt?
>
> - If `D(D)` halts, then `H(D, D)` returned True, so D took the first branch and loops
>   forever. Contradiction.
> - If `D(D)` does not halt, then `H(D, D)` returned False, so D took the second branch and
>   returns. Contradiction.
>
> Both cases contradict, so H does not exist. ∎

The proof is diagonalization — the same move as Cantor's. Write it out yourself; it is four
lines and it is the most important four lines in the subject.

**What it does not say.** It does not say you cannot determine whether *some particular*
program halts. Many programs are obviously terminating; termination checkers prove it for
large useful classes (this is what makes Idris, Agda, and Rust's `const` evaluation work). It
says no *single* algorithm works for *all* programs.

That distinction is the practically important one, and it is the shape of every consequence
below: **you can always be correct on a restricted class, or conservative in general — never
both complete and sound.**

### 2.4 Reductions and other undecidable problems

Undecidability propagates by reduction, in the same direction as NP-hardness (L08 §2.2): to
show B is undecidable, reduce a known undecidable problem to B.

Undecidable problems worth knowing:

- **Halting.** The archetype.
- **The totality problem** — does P halt on *all* inputs?
- **Program equivalence** — do P and Q compute the same function? (Hence: no perfect
  optimizing compiler, no perfect refactoring verifier, and no way to detect all equivalent
  mutants.)
- **Post's correspondence problem** — a combinatorial puzzle, undecidable, and the source of
  many reductions.
- **Hilbert's 10th** — integer solutions to a polynomial equation (Matiyasevich, 1970).
- **The tiling problem** — can these tiles tile the plane?
- **Type inference for System F** with full polymorphism (Wells, 1994) — which is why real
  languages restrict to Hindley–Milner or require annotations (CS-641 L05).
- **The word problem for groups**, and various questions in formal language theory:
  is a context-free grammar ambiguous? Do two CFGs generate the same language? Both
  undecidable.

### 2.5 Rice's theorem

The generalization that does the real work.

> **Theorem (Rice, 1951).** Let S be any set of computable functions that is neither empty nor
> everything. Then the problem "does program P compute a function in S?" is **undecidable**.

In words: **every non-trivial semantic property of programs is undecidable.**

"Semantic" means: a property of the *function computed*, not of the *syntax*. So:

**Undecidable** (semantic properties):

- Does P ever return 0?
- Does P terminate on all inputs?
- Is P equivalent to Q?
- Does P ever dereference null?
- Does P leak memory?
- Is this code reachable?
- Does P contain a virus (defined behaviourally)?
- Does P satisfy this specification?
- Is this variable ever `None` here?

**Decidable** (syntactic properties):

- Does P contain more than 100 lines?
- Does P call `eval` anywhere in its source?
- Does P have a variable named `x`?
- Is P syntactically valid?

The line between them is exactly the line between what a linter can do perfectly and what it
cannot. And it explains why every real static analyzer is either **unsound** (misses real
bugs), **incomplete** (reports false positives), or **both** — never neither. That is not an
implementation weakness; it is a theorem.

### 2.6 The engineering consequences

Now the payoff. Every practical tool copes with Rice's theorem in one of four ways, and
recognizing which is which tells you what to expect from it:

**1. Be conservative (sound, incomplete).** Reject anything not provably safe. Rust's borrow
checker rejects some correct programs. A type checker rejects some correct programs. Static
analyzers that never miss a bug report false positives. **This is the right default for
safety-critical properties.**

**2. Be optimistic (complete, unsound).** Report only what is certainly a bug; miss others.
Most linters do this, because a high false-positive rate kills adoption (SE-511 L08 §2.1).

**3. Restrict the language.** Make the property decidable by removing expressive power.
Total functional languages (Agda, Idris) guarantee termination by forbidding general
recursion. Regular expressions are decidable because they are not Turing-complete. SQL
without recursion terminates. **Configuration languages that are deliberately not
Turing-complete — Dhall, Starlark, CUE — are this strategy applied to config, and the choice
is deliberate and correct.**

**4. Bound the problem.** Model checkers explore a bounded state space; bounded model checking
looks for bugs within k steps; symbolic execution bounds path length. You get "no bug within
these bounds", which is a real and useful guarantee, just not a total one (FM-751 L03).

**5. Ask for help.** Interactive theorem provers require human-supplied invariants and proofs.
Undecidability means the human cannot be removed in general — but the machine checks the
proof, which is where the value is.

Every tool you use is one of these. Ask which, and you will know what it can promise.

### 2.7 The practical reading

What to actually take from this lesson:

- **Stop asking for impossible tools.** "Can we have a linter that finds all the bugs and
  never false-positives?" No. Not with more effort, not with AI, not ever. The right question
  is which side to err on and by how much.
- **Understand your tools' position.** Is `mypy` sound? No — it is deliberately unsound in
  several places for usability (SE-511 L06 §2.2). Knowing that tells you what a green check
  means.
- **Recognize when restricting is the answer.** If you find yourself needing to analyse an
  arbitrary Turing-complete input, consider whether the input needs to be Turing-complete.
  Config languages, query languages, and rule engines are usually better *not* general, and
  this is why.
- **Bounded verification is real.** "No deadlock in any execution of up to 8 steps with 3
  processes" is a strong statement, obtainable, and far better than nothing.
- **Undecidability is not an excuse.** "It's undecidable" does not mean "we can't do anything
  useful". Every tool in the list above is useful. The theorem constrains what you can
  promise, not what you can build.

### 2.8 Beyond: degrees and the arithmetical hierarchy

Briefly, because it is the natural next question: are all undecidable problems equally hard?
No.

**Turing reducibility** and **degrees** classify them. The halting problem is `Σ₁`-complete;
the totality problem ("halts on all inputs") is `Π₂`-complete and is *strictly harder* — a
machine with a halting oracle still could not decide totality.

The **arithmetical hierarchy** stratifies by quantifier alternation: `Σ₁` is `∃`-statements
about computations ("there exists a halting run"), `Π₁` is `∀` ("all runs halt"), `Σ₂` is
`∃∀`, and so on.

This matters for verification: proving a **safety** property ("nothing bad happens") is `Π₁`
— you must check all executions. Proving a **liveness** property ("something good eventually
happens") is `Π₂` — harder, and it is why liveness is harder to verify and to test
(PY-601 L01 §2.3).

## 3. Construction: undecidability in practice

**Exercise A — the proofs.** On paper:

1. Write the halting problem diagonalization in full.
2. Prove the totality problem undecidable by reduction from halting.
3. Prove program equivalence undecidable by reduction from halting.
4. State Rice's theorem precisely and use it to show "does P ever print anything?" is
   undecidable in one line.
5. Show that "does P halt within 1,000 steps?" **is** decidable, and explain exactly what
   changed.

Item 5 is the important one: it is the bounded-verification idea, and understanding why
bounding restores decidability is what makes model checking sensible.

**Exercise B — classify your tools.** For five tools you use daily (type checker, linter,
test framework, coverage tool, dependency resolver), determine:

- What semantic property is it approximating?
- Is it sound, complete, both, or neither?
- What is its coping strategy from §2.6?
- Construct a program where it is wrong (a false positive or a false negative).

Constructing the counterexample is the exercise. For `mypy` this takes about five minutes and
is genuinely clarifying.

**Exercise C — build a terminator.** Write a program that attempts to decide whether a simple
Python function terminates. Handle: no loops, bounded `for` loops over literals, `while` loops
with a provably decreasing integer variable. Report "terminates", "does not terminate", or
"unknown".

Then:

- Measure what fraction of functions in a real codebase you can classify.
- Construct a function your analyzer says "unknown" for but that obviously terminates.
- Construct one that obviously does not terminate but that your analyzer says "unknown" for.

**The "unknown" category is the whole lesson**: a total function must have one, and its size
is the practical measure of the tool's quality.

**Exercise D — a deliberately non-Turing-complete language.** Design and implement a small
configuration or rule language that is *provably* terminating: no unbounded loops, no
recursion, or recursion only on structurally smaller arguments.

Then:

- Prove that every program in your language terminates (a structural induction; it will be
  short).
- Show a useful thing it can express.
- Show a thing it cannot, and argue whether that is acceptable.
- Compare with a Turing-complete alternative (embedding Python, say) on: expressiveness,
  safety, analysability, and what you can promise a user about a config file they wrote.

This is the practical payoff of the whole lesson, and it is a real design pattern: Dhall,
Starlark, CUE, and Rego all exist because someone made this argument.

**Exercise E — bounded verification.** Take a small concurrent algorithm (a lock protocol, a
state machine). Write an exhaustive checker that explores all interleavings up to k steps and
verifies an invariant. Report: the state space size as a function of k, the largest k you can
check, and whether the invariant held.

Then state precisely what you have and have not proved. That statement is the deliverable,
and it is the same statement a model checker gives you (FM-751 L03).

## 4. Failure modes

- **"It's undecidable, so we can't do anything."** Wrong; every tool in §2.6 is useful.
- **Expecting a sound and complete analyzer.** Impossible. Choose a side.
- **Not knowing which side your tool errs on.** A green `mypy` is not a proof of anything;
  knowing *why* is what makes it useful.
- **Making a configuration language Turing-complete** without deciding to. It happens by
  accident — a template language gains conditionals, then loops, then recursion — and then
  nothing can be analysed.
- **Confusing undecidable with NP-hard.** Different boundaries; NP-hard problems are solvable,
  just possibly slowly.
- **Applying a general impossibility to a restricted case.** Termination of your specific
  function may be entirely provable.
- **Reporting a bounded result as a total one.** "No bug found up to depth 8" is not
  "no bugs".
- **Ignoring the safety/liveness asymmetry.** Liveness is genuinely harder, both to verify
  and to test.

## 5. Exercises

### Warm-up (30 min)

**W1.** Write the halting proof from memory. Then explain in one sentence why the same
argument does not show "does P halt in 1,000 steps?" is undecidable.

**W2.** Use Rice's theorem to show five properties undecidable, in one line each.

**W3.** Construct a Python program on which `mypy` is *unsound* (type-checks, fails at
runtime) and one on which it is *incomplete* (correct, rejected).

### Core (3 h)

**C1 — The proofs.** Complete Exercise A, all five, on paper.

**C2 — Tool classification.** Complete Exercise B for five tools, including a constructed
counterexample for each. Deliverable: the table plus the counterexamples plus a 400-word note
on what each tool's green result actually licenses you to believe.

**C3 — The terminator.** Complete Exercise C. Deliverable: the analyzer, the coverage
statistics on a real codebase, and both constructed counterexamples. Report the fraction of
real functions in the "unknown" bucket and what you would need to shrink it.

**C4 — The bounded language.** Complete Exercise D. Deliverable: the language, the termination
proof, the expressiveness comparison, and a recommendation with the argument you would use to
persuade a team not to embed a general-purpose language in their config.

### Challenge

**X1.** Read Turing (1936), at least §§1–8. Write 1,500 words on what he was actually
proving (the *Entscheidungsproblem*, not "can computers halt"), how the machine model was a
*means* to that end, and what the paper assumes about "effective procedure". Then connect it
to the Church–Turing thesis and explain why it is not a theorem.

**X2.** Implement a bounded model checker for a small concurrent language: explore all
interleavings to depth k, check safety invariants, and report a counterexample trace when one
is found. Test it on a deliberately racy protocol. Then measure the state-space explosion as a
function of processes and depth, implement partial-order reduction to mitigate it, and report
the improvement. This is FM-751 L03 built from scratch, and doing it makes model checking
concrete rather than magical.

## 6. Self-check

1. Give the counting argument that most functions are uncomputable.
2. Write the halting problem proof.
3. State the Church–Turing thesis and say why it is not a theorem.
4. State Rice's theorem and give three undecidable and three decidable program properties.
5. Give the five coping strategies for undecidability, with a real tool for each.
6. Why is "halts within 1,000 steps" decidable, and what does that license?
7. Distinguish undecidable from NP-hard.
8. Why is liveness harder to verify than safety, in terms of the arithmetical hierarchy?

## 7. Primary sources

- Turing, "On Computable Numbers, with an Application to the Entscheidungsproblem"
  (Proc. LMS, 1936). Read §§1–8.
- Rice, "Classes of Recursively Enumerable Sets and Their Decision Problems" (1953).
- Sipser, chs. 3–5. The clearest exposition available.
- Church, "An Unsolvable Problem of Elementary Number Theory" (1936) — the lambda calculus
  version, published a few months before Turing's.
- Hopcroft, Motwani & Ullman, *Introduction to Automata Theory, Languages, and Computation*.
- Wells, "Typability and type checking in System F are equivalent and undecidable" (1999).

---

**Previous:** [L09](L09-coping-with-hardness.md) ·
**Course complete.** Next: [problem sets](problem-sets.md), [exam](exam.md), and
[CS-641](../CS-641-programming-languages/syllabus.md).
