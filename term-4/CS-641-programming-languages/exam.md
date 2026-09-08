# CS-641 — Written Examination

**Time allowed: 3 hours. Closed book, no interpreter.**
**Answer FOUR from Section A and ONE from Section B.**
Section A: 15 marks each. Section B: 40 marks.

Inference rules must be written in the standard notation. Proofs must cover every case.

---

## Section A — answer FOUR

### The lambda calculus (L01)

**A1.** Give the three-line grammar of the untyped lambda calculus and the three concrete-syntax
conventions. Define free and bound variables formally, and state the substitution rules including
the side condition, explaining exactly what it prevents. *(15)*

**A2.** Give three implementation strategies for avoiding variable capture, with the cost of each,
and say which real interpreters use. Then explain how evaluation order is encoded in the choice of
metavariables in a small-step rule. *(15)*

**A3.** Distinguish call-by-value, call-by-name, call-by-need and normal order, giving a term that
separates CBV from normal order. State the Church–Rosser theorem and its consequence, and explain
the Church encoding pattern connecting it to a design pattern you already use. *(15)*

### Operational semantics (L02)

**A4.** Distinguish operational, denotational and axiomatic semantics and say what each is for.
Distinguish small-step from big-step and explain why soundness proofs need small-step. *(15)*

**A5.** Define a stuck term and state the relationship between stuckness and type systems. Write the
big-step application rule with environments and closures, and identify the single word whose change
turns lexical scoping into dynamic scoping. *(15)*

**A6.** Explain why adding mutable references changes every rule in the language, and what that
tells you about effects. Give six properties you can test about an interpreter, state what
infrastructure a good error message requires and when it must be added, and explain why a
recursive-descent interpreter limits interpreted recursion depth with the fix. *(15)*

### The simply-typed lambda calculus (L03)

**A7.** State progress and preservation precisely, and say what "go wrong" means in this context.
Then give four things type soundness does *not* guarantee. *(15)*

**A8.** State the canonical forms lemma and identify exactly where the progress proof uses it and
why the proof fails without it. Give the key case of the preservation proof and the lemma it
requires. *(15)*

**A9.** Explain why `fix` cannot be typed in STLC and what happens to the theorems when it is added
as a primitive. Explain, in terms of canonical forms, why `null : T` breaks progress. Then say what
makes the STLC typing rules implementable as a simple recursion, and what erasure is and what it
implies about runtime type information. *(15)*

### Polymorphism and inference (L04)

**A10.** Name four kinds of polymorphism with an example of each. Explain why System F inference is
undecidable and state the two restrictions Hindley–Milner imposes to recover decidability. *(15)*

**A11.** Explain the `let`/lambda asymmetry with a concrete pair of programs, and say why it exists.
Define a principal type and explain why principality matters. *(15)*

**A12.** State what the occurs check prevents and which famous term it rejects, and what that implies
about typing the Y combinator. State the generalization condition and demonstrate with a program
what goes wrong without it. Then state the value restriction and the general lesson it teaches about
mutation and polymorphism. *(15)*

### Subtyping and variance (L05)

**A13.** State the substitution principle and the subsumption rule. Derive the function subtyping
rule from first principles, explaining both premises. *(15)*

**A14.** Give the input/output rule for variance and apply it to five generic types. Then prove that
mutable containers must be invariant. *(15)*

**A15.** Explain Java's array covariance, what it costs at runtime, and why Python did not repeat the
mistake. Distinguish top, bottom and `Any`, say what a bound in `∀α <: T` buys over simply taking a
`T`, and explain why subtyping is hard to combine with HM inference and what Python's answer is.
*(15)*

### Algebraic data types and the expression problem (L06)

**A16.** Define product and sum types and give the counting algebra including the identity elements.
Show how sum types make illegal states unrepresentable, using the state-count argument. *(15)*

**A17.** State the expression problem with all four of Wadler's requirements. Give the two-by-two
table of representations and say which suits which direction of change. *(15)*

