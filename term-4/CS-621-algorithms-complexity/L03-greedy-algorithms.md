# CS-621 · Lesson 03 — Greedy Algorithms and Exchange Arguments

**Estimated study time:** 4 hours
**Prerequisites:** L01, L02

---

## 1. Orientation

Greedy algorithms are the shortest to write and the hardest to trust. "Take the locally best
option at each step" gives an optimal answer for interval scheduling, Huffman coding, and
minimum spanning trees — and a badly suboptimal one for knapsack, set cover, and travelling
salesman.

The technique is therefore inseparable from its proof. **A greedy algorithm without a proof
is a heuristic**, and calling it an algorithm is a category error that leads directly to
shipping something that is wrong on inputs you have not seen.

So this lesson is about proof technique as much as about algorithms: the exchange argument
and the "greedy stays ahead" argument are the two tools, and they are reusable far beyond
this course.

## 2. Theory

### 2.1 The pattern

```
solve(items):
    sort or prioritize items by some criterion
    solution = ∅
    for item in that order:
        if adding item is feasible:
            add it
    return solution
```

The entire design is **the choice of criterion**, and the entire correctness question is
whether that criterion is right. Most wrong greedy algorithms are wrong because the obvious
criterion is not the correct one — and the obvious one is often *nearly* right, which is
worse, because it passes your tests.

### 2.2 Proving greedy correct: two arguments

**Argument 1 — "greedy stays ahead".** Show that after each step, the greedy partial
solution is at least as good as any other partial solution by some measure. Then it must be
at least as good at the end.

*Interval scheduling.* Given intervals with start and finish times, select the largest set of
mutually non-overlapping intervals. Greedy: **repeatedly take the interval that finishes
earliest** among those compatible with what you have.

> **Claim.** Greedy is optimal.
> **Proof.** Let greedy select `g₁, …, g_k` in order of finish time, and let any optimal
> solution be `o₁, …, o_m` also in order. We show by induction that `finish(g_i) ≤
> finish(o_i)` for all i ≤ k.
> *Base:* g₁ is the earliest-finishing interval of all, so `finish(g₁) ≤ finish(o₁)`.
> *Step:* assume `finish(g_{i−1}) ≤ finish(o_{i−1})`. Since `o_i` starts after `o_{i−1}`
> finishes, it also starts after `g_{i−1}` finishes, so `o_i` was available to greedy at step
> i. Greedy chose the earliest-finishing available interval, so `finish(g_i) ≤ finish(o_i)`.
> Now suppose `m > k`. Then `o_{k+1}` exists and starts after `finish(o_k) ≥ finish(g_k)`, so
> `o_{k+1}` was available to greedy after step k — contradicting that greedy stopped. Hence
> `m ≤ k`, and greedy is optimal. ∎

Note the shape: an induction that greedy is "ahead", then a contradiction showing it cannot
be shorter. This is the template.

**Argument 2 — exchange argument.** Take any optimal solution. Show you can transform it,
step by step, into the greedy solution without making it worse. Therefore greedy is optimal.

*Minimizing average completion time.* Given jobs with processing times, schedule them on one
machine to minimize the sum of completion times. Greedy: **shortest job first**.

> **Proof (exchange).** Suppose an optimal schedule has adjacent jobs i then j with
> `p_i > p_j`. Swapping them leaves every other job's completion time unchanged (the pair
> occupies the same total interval). Before: the pair contributes `(t + p_i) + (t + p_i +
> p_j)`. After: `(t + p_j) + (t + p_j + p_i)`. The difference is `p_i − p_j > 0`, so the swap
> *strictly improves*. Hence no optimal schedule has an out-of-order adjacent pair, so
> shortest-job-first is optimal. ∎

The exchange argument's shape: **assume optimal differs from greedy, find a local swap that
does not hurt, repeat until it is greedy.** Both proofs are short once you know the shape,
and writing three or four of them is what makes greedy design possible rather than guesswork.

### 2.3 Where greedy fails, and why

**0/1 knapsack.** Items with weight and value; maximize value within capacity W. The obvious
greedy — by value/weight ratio — fails:

```
W = 10;  items: A(w=6, v=12, ratio 2), B(w=5, v=9), C(w=5, v=9)
Greedy by ratio: takes A (12), then neither B nor C fits. Total 12.
Optimal: B + C = 18.
```

Why it fails: the choices are *not independent*. Taking A forecloses the pair. Greedy assumes
that a locally good choice never forecloses a better global structure, and here it does.

**Fractional knapsack**, by contrast, *is* solved optimally by the same ratio greedy — because
you can take a fraction, so no choice forecloses anything. **The difference between the two
is the whole lesson about when greedy works.**

**Set cover.** Greedy (take the set covering the most uncovered elements) is not optimal, but
it is a `ln n`-approximation — and, remarkably, that is essentially the best possible unless
P = NP (L09). So a failed greedy can still be the right answer with a proved guarantee.

