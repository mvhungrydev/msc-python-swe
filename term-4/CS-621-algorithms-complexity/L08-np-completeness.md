# CS-621 · Lesson 08 — NP-Completeness and Reductions

**Estimated study time:** 5 hours
**Prerequisites:** L01–L05

---

## 1. Orientation

The practical value of this lesson is a permission and a prohibition.

**The permission**: when you can show a problem is NP-hard, you are entitled to stop looking
for an exact efficient algorithm and start designing an approximation, a heuristic, or a
restriction — and you can *justify that decision to other people*. Without the proof, "I
couldn't find a fast algorithm" is an admission; with it, it is a theorem.

**The prohibition**: it stops three weeks of a smart engineer's time being spent on a problem
that fifty years of research has not solved.

The secondary value is recognition. Most hard problems in industry are a known NP-complete
problem in disguise, and the disguise is usually thin. Scheduling with constraints is graph
colouring. Choosing which features to ship within a budget is knapsack. Deciding which
servers to shut down is set cover. Once you recognize it, the entire literature — including
the good approximation algorithms — is available to you.

## 2. Theory

### 2.1 Decision problems and the classes

Complexity theory is stated for **decision problems** — yes/no questions — because the
machinery is cleaner. Optimization problems are handled by their decision versions
("is there a solution of value ≥ k?"), and the two are polynomially equivalent by binary
search on k (L02 §2.3).

**P** — decidable in polynomial time by a deterministic Turing machine.

**NP** — decidable in polynomial time by a *nondeterministic* machine. Equivalently, and far
more usefully:

> **A problem is in NP if a proposed solution can be *verified* in polynomial time.**

The verifier definition is the one to carry. Is this graph 3-colourable? Given a colouring,
you can check it in linear time. Is this formula satisfiable? Given an assignment, evaluate
it. **NP is the class of problems whose solutions are easy to check.**

Note that "easy to check" says nothing about being easy to find, and the P vs NP question is
exactly whether those coincide.

**co-NP** — the complements. Verifying a *no* answer is easy. "Is this formula
unsatisfiable?" is in co-NP, and is not known to be in NP.

**NP-hard** — at least as hard as everything in NP: every NP problem reduces to it in
polynomial time. An NP-hard problem need not be in NP, and need not be a decision problem
(the halting problem is NP-hard and undecidable).

**NP-complete** — in NP *and* NP-hard. The hardest problems in NP, and they all stand or fall
together: a polynomial algorithm for any one gives one for all.

```
        ┌─────────── NP-hard ───────────┐
        │                               │
   ┌────┴──── NP ────┐          (halting problem,
   │   ┌── NPC ──┐   │           TSP optimization,
   │   │         │   │           and other non-NP problems)
   │   └─────────┘   │
   │   ┌── P ──┐     │
   │   └───────┘     │
   └─────────────────┘
                        (assuming P ≠ NP)
```

### 2.2 Reductions

`A ≤_p B` ("A reduces to B in polynomial time") means: there is a polynomial-time function f
mapping instances of A to instances of B such that `x ∈ A ⟺ f(x) ∈ B`.

The intuition: **if you can solve B, you can solve A** — transform, solve, done. So B is at
least as hard as A.

**The direction is the thing people get wrong.** To prove B is NP-hard, you reduce a *known
NP-hard* problem A **to** B. Not B to A. The mnemonic: *reduce FROM known-hard TO your
problem.* Reducing your problem to SAT proves your problem is *easy* (or at least no harder
than SAT), which is useful for solving it but proves nothing about hardness.

Reduction is transitive, so once you have one NP-complete problem you get the rest by
chaining.

**The template for proving B NP-complete:**

1. **Show B ∈ NP.** Give a certificate and a polynomial verifier. Usually two lines, and
   frequently forgotten — without it you have proved NP-hard, not NP-complete.
2. **Choose a known NP-complete problem A** structurally similar to B.
3. **Give the reduction f**, mapping instances of A to instances of B.
4. **Show f is polynomial-time.**
5. **Prove correctness, both directions**: `x ∈ A ⟹ f(x) ∈ B` and `f(x) ∈ B ⟹ x ∈ A`. The
   second direction is where most attempted proofs fail, and it is where the real work is.

### 2.3 Cook–Levin and the first NP-complete problem

**Theorem (Cook 1971, Levin 1973).** SAT is NP-complete.

The proof idea: for any problem in NP with a polynomial-time verifier, encode the entire
computation of that verifier — the tape contents at every step, the head position, the state
— as boolean variables, and write clauses asserting that the computation is valid and
accepting. The formula is satisfiable exactly when an accepting certificate exists.

