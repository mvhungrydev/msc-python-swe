# FM-751 · Lesson 09 — SMT Solvers and Z3 for Engineers

**Estimated study time:** 4.5 hours
**Prerequisites:** L02, L04; CS-621 L08 (NP-completeness)

---

## 1. Orientation

An SMT solver answers one question: **is this formula satisfiable, and if so, give me a satisfying
assignment.**

That sounds narrow and is extraordinarily general, because an enormous range of engineering
questions are satisfiability questions in disguise:

- Can any principal reach this S3 bucket? (CA-731 L03's blast radius — and this is literally how AWS
  answers it.)
- Can a packet from the internet reach this database? (CA-731 L04's reachability.)
- Is there an input that makes this function return a negative balance?
- Can these two access policies ever disagree?
- Does a valid schedule exist for these tasks under these constraints?
- Are these two versions of a function equivalent?
- Is there an assignment of pods to nodes satisfying every affinity rule?

The practical point of this lesson, and it is a strong one:

> **Z3 is a tool a working engineer can pick up in an afternoon and use to answer questions that no
> amount of reading a policy document will settle.** Unlike model checking, it needs no new mental
> model of behaviour over time; you write constraints and ask a question. It is the lowest-barrier
> formal method in the course, and the one you are most likely to use next month.

The theory is also the payoff of CS-621 L08: SAT is the canonical NP-complete problem, and modern
solvers routinely handle instances with millions of variables. **NP-complete does not mean
intractable in practice**, and understanding why is worth having.

## 2. Theory

### 2.1 SAT, and why it works

**SAT**: given a propositional formula, is there an assignment making it true? NP-complete
(CS-621 L08), and worst-case exponential.

Yet modern SAT solvers dispatch industrial instances with millions of variables routinely. The
techniques, in rough order of importance:

- **DPLL**: backtracking search with unit propagation — if a clause has one unassigned literal
  left, that literal is forced.
- **CDCL** (conflict-driven clause learning): on a conflict, analyse *why* it occurred and add a
  learned clause forbidding that cause. This prunes vast regions of the search space and is the
  single innovation that made SAT practical.
- **Non-chronological backtracking**: jump back to the decision that actually caused the conflict
  rather than the most recent one.
- **Activity-based heuristics** (VSIDS): branch on variables involved in recent conflicts.
- **Restarts**: periodically restart the search, keeping learned clauses, to escape bad regions.

The lesson to carry: **worst-case complexity is about adversarial instances; real instances have
structure, and solvers exploit it.** That generalises well beyond SAT, and it is a useful corrective
to reading a complexity class as a verdict.

### 2.2 SMT: SAT modulo theories

Pure SAT speaks only booleans. **SMT** extends it with **theories** that give meaning to symbols:

- **Linear integer/real arithmetic** — `x + 2*y <= 10`. The workhorse.
- **Bitvectors** — fixed-width machine integers with exact overflow semantics. **Essential for
  reasoning about real code**, because `int32` is not ℤ, and the overflow bug in L02's binary search
  is invisible if you model integers as unbounded.
- **Arrays** — with `select` and `store`; models memory and maps.
- **Uninterpreted functions** — a function about which you know nothing except that it is a function
  (`x = y ⇒ f(x) = f(y)`). Useful for abstracting something you do not want to model.
- **Strings and regular expressions** — increasingly capable, and directly applicable to policy and
  URL matching.
- **Datatypes** — algebraic types, useful for structured data.

The architecture: a SAT solver over the boolean structure, plus theory solvers that check whether a
candidate boolean assignment is consistent within their theory, reporting conflicts back as new
clauses (DPLL(T)).

The practical consequence: **stay in the decidable, efficient fragments.** Linear arithmetic is
decidable and fast; non-linear integer arithmetic is undecidable and the solver may not terminate;
quantifiers make things undecidable in general and Z3 uses heuristic instantiation, which is
frequently good enough and occasionally returns `unknown`. Knowing which fragment you are in
explains why one query returns in milliseconds and a superficially similar one runs forever.

### 2.3 The three answers

Z3 returns one of:

- **`sat`** — a satisfying model exists, and you can extract it. If you asked "can an attacker reach
  the database?", `sat` plus the model is the attack path, concretely.
- **`unsat`** — no assignment satisfies the constraints. This is the *proof*: the property holds. To
  verify a property, **assert its negation and hope for `unsat`** — this is the standard idiom and it
  is worth internalising because it inverts the intuition.
- **`unknown`** — the solver gave up: a timeout, an undecidable fragment, or incomplete heuristics.
  Not a proof of anything. Report it as what it is rather than treating it as `unsat`.

Two features that turn a solver into a tool:

- **`unsat` cores**: the minimal subset of assertions that are jointly contradictory. When a
  configuration is unsatisfiable, the core tells you which constraints conflict — that is a
  *diagnosis*, not just a verdict, and it is what makes solver-based tooling usable by people who
  did not write the model.
- **Optimisation** (νZ): `minimize`/`maximize` an objective subject to constraints, turning
  satisfiability into optimisation — scheduling, bin packing, cost minimisation.

### 2.4 Engineering applications

Where this pays for a working engineer, with the shape of the encoding:

**Policy analysis.** Encode access policies as constraints over principal, action, resource and
conditions, then ask: is there a request that policy A permits and policy B denies? Is there any
request from outside the organisation that reaches this resource? **This is exactly how AWS's IAM
Access Analyzer works** — Zelkova encodes IAM policies into SMT and answers reachability questions
soundly, and the CAV 2018 paper describing it is the best available example of formal methods
deployed at scale for ordinary users who have no idea they are using them.

**Network reachability.** Encode forwarding rules, security groups and ACLs as constraints on packet
headers, then ask whether a packet with given properties can traverse from A to B. This is CA-731
L04's reachability analyser, and it is what the cloud providers' own tools do. **It answers
definitively what rule-by-rule inspection answers only probably**, which is the whole value.

**Configuration validation.** Are these constraints jointly satisfiable? Given resource limits,
affinity rules and quotas, does a valid placement exist — and if not, which constraints conflict
(the `unsat` core)?

**Program verification.** Verification conditions from wp calculation (L02 §2.4) are discharged by
an SMT solver. This is what Dafny, Frama-C and ESC/Java do, and it is what you built in L02 Stage 8.

**Test input generation.** Encode a path condition through a program and solve for an input that
takes it — the basis of symbolic execution and whitebox fuzzing (KLEE, SAGE).

**Equivalence checking.** Are these two implementations equivalent for all inputs? Encode both and
assert they differ; `unsat` means equivalent. Genuinely useful after a refactor of a pure function,
and startlingly easy with bitvectors.

**Scheduling and allocation.** Constraints plus an objective; νZ does the rest.

### 2.5 Modelling well

The craft, and the difference between a query that returns in 50 ms and one that runs overnight:

- **Choose the right theory.** Bitvectors for machine arithmetic (and only bitvectors will find the
  overflow); integers for counting; reals only when you mean reals.
- **Avoid quantifiers where possible.** A quantifier-free formula over a decidable theory is fast; a
  quantified one may be `unknown`. Bounded quantification over a small finite domain should be
  *expanded* into a conjunction rather than left as a `ForAll`.
- **Avoid non-linear integer arithmetic.** `x * y` where both are variables is where queries go to
  die. Reformulate, or bound and case-split.
- **Keep the encoding small.** Solver time is superlinear in formula size; encode the question, not
  the world.
- **Use incremental solving** (`push`/`pop`) when asking a series of related questions — the solver
  keeps its learned clauses and the second query is far faster than the first.
- **Set a timeout, always**, and handle `unknown` explicitly.
- **Validate the encoding.** Assert something you know is satisfiable and confirm `sat`; assert
  something you know is contradictory and confirm `unsat`. **An encoding bug that makes everything
  trivially `unsat` looks exactly like a successful proof**, and this is the single most dangerous
  failure mode in solver-based tooling — the direct analogue of L05's vacuity check.

### 2.6 Where it stops

- **Undecidable fragments.** Non-linear integer arithmetic, and general quantified first-order logic
  with theories. The solver may return `unknown` or run forever.
- **Scale.** Large formulas time out. The encoding usually matters more than the solver.
- **`unknown` is not `unsat`.** Treating it as a proof is the worst error available here.
- **It proves things about the model.** Same caveat as L04 §2.5: your encoding of the policy is not
  the policy; your model of the network is not the network. A soundness argument relating the two is
  a separate obligation, and Zelkova's paper is worth reading for how seriously a production system
  takes it.
- **It has no notion of time or behaviour.** It answers questions about states and constraints, not
  about executions. That is TLA+'s job, which is why the two are complementary rather than competing.

## 3. Construction: use Z3 on real problems

Build in `mpse/fm751/l09/`. `pip install z3-solver`.

**Stage 1 — the basics.** Solve five puzzles to get fluent: a Sudoku, the eight queens, a graph
colouring, a small scheduling problem with precedence constraints, and a knapsack with `maximize`.
For each, extract and print the model. Then make one unsatisfiable and print the `unsat` core.

**Stage 2 — validate your encodings.** For each Stage 1 problem, add the two sanity checks from
§2.5: assert something known-satisfiable and confirm `sat`; assert a contradiction and confirm
`unsat`. Then deliberately introduce an encoding bug that makes a problem trivially unsatisfiable,
and observe how it looks identical to a successful proof. **Adopt these checks as a permanent
habit.**

**Stage 3 — bitvectors and real integers.** Encode the L02 binary search overflow: with `Int`, prove
`(lo + hi) / 2` is between `lo` and `hi`; with `BitVec(32)`, find the counterexample. Then verify
that `lo + (hi - lo) / 2` has no counterexample in bitvectors. This is the clearest demonstration in
the course that the theory choice is a modelling decision with consequences.

**Stage 4 — equivalence checking.** Take three pairs of functions that should be equivalent: a
bit-twiddling trick and its obvious version (`x & (x-1)` versus clearing the lowest set bit); a
refactored arithmetic expression; and an optimised branch-free `abs` against the branching one.
Prove or refute equivalence with bitvectors. Then introduce a subtle difference in one and find the
distinguishing input.

**Stage 5 — policy analysis.** Encode a set of access policies — IAM-style, or your CA-731 L03
policy engine's rules — as constraints over (principal, action, resource, conditions). Then answer:
can any principal outside the organisation perform this action? Do these two policies ever disagree,
and on what request? Does adding this statement grant anything new? Compare the answers against what
you would have concluded by reading the policies, and report any case where reading was wrong.

**Stage 6 — network reachability.** Encode your CA-731 L04 VPC — route tables, security groups,
NACLs — as constraints on a packet (source, destination, port, protocol). Then answer: can anything
on the internet reach the database subnet? Which rule permits it? Then remove that rule and confirm
`unsat`. Compare against the configuration-level checker you built in CA-731 L04 Stage 9 and report
what the reachability encoding catches that rule inspection does not.

**Stage 7 — verification conditions.** Extend your L02 Stage 8 verifier: full wp calculation for a
small imperative language, discharged by Z3, with loop invariants supplied as annotations. Verify
five programs, refute three buggy ones with counterexample inputs extracted from the model, and find
one correct program it cannot prove — then explain which invariant it needed.

**Stage 8 — configuration.** Take a real constraint satisfaction problem from your platform: pod
placement under affinity, anti-affinity and resource limits; or capacity allocation under quotas.
Encode it, solve it, and — when it is unsatisfiable — use the `unsat` core to report *which
constraints conflict*. Then add an objective and optimise. That `unsat` core reporting is what turns
this from a solver into a tool someone else can use.

**Stage 9 — symbolic execution, in miniature.** Build a small symbolic executor for a subset of
Python: track path conditions as Z3 constraints, and at each branch fork with the condition and its
negation. At the end of each path, solve for a concrete input that reaches it. Run it on a function
with several branches and generate a test suite achieving full path coverage. Then run it on
something with a loop and observe the path explosion — and implement a bound, reporting what
coverage you achieved within it. This connects L08's generation problem to L09's solving, and it is
how KLEE and SAGE work.

## 4. Failure modes

- **Treating `unknown` as `unsat`.** The worst error available; it is not a proof.
- **No timeout.** The query runs forever and the tool hangs.
- **An encoding bug producing trivial `unsat`.** Looks exactly like success. Run the sanity checks.
- **Integers where bitvectors were needed.** The overflow bug is invisible.
- **Non-linear integer arithmetic.** May never terminate.
- **Unnecessary quantifiers.** `unknown`, where expansion would have given an answer in
  milliseconds.
- **Encoding the world instead of the question.** Slow, and usually unnecessary.
- **No `unsat` core reporting.** "Infeasible" without saying why is not a usable tool.
- **Not using incremental solving** for a series of related queries.
- **Confusing the model with reality.** Your encoding of the policy is not the policy; state the
  soundness argument.

## 5. Exercises

### Warm-up (30 min)

1. Explain why SAT is NP-complete and yet solvers handle millions of variables, naming the key
   innovation.
2. Give six SMT theories and say which one you need to reason about machine arithmetic and why.
3. Explain the three answers, the idiom for proving a property, and why `unknown` is not a proof.

### Core (3.5 h)

4. Complete Stages 1–3: five puzzles with models and a core, the validation habit established with
   the trivial-`unsat` demonstration, and the binary search overflow found with bitvectors.
5. Complete Stage 4 — three equivalence checks with one distinguishing input found.
6. Complete Stage 5 and report any case where reading the policies gave the wrong answer.
7. Complete Stage 6 and report what reachability catches that rule inspection does not.

### Challenge

8. Complete Stages 7–9: the wp verifier with a correct-but-unprovable program explained, the
   configuration solver with `unsat` core diagnosis and optimisation, and the symbolic executor with
   its path explosion bounded.
9. Build a **policy blast-radius analyser** in the spirit of Zelkova: encode a real IAM (or
   Kubernetes RBAC) configuration into SMT, including resource policies, conditions, permission
   boundaries and role-assumption chains. Then answer, soundly: for each principal, what is the set
   of actions it can reach, including via escalation paths (CA-731 L03 Stage 9)? Compare against the
   graph-based analyser you built there: which findings does each produce that the other misses?
   Then — the essential part — write the **soundness argument**: what does your encoding assume about
   the real policy evaluation engine, where might the encoding be *unsound* (claiming something is
   unreachable when it is not), and where is it merely *incomplete* (claiming it cannot decide)?
   Unsoundness in a security tool is much worse than incompleteness, and being able to say which
   yours is, and why, is the difference between a tool and a liability.

## 6. Self-check

1. Why does NP-completeness not make SAT useless in practice? Name the key technique.
2. Give six SMT theories with an application of each.
3. Why must bitvectors be used to reason about machine integers?
4. Give the three answers and the idiom for proving a property.
5. What is an `unsat` core and why does it turn a solver into a tool?
6. Give five engineering applications with the shape of the encoding.
7. Give six modelling rules for keeping queries fast and decidable.
8. What are the two validation checks every encoding needs, and what do they catch?
9. Why is a trivially-unsatisfiable encoding the most dangerous failure mode?
10. What can SMT not do that TLA+ can, and vice versa?

## 7. Primary sources

- **de Moura & Bjørner, "Z3: An Efficient SMT Solver" (TACAS 2008)** and the Z3 Python tutorial —
  start with the tutorial, read the paper after.
- **Backes et al., "Semantic-based Automated Reasoning for AWS Access Policies using SMT"
  (FMCAD/CAV 2018)** — Zelkova; the best available example of formal methods deployed invisibly at
  enormous scale, and the model for Stage 9.
- Barrett & Tinelli, "Satisfiability Modulo Theories" (Handbook of Model Checking, 2018) — the
  survey.
- Marques-Silva & Sakallah, "GRASP" (1999) and Moskewicz et al., "Chaff" (DAC 2001) — CDCL and
  VSIDS, the innovations that made SAT practical.
- Cadar, Dunbar & Engler, "KLEE" (OSDI 2008) — symbolic execution; Stage 9's real version.
- Bjørner, Phan & Fleckenstein, "νZ — An Optimizing SMT Solver" (TACAS 2015).
- Leino, "Dafny: An Automatic Program Verifier" (LPAR 2010) — wp plus SMT, packaged.
- Jayaraman et al. and the network verification literature (Batfish, Minesweeper) — CA-731 L04's
  reachability question, solved with SMT.
- CS-621 L08, re-read: this lesson is what NP-completeness looks like when engineers stop treating it
  as a verdict.

---

**Previous:** [L08](L08-property-based-testing.md) · **Next:**
[L10 — Runtime Verification, Contracts, and Choosing a Technique](L10-choosing-a-technique.md)