**A18.** Explain what the visitor pattern does to the expression problem's trade-off and what it does
not do. Name three genuine solutions with their costs, explain why exhaustiveness is required by the
typing rule rather than merely desirable, and say what GADTs buy for an interpreter and why Python
cannot express them. *(15)*

### Effects and monads (L07)

**A19.** State what purity buys, with five concrete consequences. Explain why laziness forced Haskell
to be pure. *(15)*

**A20.** Give the three parts of a monad and the three laws, saying which law licenses which
refactoring. Name six monads you already use without calling them that, and identify the operation
that is `bind` in each. *(15)*

**A21.** Give three things the monad abstraction buys and three of its limitations. Compare `Result`
with exceptions across six dimensions and say when each is right in Python. Then state the colouring
problem, give the five responses, and name the best one for Python — explaining why generators are
effect handlers and what Python's cannot do. *(15)*

### Type systems in practice (L08)

**A22.** Name the four axes along which type systems vary and explain the tensions between them.
Define consistency and the gradual guarantee, and say why non-transitivity is essential rather than a
defect. *(15)*

**A23.** Explain why practical gradual type systems are unsound and what the measured cost of
soundness turned out to be. State the Curry–Howard correspondence with five rows of the
correspondence table. *(15)*

**A24.** State what refinement types buy over full dependent types and what makes them feasible.
State what affine types give Rust and which PY-601 discipline they encode. Then give the five
questions for reading any type system, and state precisely what a green `mypy --strict` run licenses
you to believe. *(15)*

### Building it (L09)

**A25.** Give the compiler pipeline and identify the two decisions that shape everything downstream.
State what a core language buys and what aggressive desugaring costs. *(15)*

**A26.** Give the seven components of a good error message, with an example message that has all
seven. Explain Pratt parsing's binding powers and how associativity is encoded. *(15)*

**A27.** Give six kinds of test for a language implementation and say which is most valuable and
why. State the single biggest performance win available to a tree-walking interpreter, name four
things you would deliberately leave out of a small language with a reason for each, and identify the
section of an implementation write-up that people omit. *(15)*

### Synthesis across the course

**A28.** Trace one idea — the relationship between a typing rule and an evaluation rule — through
STLC (L03), HM (L04), subtyping (L05) and gradual typing (L08). At each stage state what the type
system rules out, what it newly permits, and what the proof obligation becomes. *(15)*

**A29.** "Well-typed programs cannot go wrong" is a theorem about a specific language. Explain what
the theorem actually says, what its hypotheses are, and then explain — for each of `null`, array
covariance, `Any`, and unchecked casts — which hypothesis is violated and what the practical
consequence is. *(15)*