Read the proof once, in Sipser. You will not use it directly, but it is the foundation, and
understanding that "a computation is a satisfiability problem" is a genuinely load-bearing
idea — it is also why SAT solvers can verify hardware and software (FM-751 L06).

**3-SAT** (every clause has exactly 3 literals) is also NP-complete, and is the usual
starting point for reductions because its structure is convenient. Note the boundary:
**2-SAT is in P** (L05 §2.4, via SCC). One literal per clause fewer, and the problem changes
class entirely. That kind of sharp boundary is characteristic.

### 2.4 Karp's 21 and the core catalogue

Karp (1972) showed 21 problems NP-complete by reduction from SAT, establishing that this was
a widespread phenomenon rather than a curiosity. The ones to know cold:

| Problem | Statement |
|---|---|
| **3-SAT** | Satisfy a CNF formula with 3 literals per clause |
| **Clique** | Is there a set of k mutually adjacent vertices? |
| **Independent set** | k mutually non-adjacent vertices? |
| **Vertex cover** | k vertices touching every edge? |
| **Hamiltonian cycle** | A cycle visiting every vertex once? |
| **TSP (decision)** | A tour of length ≤ k? |
| **Graph colouring** | Colour with k colours, no two adjacent alike? |
| **Subset sum** | A subset summing to exactly T? |
| **Knapsack (decision)** | Value ≥ V within weight W? |
| **Set cover** | k sets covering the universe? |
| **Bin packing** | Fit items into k bins? |
| **Partition** | Split into two equal-sum halves? |
| **3-dimensional matching** | A perfect matching in a tripartite hypergraph? |
| **Scheduling with precedence** | Schedule on m machines by deadline d? |
| **Integer programming** | Is there an integer solution? |

**The three easiest reductions, worth doing by hand:**

*Independent set ≤ Vertex cover.* S is independent ⟺ V∖S is a vertex cover. So the graph has
an independent set of size k iff it has a vertex cover of size n−k. Trivially polynomial, and
correct in both directions. (Two sentences, and it is a complete proof.)

*3-SAT ≤ Independent set.* Build a triangle per clause, one vertex per literal. Add an edge
between every pair of contradictory literals across clauses. Claim: the formula is
satisfiable iff there is an independent set of size m (the number of clauses).

Forward: a satisfying assignment picks one true literal per clause; those m vertices are
pairwise non-adjacent (not in the same triangle, not contradictory). Backward: an independent
set of size m must take exactly one vertex per triangle, and the chosen literals are
consistent (no contradictory pair), so setting them true satisfies every clause. ∎

*Vertex cover ≤ Set cover.* Elements are edges; for each vertex, a set containing its incident
edges. A vertex cover of size k is exactly a set cover of size k. ∎

Work each of these on paper. They are the model for every reduction you will construct.

### 2.5 What NP-completeness does and does not mean

Careful distinctions, because the informal version supports bad inferences:

- **It is a worst-case statement.** Real SAT solvers routinely solve industrial instances with
  millions of variables. NP-hardness does not mean "hard on your inputs" — it means "no
  algorithm is fast on *all* inputs".
- **It says nothing about small n.** A Θ(2ⁿ) algorithm at n = 25 is 33 million operations —
  instantaneous. If your n is bounded and small, NP-hardness is irrelevant.
- **It is about the *problem*, not your instances.** Your instances may have structure
  (bounded treewidth, planarity, a small number of distinct values) that makes them easy.
- **Approximation may be easy.** Vertex cover has a trivial 2-approximation; set cover a
  `ln n` one. Some NP-hard problems approximate beautifully; others (like general TSP)
  cannot be approximated at all unless P = NP.
- **P ≠ NP is unproven.** Almost universally believed, with a $1M prize outstanding. All of
  the above is conditional on it — and the conditionality does not matter in practice.

**Pseudo-polynomial and strong NP-completeness.** Knapsack's Θ(nW) DP (L04 §2.4) is polynomial
in W but W is written in `log W` bits, so it is exponential in the *input size*. Such
algorithms are *pseudo-polynomial*, and problems admitting them are "weakly" NP-complete. A
problem is **strongly NP-complete** if it stays NP-complete when all numbers are bounded by a
polynomial in the input size — 3-SAT, clique, and TSP are; subset sum and knapsack are not.
The distinction matters: it tells you whether the DP escape hatch is available.

