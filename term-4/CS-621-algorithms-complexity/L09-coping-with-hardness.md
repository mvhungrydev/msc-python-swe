# CS-621 · Lesson 09 — Coping with Hardness

**Estimated study time:** 4 hours
**Prerequisites:** L03, L08

---

## 1. Orientation

You have proved the problem NP-hard. Now what?

This is the lesson that converts theory into engineering. The options are genuinely
different, they have different guarantees, and choosing between them badly is how teams end
up with an untrustworthy heuristic that nobody dares change.

The five strategies:

1. **Solve it anyway** — modern solvers are extraordinary and your instances may be easy.
2. **Approximate with a proved ratio.**
3. **Exploit special structure** in your instances.
4. **Parameterize** — exponential in a small parameter, polynomial in the input.
5. **Heuristic with measurement** — no guarantee, but a known gap.

They are ordered by how much you can promise, and you should try them in roughly that order.

## 2. Theory

### 2.1 Strategy 1: solve it anyway

The most underused option. NP-hardness is a worst-case statement (L08 §2.5), and modern
solvers exploit structure that random worst-case instances lack.

**SAT solvers.** CDCL (conflict-driven clause learning) solvers — MiniSat, Glucose,
CaDiCaL, Kissat — routinely solve industrial instances with millions of variables. Hardware
verification, software model checking, dependency resolution (SE-511 L09 §2.3), and
configuration all run on them.

**SMT solvers** (Z3, CVC5) add theories: integers, reals, bit-vectors, arrays,
uninterpreted functions. Enormously more expressive, and the interface you will actually use
(FM-751 L06).

**MIP solvers** (Gurobi, CPLEX, HiGHS, CBC, OR-Tools) for integer programs. Branch and bound
plus cutting planes plus presolve. Commercial ones are startlingly good.

**CP solvers** (OR-Tools CP-SAT) for constraint programming — often the best choice for
scheduling and assignment, and CP-SAT in particular is exceptional.

The practical procedure: **encode your problem and try a solver before writing any heuristic.**
An afternoon of encoding may replace a month of heuristic development, and the result is
optimal with a proof.

When it fails: instances near the phase transition (L08 §3, Exercise C); very large
instances; and problems where the natural encoding blows up. When it succeeds, you also get
something a heuristic never gives you: **a bound**, so you know how good the answer is.

### 2.2 Strategy 2: approximation algorithms

An algorithm with a proved ratio: for a minimization problem, `ALG ≤ ρ · OPT` for all
instances.

**Vertex cover, 2-approximation.** Repeatedly pick any uncovered edge and take *both*
endpoints.

> **Proof.** The edges picked form a matching (no two share a vertex, since after picking an
> edge both its endpoints are covered). Any vertex cover must include at least one endpoint of
> each matching edge, so `OPT ≥ |M|`. Our solution has `2|M|` vertices. Hence
> `ALG = 2|M| ≤ 2·OPT`. ∎

Three lines, and it illustrates the standard technique: **bound OPT from below by something
you can count, then bound your algorithm above by a multiple of it.** You never compute OPT;
you bound it.

**Set cover, `ln n`-approximation.** Greedily take the set covering the most uncovered
elements. And — remarkably — `(1−ε)ln n` is the best possible unless P = NP (Dinur–Steurer,
2014). So the obvious greedy is *optimal among polynomial algorithms*.

**Metric TSP, 3/2-approximation (Christofides).** Build an MST, find a minimum-weight perfect
matching on the odd-degree vertices, combine into an Eulerian multigraph, shortcut. Requires
the triangle inequality. (A 2-approximation from just doubling the MST is easier and worth
implementing first.) The bound stood from 1976 until a 2020 improvement by an amount too
small to matter practically — which is itself informative about how hard this is.

**General TSP is inapproximable** to within any constant factor unless P = NP. The metric
assumption is doing all the work.

**PTAS and FPTAS.** A polynomial-time approximation scheme takes ε and gives a
`(1+ε)`-approximation in time polynomial in n (possibly exponential in 1/ε). An FPTAS is
polynomial in both n and 1/ε.

**Knapsack has an FPTAS**: round the values to multiples of `εV/n`, run the DP on the rounded
values. Time `O(n²/ε)`. This is the best possible outcome for an NP-hard problem — arbitrary
accuracy for polynomial cost — and it is worth knowing which problems have one.