**A30.** Explain the relationship between the expression problem (L06), the visitor pattern (and
SE-521 L04's treatment of it), and the choice between a Protocol and an ABC (SE-511 L07). All three
are the same question asked in three vocabularies — state the question, and say what each vocabulary
makes easy to see that the others obscure. *(15)*

---

## Section B — answer ONE

**B1. (40)** Design a type system for a small domain-specific language used to describe data
pipelines: stages that read, transform, and write typed records, composed with an operator that
requires the output schema of one stage to match the input schema of the next.

Your answer must include:

- The types, in a grammar.
- The typing rules for stage composition, transformation, and the base stages, in inference-rule
  notation.
- Whether you chose nominal or structural typing for record schemas, with the argument.
- Whether you chose subtyping (allowing a stage that produces extra fields to feed a stage
  requiring fewer), and what that costs you in inference.
- The inference story: full HM, local, or full annotation — with the reasoning.
- A statement of what your system guarantees, and at least three failures it does *not* catch.
- The error message you would produce for the most common mistake (a schema mismatch in a long
  pipeline), with all seven components.
- What you deliberately left out.

Marks are for the coherence of the design decisions and for the honesty of the guarantee.

**B2. (40)** You are asked to review a proposal to migrate a 300,000-line Python codebase to
strict typing. The proposer claims it will "eliminate a whole class of bugs" and proposes a
six-month project to annotate everything.

Write the review. It must include:

- What the proposal would and would not actually catch, with specific bug classes, and evidence
  for each claim.
- What "strict" means precisely, and which of `mypy --strict`'s flags carry most of the value.
- The unsoundnesses that will remain, and a program demonstrating each.
- What measurement you would take *first* to size the benefit, and what number would change the
  recommendation.
- Why annotating everything at once is the wrong strategy, and what the right one is.
- The role of runtime validation, and why annotations do not replace it.
- What the proposer is right about, stated fairly.
- Your recommendation, with a smaller first step and the criterion for continuing.

**B3. (40)** Argue for or against:

> "The empirical case for static type systems is far weaker than the profession believes. The
> studies are small, confounded, and mixed. The defects types catch are the cheap ones — caught
> by the first test run — while the expensive defects are logical and semantic, and no
> mainstream type system touches them. Type systems are a tooling and refactoring convenience
> sold as a correctness technology."

Required: state the opposing position at its strongest; distinguish clearly between the claims
that are empirical (defect rates) and those that are analytical (what a sound system proves);
give at least four specific pieces of evidence from this course, including at least one
*measurement* you know how to take; address the refactoring argument separately, since it is
the strongest one and is not about correctness; concede where the position is right; and end
with a falsifiable claim about what evidence would change your mind.

Even-handedness is assessed, and the empirical literature genuinely is thin. An answer that
overclaims for types cannot score above 24.

---

**B4. (40)** A team has built an internal rules engine. Rules are written in a small textual
language, parsed into an AST, and interpreted. It has been in production for two years. Symptoms:
roughly one incident per quarter is caused by a rule that "looked right" but did something else;
errors point at the wrong line about a third of the time; adding a new operator requires edits in
eleven places; and a rule that loops forever takes the whole service down.

Write the assessment and the plan. Your answer must: identify which of the four symptoms is a
*language design* problem, which is an *implementation* problem, and which is both, with reasoning;
explain the eleven-places problem in terms of the expression problem, state which representation
they have chosen and which direction of change it makes expensive, and give the fix; explain what
kind of type system would eliminate the "looked right but did something else" class, being specific
about which errors it would and would not catch — and state honestly what fraction of the incidents
it would have prevented, given that you cannot know; state what infrastructure the error messages
need and why it must be added at parse time rather than retrofitted; address the non-termination
problem, including whether you can decide it statically and what that means for the design; and give
the sequence of changes with the reasoning for the order.

Marks are for correctly separating the design problems from the implementation problems, and for the
honesty of the type-system claim.

**B5. (40)** You are designing a configuration and policy language for an infrastructure platform.
Users write policies; the platform evaluates them. Requirements: policies must always terminate;
errors must be caught before deployment, not at evaluation time; the language must be extensible
with new predicates by the platform team without breaking existing policies; users are competent
engineers but not language experts; and a policy that evaluates incorrectly is a security incident.

Design the language and defend it. Your answer must: state the evaluation strategy and justify it;
state how termination is guaranteed and what expressiveness you gave up to get it — naming the
result that says you had to give up something; design the type system, stating for each construct
the typing rule, and say what class of error it eliminates; explain how the extensibility requirement
interacts with the expression problem, and what your representation choice makes cheap and
expensive; state what your soundness theorem would say and what it would *not* say, given the
security requirement; address the error messages specifically, since the requirement that errors are
caught before deployment is only useful if they can be acted on; and state which two features
competent users will ask for that you will refuse, and the argument for each refusal.

Then state the one design decision most likely to be wrong, and the evidence after six months of use
that would tell you so.

---

## Marking guidance

Section A per question: 6 for the standard correct answer, complete; 4 for precision — every
case, correct rule notation, correct side conditions; 3 for an example not drawn from the
lessons; 2 for a stated limitation of your own answer or a connection to another course.

An inference rule with a missing premise, or a proof with a missing case, scores as incomplete.

Section B: a correct, complete answer is 24/40. The remainder is judgement, the quality of the
design reasoning, and honesty about what your system does not guarantee.
