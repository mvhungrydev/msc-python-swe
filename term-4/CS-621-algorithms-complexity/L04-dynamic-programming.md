# CS-621 · Lesson 04 — Dynamic Programming

**Estimated study time:** 5 hours
**Prerequisites:** L01–L03

---

## 1. Orientation

Dynamic programming has a reputation for being hard, and the reputation comes from how it is
usually taught: as a collection of tricks, each with a table you are shown how to fill.

It is not a collection of tricks. It is one idea:

> **When a recursive solution's subproblems overlap, compute each once and reuse it.**

That is the whole thing. Everything else — memoization versus tabulation, the order of
iteration, reconstructing the solution, reducing space — is mechanical once you have found
the recurrence. And finding the recurrence is a skill with a procedure, which this lesson
gives you.

The strategic value: DP converts exponential brute force into polynomial time for an enormous
class of problems, and it is the standard tool when greedy fails because choices are not
independent (L03 §2.3).

## 2. Theory

### 2.1 The two conditions

DP applies when a problem has:

**1. Optimal substructure.** An optimal solution contains optimal solutions to subproblems.
Formally: the optimal value of the whole can be written as a function of optimal values of
strictly smaller subproblems.

**2. Overlapping subproblems.** The naive recursion solves the same subproblem many times.

Without (1), the recurrence is invalid. Without (2), you have divide and conquer (L02) —
recursion with no reuse to exploit.

The counterexample worth knowing for (1): **longest simple path in a graph does not have
optimal substructure.** The longest simple path from a to c through b is not the longest a→b
path followed by the longest b→c path — they might share vertices, violating simplicity. This
is why longest path is NP-hard while shortest path is easy, and it is the sharpest
illustration of what optimal substructure buys.

### 2.2 The procedure

Five steps, in this order, every time:

**1. Define the subproblem in words.** Precisely. *"Let `OPT(i)` be the maximum value
obtainable using only the first i items."* Most DP failures are failures at this step: a
vague subproblem produces a recurrence you cannot verify.

**2. Write the recurrence.** Express `OPT(i)` in terms of strictly smaller subproblems.
Usually by case analysis on the last decision: *what were the possible final choices, and
what does each leave behind?*

**3. State the base cases.** Small enough to answer directly.

**4. Determine the evaluation order.** Every subproblem must be computed before it is used.
Either memoize (top-down, and the order takes care of itself) or tabulate (bottom-up, and you
must get the loop order right).

**5. Reconstruct the solution**, if you need the actual choices and not just the value. Either
store back-pointers, or re-derive by comparing values.

The complexity is then, almost always: **(number of subproblems) × (time per subproblem)**.
That formula is worth internalizing because it tells you immediately whether a DP formulation
will be fast enough — before you write it.

### 2.3 Memoization versus tabulation

```python
from functools import cache

@cache
def fib(n: int) -> int:                       # top-down
    return n if n < 2 else fib(n-1) + fib(n-2)

def fib_bottom_up(n: int) -> int:             # bottom-up
    a, b = 0, 1
    for _ in range(n):
        a, b = b, a + b
    return a
```

| | Memoization (top-down) | Tabulation (bottom-up) |
|---|---|---|
| Order | automatic | you must get it right |
| Computes | only reachable subproblems | all of them |
| Overhead | recursion + dict | array indexing |
| Space optimization | hard | easy (rolling arrays) |
| Stack depth | can overflow (PY-602/PY-601: no TCO, limit 1,000) | none |
| Debugging | natural — it reads like the recurrence | requires reading the loops |

**Write it memoized first.** It is a direct transcription of the recurrence, so it is much
easier to get right, and `functools.cache` makes it two lines. Convert to tabulation only if
you need the space optimization or hit the recursion limit — and then verify the tabulated
version against the memoized one on random inputs.

The caveat: `functools.cache` on a method retains every `self` forever (PY-501 L09 §4). For a
DP inside a class, memoize a module-level function or use an explicit dict.

### 2.4 The canonical problems and what each teaches