**The inapproximability hierarchy**, which tells you what to hope for:

| Class | Example | What to expect |
|---|---|---|
| FPTAS | knapsack, subset sum | any accuracy you want |
| PTAS | Euclidean TSP, some scheduling | any accuracy, expensive in 1/ε |
| Constant factor | vertex cover (2), metric TSP (1.5) | a fixed, provable ratio |
| Logarithmic | set cover (ln n) | ratio grows slowly with n |
| Polynomial | max clique (n^(1−ε)) | essentially no useful guarantee |
| None | general TSP | no constant factor possible |

Knowing where your problem sits tells you, before you start, whether a guarantee is
achievable.

### 2.3 Strategy 3: exploit structure

Your instances are not adversarial. Real structure that makes hard problems easy:

- **Planarity.** Many NP-hard graph problems have PTASes on planar graphs (Baker's technique).
  Road networks are nearly planar.
- **Bounded treewidth.** DP over a tree decomposition solves many NP-hard problems in
  `O(f(w)·n)`. Real dependency graphs often have small treewidth.
- **Bounded degree.** Vertex cover on degree-≤3 graphs is easier.
- **Interval structure.** Interval graphs make colouring, independent set, and clique
  polynomial. Any problem about time intervals is worth checking for this — and scheduling
  problems frequently *are* interval problems in disguise, which moves them from L08's hard
  table to its easy one.
- **Bipartiteness.** Matching, vertex cover (König), and colouring all become easy.
- **Small numbers.** Pseudo-polynomial DP works when the values are bounded (L08 §2.5).
- **Fixed dimensions.** Geometric problems in the plane are often much easier than in high
  dimensions.

**The move**: before accepting hardness, characterize your actual instances. Measure the
treewidth, check for planarity, check whether the constraint graph is an interval graph.
Fifteen minutes of measurement can move you into a polynomial class.

### 2.4 Strategy 4: parameterized complexity

The insight: hardness may depend on a parameter that is *small in practice*.

A problem is **fixed-parameter tractable (FPT)** in parameter k if it is solvable in
`O(f(k) · n^c)` — arbitrary dependence on k, polynomial in n.

**Vertex cover is FPT.** `O(2^k · n)`: pick any edge; one of its two endpoints must be in the
cover; branch on both, decrementing k. Depth k, branching factor 2. So for k = 20 and
n = 10⁶, this is entirely practical while the problem is NP-hard in general.

Contrast **clique**, which is W[1]-hard — believed not FPT. `O(n^k)` is the best known, and
that is very different from `O(2^k · n)`: at n = 10⁶ and k = 10, one is 10⁶ operations and
the other is 10⁶⁰.

**Kernelization** — reduce the instance in polynomial time to a "kernel" of size bounded by a
function of k, then brute-force the kernel. Vertex cover has a `2k`-vertex kernel: any vertex
of degree > k must be in any cover of size ≤ k, so take it and decrement.

Useful parameters in practice: solution size, treewidth, number of distinct values, maximum
degree, number of constraint types, VC dimension.

**The question to ask about any hard problem: what is small about my instances?** That
question, asked systematically, is what parameterized complexity contributes, and it is
useful even without the theory.

### 2.5 Strategy 5: heuristics, measured

No guarantee, but often excellent in practice. The main families:

- **Greedy** (L03) — fast, simple, sometimes provably good.
- **Local search** — start somewhere, improve by local moves until stuck. Add restarts.
- **Simulated annealing** — accept worsening moves with a temperature-dependent probability.
  Converges to the optimum given infinite time, which is not a useful guarantee, but it works
  well.
- **Tabu search** — local search with a memory of recent moves to escape cycles.
- **Genetic algorithms** — popular, and usually beaten by well-designed local search. Be
  skeptical.
- **Large neighbourhood search** — destroy part of a solution and repair it optimally with an
  exact solver. Often the best practical approach for routing and scheduling, and it composes
  with strategy 1.
- **Beam search / branch and bound with a time limit** — exact search that returns the best
  found plus a bound.

**The non-negotiable requirement: measure the gap.**

You cannot compare against OPT for hard instances, so use one of:

1. **Small instances** where brute force finds OPT.
2. **A lower bound** — an LP relaxation, or the bound from a truncated branch and bound. Then
   `gap = (ALG − LB) / LB` is an upper bound on the true gap.
3. **A better algorithm** run with a long time limit as a reference.
4. **Known-optimum benchmark instances** (TSPLIB and its equivalents exist for most classic
   problems).

Then report the distribution of the gap over *your* instance distribution, and — this is the
part that matters operationally — **monitor it**, because a change in the input distribution
can silently degrade a heuristic that was fine for two years.

### 2.6 Choosing

The procedure:

1. **How large are the instances, actually?** If n ≤ 20, brute force. If n ≤ 40, DP over
   subsets or branch and bound.
2. **Try a solver.** An afternoon. Measure the solve time at realistic scale.
3. **Is there structure?** Measure it (§2.3).
4. **Is a parameter small?** (§2.4.)
5. **Is there a known approximation with an acceptable ratio?** (§2.2.)
6. **Otherwise: heuristic, with the gap measured and monitored.**

And — asked first, before any of it:

**0. Do you need the optimum?** Very often the answer is no. A solution within 3% of optimal,
computed in 50 ms, may be strictly better for the business than the optimum in 40 minutes.
The question is not "what is optimal?" but "what decision does this feed, and how sensitive
is that decision to the last 3%?" Asking it has ended more of these projects, correctly, than
any algorithm has solved.

### 2.7 The engineering framing

Present the choice honestly:

| Approach | Guarantee | Time | When |
|---|---|---|---|
| Exact solver | optimal, with proof | variable, may not finish | small/structured; correctness matters |
| Exact + time limit | best found + a bound | bounded | you can use "within X% of optimal" |
| Approximation | ratio ρ, always | polynomial | you need a promise for every instance |
| FPT | optimal | `f(k)·n^c` | the parameter is genuinely small |
| Heuristic | none | fast | measured gap is acceptable |

**The "exact + time limit" row is undersold.** A MIP or CP solver stopped after 10 seconds
returns both a solution and a bound on how far it might be from optimal. That is a strictly
better product than a heuristic's bare answer: you can *tell the user* the answer is within
1.2% of optimal, and you can alert when that degrades.

## 3. Construction: coping, end to end

Take a genuinely NP-hard problem from your own domain. If you do not have one: assigning n
services to m machines with affinity and anti-affinity constraints while balancing load.

**Step 1 — characterize the instances.** n, m, density, structure. Are they planar,
interval-shaped, bipartite, bounded-degree? Measure the treewidth (there are libraries).
Report what you find. Do not skip this; it is where the surprises are.

**Step 2 — establish ground truth.** For the smallest realistic instances, compute the
optimum by brute force or with an exact solver. You need this to measure anything.

**Step 3 — try a solver.** Encode as CP-SAT (OR-Tools) or MIP. Measure the solve time across
your instance sizes. Find the size where it stops finishing. Record both the solution and the
bound.

**Step 4 — the parameter question.** What is small about your instances? Try an FPT
formulation if one applies. Report whether the parameter is genuinely small in your data —
this frequently fails, and the failure is informative.

**Step 5 — approximation.** Is there a known approximation for your problem or a close
relative? Implement it. Measure the actual ratio against ground truth, and compare with the
proved ratio (the actual is almost always far better, which is worth quantifying).

**Step 6 — heuristics.** Implement greedy and local search with restarts. Measure the gap
against ground truth (small instances) and against the solver's bound (large ones).

**Step 7 — large neighbourhood search.** Combine: use the heuristic for the overall
structure, and the exact solver to re-optimize small subproblems. Measure. This combination
frequently wins and is under-used.

**Step 8 — the comparison.** One table: solution quality (gap), time, and the guarantee, for
every approach, across instance sizes. Plus the anytime behaviour — for the solver and local
search, plot solution quality against time budget.

**Step 9 — the recommendation and the monitoring.** Which approach, at what instance size,
with what fallback. Then: how would you detect in production that the gap has degraded
because the input distribution changed? Design that monitor. Its absence is the most common
long-term failure of a shipped heuristic.

## 4. Failure modes

- **Not trying a solver.** The most common and most expensive omission.
- **Writing a heuristic for an instance size where brute force works.**
- **Not measuring the gap.** A heuristic with an unknown gap is an unquantified risk.
- **Measuring the gap once and never again.** Distributions change.
- **Reporting a heuristic's answer as optimal.**
- **Assuming worst-case hardness applies to your instances** without checking for structure.
- **Genetic algorithms by default.** They are appealing and usually beaten by simpler local
  search; if you use one, benchmark it against local search first.
- **Ignoring the anytime property.** A solver that gives a good answer in 1 s and the optimum
  in 1 h may be usable at either budget.
- **Optimizing the wrong objective.** The mathematically clean objective often is not what the
  business wants; check before spending weeks on it.
- **Not asking whether the optimum is needed.** §2.6 step 0.

## 5. Exercises

### Warm-up (30 min)

**W1.** Implement and prove the vertex cover 2-approximation. Measure the actual ratio on 1,000
random graphs and compare with the bound.

**W2.** Implement the FPT vertex cover algorithm. Find the largest k for which it is practical
at n = 10⁵.

**W3.** Encode graph colouring in Z3 or OR-Tools and solve instances up to the size where it
stops finishing. Report that size.

### Core (3 h)

**C1 — The full coping study.** Complete §3, all nine steps, on a real problem. Deliverable:
the instance characterization, the ground truth, all five approaches implemented, the
comparison table with gaps and guarantees, the anytime plots, the recommendation, and the
monitoring design.

**C2 — Approximation ratios, measured.** For three approximation algorithms (vertex cover 2,
set cover ln n, metric TSP 2 via MST-doubling), measure the *actual* ratio over 1,000
instances and compare with the proved bound. Report the distribution. Then construct, for
each, an instance that achieves (or nearly achieves) the worst case — this is what shows you
the bound is tight.

**C3 — Knapsack FPTAS.** Implement it. Verify that the `(1−ε)` guarantee holds empirically for
several ε. Plot solution quality and runtime against ε. Then compare against: exact DP, greedy
by ratio, and a MIP solver. Report which you would ship for three different (n, W) regimes.

**C4 — Large neighbourhood search.** Complete §3 step 7 properly on a routing or scheduling
problem: a construction heuristic, then repeated destroy-and-repair using an exact solver on
subproblems of tunable size. Sweep the subproblem size and report quality against time. Find
the optimum size and explain it.

### Challenge

**X1.** Take a real scheduling or assignment problem from your organization. Do the full study
and *present the result to the people who own it*, including: the current heuristic's measured
gap, what an exact solver achieves, the runtime trade, and a recommendation. Report what
happened — including if the answer was "the current heuristic is fine", which is a common and
valuable finding.

**X2.** Read Vazirani's *Approximation Algorithms* chapters on LP relaxation and rounding.
Implement the LP-relaxation-plus-rounding approximation for set cover, and compare with the
greedy `ln n` algorithm on the same instances. Report which is better in practice and by how
much, and explain the discrepancy between the theoretical ratios and the measured ones.

## 6. Self-check

1. Give the five coping strategies in order, and the question that comes before all of them.
2. Give the vertex cover 2-approximation and its proof, and name the general proof technique.
3. Give the inapproximability hierarchy with an example at each level.
4. Define FPT and contrast `O(2^k·n)` with `O(n^k)` numerically at n = 10⁶, k = 10.
5. Name five kinds of structure that make NP-hard problems easy.
6. Give four ways to measure a heuristic's gap without knowing OPT.
7. Why is "exact solver with a time limit" often better than a heuristic?
8. What monitoring does a shipped heuristic need, and why?

## 7. Primary sources

- Vazirani, *Approximation Algorithms* (2001).
- Williamson & Shmoys, *The Design of Approximation Algorithms* (2011, free online).
- Cygan et al., *Parameterized Algorithms* (2015, free online).
- Christofides, "Worst-case analysis of a new heuristic for the travelling salesman problem"
  (1976).
- Dinur & Steurer, "Analytical approach to parallel repetition" (STOC 2014) — the set cover
  inapproximability result.
- OR-Tools CP-SAT documentation, and the annual MiniZinc/SAT competition results, for what
  solvers can actually do.

---

**Previous:** [L08](L08-np-completeness.md) · **Next:**
[L10 — Computability, Halting, and Rice's Theorem](L10-computability.md)