### 2.6 Beyond NP

Worth knowing the landscape exists:

- **PSPACE** — polynomial space. Contains NP. Complete problems: QBF (quantified boolean
  formulas), generalized games (Go, Hex), and — relevant to FM-751 — many model-checking
  problems.
- **EXPTIME** — provably strictly larger than P. Some games are EXPTIME-complete.
- **#P** — counting the solutions. `#SAT` is #P-complete, and counting can be much harder
  than deciding: counting perfect matchings is #P-complete while *finding* one is in P.
  Relevant to probabilistic inference.
- **The polynomial hierarchy** — alternating quantifiers. `∃∀` problems (like "is this circuit
  minimal?") sit at the second level.
- **Undecidable** — no algorithm at all. L10.

The practical hierarchy for an engineer: *in P* → *NP-hard but approximable* → *NP-hard and
inapproximable* → *PSPACE-hard* → *undecidable*. Each step means a different coping strategy.

### 2.7 The recognition table

The most useful thing here. When you meet a hard problem at work, check it against this:

| Your problem | The classic |
|---|---|
| Assign resources to tasks with conflicts | graph colouring |
| Pick items within a budget maximizing value | knapsack |
| Cover all requirements with fewest components | set cover |
| Visit all locations minimizing travel | TSP |
| Pack items into containers | bin packing |
| Split work evenly | partition / makespan scheduling |
| Choose a maximal set of mutually compatible things | independent set |
| Find a set of things touching every constraint | vertex cover |
| Satisfy a set of boolean constraints | SAT (2 literals/clause: easy!) |
| Order tasks with precedence and deadlines | scheduling |
| Find a group where everyone is connected | clique |
| Choose locations to serve all customers | facility location / set cover |
| Match with more than two sides | 3-dimensional matching |

**And the easy ones it is vital not to mistake for hard ones:**

| Your problem | The classic — and it is in P |
|---|---|
| Pair up two groups | bipartite matching |
| Order with dependencies (no deadlines) | topological sort |
| Connect everything cheaply | MST |
| Cheapest route | shortest path |
| Maximum throughput | max flow |
| Boolean constraints with 2 literals per clause | 2-SAT |
| Assignment with costs (one-to-one) | Hungarian algorithm |

The second table matters as much as the first. Assuming a problem is hard when it is
polynomial is the mirror error, and it leads to shipping a heuristic where an exact
algorithm was available in `scipy.optimize.linear_sum_assignment`.

## 3. Construction: proving and recognizing

**Exercise A — three reductions, by hand.**

1. Independent set ≤ Vertex cover (two sentences).
2. 3-SAT ≤ Independent set (the full both-directions proof).
3. Vertex cover ≤ Set cover.

Then two harder ones:

4. **3-SAT ≤ Subset sum.** (Hint: build numbers in base 10 with a digit per variable and per
   clause, arranged so that no carrying occurs.)
5. **Hamiltonian cycle ≤ TSP.** (Hint: weight 1 for existing edges, 2 for non-edges; ask for a
   tour of length ≤ n.)

For each: show the problem is in NP, give f, show f is polynomial, and prove both directions.
On paper.

**Exercise B — recognize and prove.** Take three problems from your own work that felt hard.
For each:

1. State it precisely as a decision problem.
2. Search the recognition table (§2.7). Which classic does it resemble?
3. If NP-hard, construct the reduction. If in P, name the polynomial algorithm.
4. **If neither is obvious, that is the interesting case** — write down what makes it
   different from the nearest classic, because that difference is either the source of the
   hardness or the special structure that makes it easy.

**Exercise C — the empirical picture.** Implement a brute-force solver and a SAT-solver-based
solver (using `python-sat` or Z3) for graph colouring. Then:

- Generate random graphs at increasing size and measure the time to decide 3-colourability.
- Plot time against n for both.
- Vary the **edge density** and find the phase transition — random 3-SAT and random graph
  colouring both exhibit a sharp threshold where instances become hard, with easy instances
  on both sides. Locate it.

This is the single most useful empirical fact about NP-hardness: **hardness is concentrated
near a phase transition, and most real instances are not there.** Seeing it changes how you
think about "NP-hard" as a practical statement.

**Exercise D — encode your problem in SAT.** Take a real constraint problem from your work
(scheduling, configuration, resource assignment). Encode it as SAT or as an integer program
and solve it with Z3, OR-Tools, or a MIP solver.

Report: encoding size, solve time at realistic scale, and whether the exact solution differs
meaningfully from whatever heuristic is used today.

**This exercise is where the course pays for itself.** Modern SAT and MIP solvers are
extraordinary, and a large number of "we use a heuristic because the problem is NP-hard"
situations should actually be "we encode it and let a solver do it in 200 ms". Knowing when
that applies is a genuinely valuable and uncommon skill.

## 4. Failure modes

- **Reducing in the wrong direction.** Reducing your problem to a known-hard one proves
  nothing about hardness.
- **Forgetting to show the problem is in NP.** You have proved NP-hard, not NP-complete.
- **Proving only one direction** of the reduction's correctness.
- **A reduction that is not polynomial.** Check the size of f(x).
- **"NP-hard, therefore hopeless."** Ignores approximation, small n, special structure, and
  the effectiveness of real solvers.
- **"NP-hard, therefore my heuristic is fine."** It might be terrible; measure the gap (L03
  §2.6, L09).
- **Assuming a problem is hard without checking the easy table.** Bipartite matching mistaken
  for something hard is a real and costly error.
- **Confusing pseudo-polynomial with polynomial.**
- **Applying worst-case hardness to your specific instances**, which may have exploitable
  structure.
- **Not trying a solver.** Twenty minutes of encoding often beats three weeks of heuristic
  tuning.

## 5. Exercises

### Warm-up (30 min)

**W1.** Complete Exercise A items 1–3.

**W2.** For five problems from the §2.7 tables, state the decision version and the certificate
that shows it is in NP.

**W3.** Give an example where a Θ(2ⁿ) algorithm is entirely acceptable, and state the n.

### Core (3 h)

**C1 — The reductions.** Complete Exercise A, all five, on paper with both directions proved.
Items 4 and 5 are the assessed ones.

**C2 — Recognition.** Complete Exercise B for three real problems. Deliverable: the precise
decision statements, the classification with justification, the reduction or the polynomial
algorithm, and — for any that fit neither — a written analysis of what makes them different.

**C3 — The phase transition.** Complete Exercise C. Deliverable: the time-versus-n plots, the
density sweep, the located phase transition, and a 400-word note on what this implies for
interpreting "NP-hard" as an engineering statement.

**C4 — Solve it with a solver.** Complete Exercise D. Deliverable: the encoding, the solve
times at three scales, the comparison against the incumbent heuristic, and a recommendation.
If the solver is not viable at your scale, say so with the evidence.

### Challenge

**X1.** Read the Cook–Levin proof in Sipser ch. 7.4. Then implement a (small-scale) reduction
from an arbitrary polynomial-time verifier to SAT: take a simple verifier, encode its
computation as a formula, and verify with a SAT solver that satisfiability corresponds to
acceptance. This is a substantial exercise and it makes the theorem concrete in a way reading
cannot.

**X2.** Take an NP-hard problem from your domain and build a proper solver comparison: a
greedy heuristic, a local-search heuristic, a SAT/SMT encoding, and a MIP encoding. Benchmark
on real instances: solution quality (against the optimum where computable, or against the
best bound), solve time, and robustness across instance types. Report which you would ship
and under what conditions you would switch. This is exactly the analysis L09 formalizes.

## 6. Self-check

1. Give the verifier definition of NP and explain why it is the useful one.
2. Distinguish NP-hard from NP-complete, and give a problem that is one but not the other.
3. State the reduction direction for proving hardness, and explain the mnemonic.
4. Give the five steps of an NP-completeness proof and say which is most often botched.
5. State Cook–Levin and its proof idea in one sentence.
6. Give the 3-SAT ≤ Independent set reduction with both directions.
7. Give five things NP-completeness does *not* mean.
8. Distinguish weak from strong NP-completeness and say why it matters.

## 7. Primary sources

- Sipser, *Introduction to the Theory of Computation*, 3rd ed., ch. 7. The clearest treatment
  available; read all of it.
- Cook, "The Complexity of Theorem-Proving Procedures" (STOC 1971).
- Karp, "Reducibility Among Combinatorial Problems" (1972). Short, and the paper that made
  this a field.
- Garey & Johnson, *Computers and Intractability* (1979). The appendix cataloguing ~300
  NP-complete problems is still the reference for recognition.
- Kleinberg & Tardos, ch. 8.
- Cheeseman, Kanefsky & Taylor, "Where the Really Hard Problems Are" (IJCAI 1991) — the phase
  transition.

---

**Previous:** [L07](L07-sketches-and-streaming.md) · **Next:**
[L09 — Coping with Hardness](L09-coping-with-hardness.md)