**Fibonacci.** Θ(2ⁿ) → Θ(n). The existence proof.

**Weighted interval scheduling.** Intervals with values; select non-overlapping to maximize
value. (Greedy fails here — that is the point of pairing it with L03.)

Subproblem: `OPT(j)` = max value using intervals 1..j (sorted by finish time).
Recurrence: `OPT(j) = max(v_j + OPT(p(j)), OPT(j−1))` where `p(j)` is the last interval
compatible with j.
Θ(n log n) including the sort and the binary searches for p.

**0/1 knapsack.** `OPT(i, w)` = max value using the first i items with capacity w.
`OPT(i, w) = max(OPT(i−1, w), v_i + OPT(i−1, w − w_i))` if `w_i ≤ w`, else `OPT(i−1, w)`.
Θ(nW) time and space; space reducible to Θ(W) with a rolling array, **iterating w downward**
so that each item is used at most once. (Iterating upward gives unbounded knapsack — a
one-character difference with a completely different meaning, and a classic bug.)

**Note that Θ(nW) is pseudo-polynomial**, not polynomial: W is a *value*, and its encoding
length is log W. This is the distinction that L08 makes precise, and knapsack is the standard
example.

**Edit distance (Levenshtein).** `OPT(i, j)` = distance between prefixes of length i and j.
`OPT(i,j) = OPT(i−1,j−1)` if the characters match, else `1 + min(OPT(i−1,j), OPT(i,j−1),
OPT(i−1,j−1))`. Θ(mn). This is the algorithm behind `difflib`, spell checkers, DNA alignment
(Needleman–Wunsch is the same recurrence with weights), and fuzzy matching.

**Longest common subsequence.** Nearly the same recurrence. The basis of `diff`, though real
diff tools use Myers' algorithm, which is faster on the typical case where the sequences are
similar.

**Matrix chain multiplication.** `OPT(i, j)` = minimum scalar multiplications for the product
`A_i…A_j`. `OPT(i,j) = min over k of OPT(i,k) + OPT(k+1,j) + cost of the final multiply`.
Θ(n³) with Θ(n²) subproblems and Θ(n) work each — the clearest example of the
subproblems × work formula.

**Bellman–Ford shortest paths.** `OPT(i, v)` = shortest path to v using at most i edges.
Θ(mn). Handles negative weights, and detects negative cycles (if anything improves on
iteration n, there is one). This is the DP that Dijkstra's greedy cannot do (L03 §2.5).

**Floyd–Warshall all-pairs shortest paths.** `OPT(k, i, j)` = shortest i→j path using only
intermediates from `{1..k}`. Θ(n³), five lines of code, and one of the most elegant
recurrences in the subject.

**Sequence alignment, TSP over subsets (Held–Karp, Θ(2ⁿn²)), optimal BSTs, subset sum, coin
change, rod cutting, maximum subarray (Kadane), longest increasing subsequence** (Θ(n²) by DP,
Θ(n log n) with a patience-sorting insight).

### 2.5 Recognizing DP

The signals, in rough order of reliability:

- **You wrote a recursion and it recomputes.** The strongest signal. Add `@cache` and measure.
- **"Maximize/minimize/count over all ways to …"** — an optimization or counting problem over
  an exponential space.
- **A sequence of decisions**, where each affects what remains available.
- **Greedy fails** because a locally good choice forecloses a better structure (L03 §2.3).
- **Small state.** The subproblem can be described by a few small parameters — an index, a
  remaining capacity, a position in two sequences, a subset (if n ≤ ~20).

And when DP does *not* apply:

- **No optimal substructure** (longest simple path).
- **State too large.** If the subproblem needs the full set of chosen items, you have 2ⁿ
  states and DP buys nothing.
- **Continuous state**, unless discretized.

### 2.6 Space optimization and reconstruction

**Rolling arrays.** If `OPT(i, ·)` depends only on `OPT(i−1, ·)`, keep two rows — or one, with
careful iteration order. Knapsack goes from Θ(nW) to Θ(W).