**Coin change.** Greedy (largest coin first) is optimal for US denominations and for any
"canonical" system, and wrong for others: with coins {1, 3, 4} and target 6, greedy gives
4+1+1 = 3 coins; optimal is 3+3 = 2 coins. **A greedy algorithm can be correct for your data
and wrong in general**, which is the most dangerous case because your tests pass.

### 2.4 Matroids: when greedy provably works

The deep answer to "when is greedy optimal?" A **matroid** is a pair (S, I) where S is a
finite set and I a family of "independent" subsets, satisfying:

1. ∅ ∈ I.
2. **Hereditary**: if A ∈ I and B ⊆ A then B ∈ I.
3. **Exchange**: if A, B ∈ I and |A| < |B|, then ∃ x ∈ B∖A with A ∪ {x} ∈ I.

**Theorem (Rado–Edmonds).** For any weighted matroid, the greedy algorithm — sort by weight
descending, add each element if it keeps the set independent — finds a maximum-weight
independent set.

Kruskal's minimum spanning tree algorithm is exactly this on the *graphic matroid*
(independent = acyclic edge sets). Interval scheduling is not a matroid but has a related
structure.

You will not usually verify the matroid axioms in practice. The value is the *intuition*: the
exchange property is precisely the "no choice forecloses a better structure" condition that
0/1 knapsack violates. When you are considering a greedy algorithm, ask: **can a locally good
choice ever make a better global solution unreachable?** If yes, greedy is probably wrong.

Matroid intersection (two matroids) is still polynomial; three is NP-hard. That boundary is
where a surprising number of scheduling problems sit.

### 2.5 The canonical greedy algorithms

**Interval scheduling** — earliest finish time. Θ(n log n). §2.2.

**Interval partitioning** — minimum rooms for all intervals. Sort by start; assign to any
free room; if none, open a new one. The number of rooms equals the maximum *depth* (maximum
number of overlapping intervals), which is an obvious lower bound — so greedy is optimal and
the proof is one line. A satisfying example of the lower bound doing the work.

**Huffman coding** — repeatedly merge the two least-frequent symbols. Θ(n log n) with a heap.
Produces an optimal prefix code. The exchange argument is a good exercise: show that in some
optimal tree the two least-frequent symbols are siblings at maximum depth.

**Minimum spanning tree.** Two greedy algorithms, both optimal:

- **Kruskal.** Sort edges by weight; add if it does not create a cycle (union–find).
  Θ(m log m).
- **Prim.** Grow one tree, always adding the cheapest edge leaving it (priority queue).
  Θ(m log n).

Both rest on the **cut property**: for any partition of the vertices, the minimum-weight edge
crossing the cut is in some MST. Proof by exchange, and worth doing.

**Dijkstra's shortest paths.** Greedy: repeatedly finalize the unvisited vertex with the
smallest tentative distance. Θ((m + n) log n) with a binary heap. **Requires non-negative
weights**, and the reason is exactly the greedy-choice argument: with a negative edge, a
vertex finalized early might later be reachable more cheaply, so the greedy choice can be
foreclosed. Understanding *why* the requirement exists is more useful than remembering it
(L05).

**Scheduling to minimize lateness** — earliest deadline first. Exchange argument.

**Fractional knapsack** — by value/weight ratio.

### 2.6 Greedy in practice

The practical position:

- **Greedy plus a proof** — ship it. Fast, simple, optimal.
- **Greedy with a proved approximation ratio** — often the right engineering answer for an
  NP-hard problem (L09). Set cover's `ln n`, and the 2-approximation for vertex cover, are
  both greedy.
- **Greedy as a heuristic with no guarantee** — acceptable *if* you measure the gap against
  optimal (or a bound) on real data, state it, and monitor it. Unacceptable if you present it
  as optimal.
- **Greedy as an initial solution** for local search or as a bound in branch and bound.

The failure mode to name: a greedy heuristic shipped as if it were optimal, whose gap nobody
has ever measured. Every "we sort by X and take greedily" in a production system deserves the
question: *how far from optimal is this on our data, and how would we know if it got worse?*

## 3. Construction: designing and proving

**Exercise A — prove or disprove.** For each, decide whether the greedy is optimal, then
either prove it (stays-ahead or exchange) or give a counterexample. Do the work before
looking anything up.

1. **Interval scheduling by shortest interval first.**
2. **Interval scheduling by earliest start time.**
3. **Interval scheduling by fewest conflicts.**
4. **Minimizing maximum lateness by earliest deadline.**
5. **Minimizing sum of completion times with release times, by shortest remaining time.**
6. **Coin change with denominations {1, 5, 10, 25}, greedy largest-first.**
7. **Maximum matching in a bipartite graph, greedily adding any available edge.**
8. **Loading a truck: put in the heaviest item that still fits.**

(1, 2, 3, 7, and 8 fail. Constructing the counterexamples is the exercise; each is small.)

**Exercise B — implement and verify.** Implement interval scheduling, interval partitioning,
Huffman, Kruskal (with union–find and path compression), and Dijkstra (with `heapq`). For
each:

- State and prove the complexity.
- Write a **brute-force reference** for small n and verify agreement on 10,000 random inputs.
  This is property-based testing (SE-511 L04 §2.2, pattern 3: model-based) and it is how you
  catch a proof you got subtly wrong.

**Exercise C — the counterexample search.** Take a greedy criterion you suspect is wrong.
Write a program that enumerates all small instances (n ≤ 8), computes the optimum by brute
force, and reports the instance with the worst greedy/optimal ratio. Run it on: knapsack by
ratio, coin change with {1,3,4}, and set cover.

This tool is genuinely useful in practice — **when you propose a greedy heuristic at work,
run this search before shipping it.** It takes an hour and it finds the counterexample your
tests will not.

**Exercise D — measure the gap on real data.** Take a real greedy heuristic (a bin-packing
assignment, a task scheduler, a cache eviction policy). Compute or bound the optimum for a
sample of real instances — by brute force where n is small, by an ILP solver
(`pulp`, `mip`, or OR-Tools) where it is not. Report the distribution of the greedy/optimal
ratio.

Then: is the gap worth closing? That is a business question and the measurement is what lets
you answer it rather than guess.

## 4. Failure modes

- **Greedy without a proof, presented as optimal.** The central failure.
- **The obvious criterion.** Shortest interval, earliest start, largest item — all natural,
  all wrong for the problems they seem to fit.
- **Greedy that is correct for your data and wrong in general.** Coin change. Tests pass;
  the next denomination change breaks it silently.
- **Assuming a greedy that works fractionally works in 0/1.**
- **Dijkstra with negative weights.** Silently wrong; use Bellman–Ford.
- **Not measuring the gap** for a heuristic in production.
- **Ignoring ties.** Greedy tie-breaking can matter for optimality and almost always matters
  for determinism and reproducibility — specify it.
- **Θ(n log n) claimed while the feasibility check is Θ(n)**, making it Θ(n²). Check the cost
  of "if adding item is feasible".

## 5. Exercises

### Warm-up (30 min)

**W1.** Construct counterexamples for Exercise A items 1, 2, and 3. Each needs at most four
intervals.

**W2.** Write the exchange argument for earliest-deadline-first minimizing maximum lateness,
in full.

**W3.** Demonstrate Dijkstra producing a wrong answer on a graph with one negative edge, and
explain which step of the greedy argument fails.

### Core (2.5 h)

**C1 — Proofs.** Complete Exercise A, all eight, with a full proof or a minimal
counterexample for each. Proofs written by hand first.

**C2 — Implement and verify.** Complete Exercise B. Deliverable: five implementations, five
brute-force references, the property-based verification over 10,000 random inputs each, and
the complexity analysis of each including the feasibility-check cost.

**C3 — The counterexample searcher.** Complete Exercise C. Deliverable: the tool, the worst
instances found for three greedy criteria, and the worst observed ratio for each. Then apply
it to a greedy criterion of your own invention for a problem from your work.

**C4 — The production gap.** Complete Exercise D on a real heuristic. Deliverable: the
optimum computation method, the ratio distribution over at least 100 real instances, a
recommendation, and — this is the assessed part — a proposal for how you would *monitor* the
gap in production so that a change in data distribution does not silently degrade it.

### Challenge

**X1.** Prove the cut property for minimum spanning trees, then use it to prove both Kruskal
and Prim optimal. Then implement Borůvka's algorithm (the third MST algorithm, and the one
that parallelizes), prove it correct via the same property, and compare all three
empirically on sparse and dense graphs.

**X2.** Read the matroid section of CLRS ch. 15 (or Kleinberg & Tardos's treatment). Verify
the three matroid axioms for the graphic matroid, and use the Rado–Edmonds theorem to
conclude Kruskal's optimality without a separate proof. Then find a scheduling problem in
your own domain, determine whether it is a matroid, and report what that tells you about
whether a greedy algorithm can be optimal for it.

## 6. Self-check

1. Give the greedy pattern and say where the entire design decision lies.
2. State the "greedy stays ahead" argument's shape and apply it to interval scheduling.
3. State the exchange argument's shape and apply it to shortest-job-first.
4. Why does ratio-greedy work for fractional knapsack and fail for 0/1?
5. State the three matroid axioms and the intuition the exchange axiom captures.
6. State the cut property and what it proves.
7. Why does Dijkstra require non-negative weights? Identify the step that fails.
8. Give four acceptable uses of a greedy algorithm and the one unacceptable one.

## 7. Primary sources

- Kleinberg & Tardos, ch. 4. The best treatment of greedy proof technique in print.
- CLRS, 4th ed., chs. 15 (greedy, including matroids) and 21 (MST).
- Edmonds, "Matroids and the Greedy Algorithm" (1971).
- Huffman, "A Method for the Construction of Minimum-Redundancy Codes" (1952). Two pages.
- Dijkstra, "A Note on Two Problems in Connexion with Graphs" (1959). Three pages, and worth
  reading for how differently algorithms were presented then.

---

**Previous:** [L02](L02-divide-and-conquer.md) · **Next:**
[L04 — Dynamic Programming](L04-dynamic-programming.md)