**Reconstruction.** Once you optimize space you can no longer walk back through the table.
Options:

- Keep back-pointers (costs the space you just saved).
- Re-derive: at each step, recompute which choice achieved the optimum. Costs time, not
  space.
- **Hirschberg's algorithm**: divide and conquer *on top of* DP, computing edit distance in
  Θ(mn) time and **Θ(min(m,n)) space** while still recovering the alignment. A genuinely
  beautiful algorithm and worth implementing once.

### 2.7 DP in practice

Where you will actually meet it:

- **Diff and merge** — LCS/edit distance.
- **Spell checking and fuzzy search** — edit distance, often with a trie and a bounded
  threshold.
- **Query optimization** — a database's join-order search is DP over subsets of relations
  (System R's classic algorithm; DI-721 L02).
- **Resource allocation and scheduling** — knapsack variants.
- **Sequence alignment** in bioinformatics.
- **Viterbi** — the most likely state sequence in an HMM; still used in decoding, speech, and
  parts of NLP.
- **Reinforcement learning** — value iteration and policy iteration are DP over states.
- **Text layout** — Knuth–Plass line breaking, which is why TeX's paragraphs look better than
  a greedy line-breaker's.

That last one is a good illustration: greedy line breaking (fill each line as much as
possible) is what most word processors do; DP over the whole paragraph is what TeX does, and
the difference is visible.

## 3. Construction: from recursion to DP

**Exercise A — the procedure, five times.** For each problem, write out all five steps of
§2.2 *before* writing code, then implement memoized, then tabulated, then space-optimized,
then with reconstruction:

1. **Weighted interval scheduling.**
2. **0/1 knapsack.**
3. **Edit distance** (with the actual edit script reconstructed).
4. **Longest increasing subsequence** (Θ(n²) first, then the Θ(n log n) version — and explain
   what changed).
5. **Coin change** (minimum coins, and count of ways — note these are *different*
   recurrences and it is instructive to see why).

For each, verify the tabulated version against the memoized one on 10,000 random inputs.

**Exercise B — a problem from your own domain.** Find one. Candidates: assigning tasks to
time slots with dependencies; choosing which caches to warm within a time budget; segmenting
a log stream into episodes; picking a subset of tests to run within a time limit maximizing
expected defect detection.

Apply the five steps. Be honest about whether the state is small enough — most real problems
have too much state, and *discovering that* is a legitimate and valuable outcome. Report the
state space size and what you would do instead (L09).

**Exercise C — the exponential-to-polynomial demonstration.** Take a naive recursive solution
that recomputes (weighted interval scheduling, or LCS) and measure its runtime at n = 15, 20,
25, 30. Fit the growth. Then add `@cache` — one line — and measure at n = 100, 1000, 10000.
Plot both. This is the most vivid demonstration in the course of what an algorithmic
improvement means, and it takes twenty minutes.

**Exercise D — Hirschberg.** Implement edit distance with alignment reconstruction two ways:
the full Θ(mn) table, and Hirschberg's Θ(min(m,n)) space version. Verify they produce the
same alignment. Measure peak memory for both on sequences of length 10⁴ (the full table is
10⁸ cells — measure how that goes) and report the crossover where the space saving matters.

**Exercise E — the state-space explosion.** Take a problem where the natural DP state is a
*subset* (TSP, set cover, or a job-assignment problem). Implement the Θ(2ⁿ·poly) bitmask DP.
Measure the largest n you can solve in one minute. Report it, and note the wall: DP over
subsets is fine to about n = 20–25 and impossible beyond. That wall is where L09's coping
strategies begin.

## 4. Failure modes

- **Vague subproblem definition.** The root cause of most wrong DP.
- **A recurrence that references an equal-or-larger subproblem.** Infinite recursion or a
  wrong answer.
- **Wrong iteration order** in tabulation. The knapsack ascending/descending bug is the
  classic: one direction is 0/1, the other is unbounded.
- **Missing base case**, or a base case that is wrong for an edge input (empty string, n = 0).
- **`@cache` on a method**, retaining every instance.
- **Unhashable arguments** to `@cache`. Lists must become tuples — and converting a list to a
  tuple on every call can dominate the runtime.
- **Recursion limit** on memoized DP with deep chains. Convert to tabulation or raise the
  limit deliberately.
- **Confusing pseudo-polynomial with polynomial.** Knapsack's Θ(nW) is exponential in the
  input *size*.
- **State too large**, discovered after implementing.
- **Optimizing space, then needing the reconstruction.** Decide first.
- **Applying DP where greedy is provably optimal.** Slower and more code for the same answer.

## 5. Exercises

### Warm-up (30 min)

**W1.** Complete Exercise C. Report the measured growth rate of the naive version and the
memoized one.

**W2.** Write the five-step procedure for the rod-cutting problem, then implement it in both
directions.

**W3.** Demonstrate the knapsack iteration-order bug: implement the rolling-array version
with ascending w and show it solves *unbounded* knapsack instead. Explain in one sentence.

### Core (3 h)

**C1 — The five problems.** Complete Exercise A. Deliverable: for each, the five written
steps, four implementations, the cross-verification, and the complexity derived from
(subproblems × work per subproblem) matched against measurement.

**C2 — Your own problem.** Complete Exercise B. Deliverable: the five steps, the state-space
size calculation, the implementation if feasible, and — if not feasible — a clear statement of
why with the alternative you would use.

**C3 — Hirschberg.** Complete Exercise D. Report peak memory for both, the crossover, and the
runtime cost of the space saving (it is roughly 2×, and knowing that trade is the point).

**C4 — The subset wall.** Complete Exercise E. Report the largest n solvable in one minute and
in one hour, and extrapolate to n = 40 and n = 60. Then write 300 words on what this implies
for how you would respond to a request to "just solve it optimally" for n = 100.

### Challenge

**X1.** Implement the System R join-order DP: given n relations with sizes and join
selectivities, find the optimal left-deep join order in Θ(2ⁿ·n²). Measure the largest n you
can handle. Then read about how real optimizers cope beyond that (heuristics, genetic
algorithms, or restricting the search space) and write 800 words connecting it to L09.
Return to this after DI-721 L02.

**X2.** Implement Knuth–Plass optimal line breaking for a paragraph, and a greedy line
breaker. Typeset the same paragraph with both and compare visually. Then measure the DP's
complexity and explain why TeX can afford it. Write 600 words on when the extra quality
justifies the algorithm — a question that is really about product, not computation.

## 6. Self-check

1. State the two conditions for DP and give a problem lacking each.
2. Give the five-step procedure, and say which step most failures occur at.
3. Give the complexity formula for a DP and apply it to matrix chain multiplication.
4. Compare memoization and tabulation on five dimensions, and say which to write first.
5. Explain the knapsack rolling-array iteration order and what the wrong direction computes.
6. What is pseudo-polynomial, and why is knapsack's Θ(nW) not polynomial?
7. What does Hirschberg's algorithm achieve and how?
8. Give five signals that a problem is a DP problem, and three that it is not.

## 7. Primary sources

- Kleinberg & Tardos, ch. 6. The best DP chapter in print; its "five steps" framing is where
  §2.2 comes from.
- Erickson, *Algorithms*, ch. 3 — an excellent alternative treatment, free online, with a
  strong emphasis on getting the recurrence right before coding.
- Bellman, *Dynamic Programming* (1957) — and the story of why it is called that, which is
  worth knowing.
- Hirschberg, "A Linear Space Algorithm for Computing Maximal Common Subsequences" (1975).
- Knuth & Plass, "Breaking Paragraphs into Lines" (1981).
- Selinger et al., "Access Path Selection in a Relational Database Management System"
  (SIGMOD 1979) — the System R optimizer.

---

**Previous:** [L03](L03-greedy-algorithms.md) · **Next:**
[L05 — Graph Algorithms and Modelling](L05-graph-algorithms.md)
