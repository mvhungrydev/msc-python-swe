# CS-621 · Lesson 02 — Divide and Conquer, and Recurrences

**Estimated study time:** 4 hours
**Prerequisites:** L01

---

## 1. Orientation

Divide and conquer is the first genuine *design technique* in this course: a way of
generating algorithms rather than a catalogue of them.

The pattern is three steps — **divide** the problem into subproblems, **conquer** them
recursively, **combine** the results — and the interesting question is always the same: *how
much work does the combine step do?* That question determines everything, and the Master
Theorem is just a compact answer to it.

The payoff is knowing when the technique applies. Mergesort, quicksort, FFT, Strassen,
closest pair, and — less obviously — MapReduce, parallel reductions, B-tree operations, and
every "split the range and recurse" algorithm are all the same shape.

## 2. Theory

### 2.1 The pattern

```
solve(P):
    if |P| is small: return brute_force(P)
    split P into a subproblems of size n/b
    solve each recursively
    combine the results
```

Cost: `T(n) = a·T(n/b) + f(n)` where `f(n)` is the divide-plus-combine cost.

Three quantities, and the whole analysis is about their balance:

- **a** — how many subproblems.
- **b** — by what factor the size shrinks.
- **f(n)** — the non-recursive work.

The recursion tree has depth `log_b n`, level i has `a^i` nodes each of size `n/b^i`, and the
work at level i is `a^i · f(n/b^i)`. **Whether the total is dominated by the root, the
leaves, or spread evenly is the entire question**, and it is worth drawing the tree at least
five times until that becomes obvious rather than memorized.

### 2.2 Solving recurrences: three methods

**Method 1 — recursion tree.** Draw it. Sum each level. Sum the levels.

For `T(n) = 2T(n/2) + n` (mergesort): level i has `2^i` nodes of size `n/2^i`, so work per
level is `2^i · (n/2^i) = n`. Depth `log₂ n`. Total `n log n`. Θ(n log n).

For `T(n) = 2T(n/2) + n²`: level i does `2^i (n/2^i)² = n²/2^i`. The sum is
`n²(1 + ½ + ¼ + …) < 2n²`. **Root-dominated**: Θ(n²).

For `T(n) = 4T(n/2) + n`: level i does `4^i (n/2^i) = 2^i n`. This *grows*; the last level
dominates. Leaves: `4^(log₂ n) = n²`. **Leaf-dominated**: Θ(n²).

Those three examples are the three cases. Draw them; the pattern is visible.

**Method 2 — substitution (guess and verify by induction).** Guess a bound, prove it by
strong induction, being careful to prove the *exact* statement (a common error is proving
`T(n) ≤ cn` from `T(n) ≤ cn + n`, which does not follow — you must strengthen the hypothesis).

> **Claim.** `T(n) = 2T(n/2) + n` with `T(1) = 1` satisfies `T(n) ≤ cn log₂ n + n`.
> **Proof.** Base: `T(1) = 1 ≤ 1`. Step: assume for n/2.
> `T(n) = 2T(n/2) + n ≤ 2(c(n/2)log₂(n/2) + n/2) + n = cn log₂ n − cn + 2n`.
> This is `≤ cn log₂ n + n` when `c ≥ 1`. ∎

**Method 3 — the Master Theorem.** For `T(n) = a·T(n/b) + f(n)` with a ≥ 1, b > 1:

Let `c* = log_b a` (the "critical exponent" — the leaf count exponent).

- **Case 1 (leaf-dominated).** If `f(n) = O(n^(c*−ε))` for some ε > 0, then
  `T(n) = Θ(n^c*)`.
- **Case 2 (balanced).** If `f(n) = Θ(n^c* · log^k n)` for k ≥ 0, then
  `T(n) = Θ(n^c* · log^(k+1) n)`.
- **Case 3 (root-dominated).** If `f(n) = Ω(n^(c*+ε))` for some ε > 0, **and** the
  regularity condition `a·f(n/b) ≤ c·f(n)` holds for some c < 1 and large n, then
  `T(n) = Θ(f(n))`.

Worked:

| Recurrence | c* = log_b a | f(n) | Case | Result |
|---|---|---|---|---|
| `2T(n/2) + n` | 1 | n | 2 (k=0) | Θ(n log n) |
| `2T(n/2) + n²` | 1 | n² | 3 | Θ(n²) |
| `4T(n/2) + n` | 2 | n | 1 | Θ(n²) |
| `7T(n/2) + n²` | log₂7 ≈ 2.807 | n² | 1 | Θ(n^2.807) — Strassen |
| `T(n/2) + 1` | 0 | 1 | 2 (k=0) | Θ(log n) — binary search |
| `2T(n/2) + n log n` | 1 | n log n | 2 (k=1) | Θ(n log² n) |
| `T(2n/3) + 1` | 0 | 1 | 2 | Θ(log n) |

**When it does not apply**, and you must use a tree or substitution:

- Unequal subproblem sizes: `T(n) = T(n/3) + T(2n/3) + n` → Θ(n log n), by tree.
- Subtractive: `T(n) = T(n−1) + n` → Θ(n²). (The Master Theorem needs division.)
- `f` not polynomially comparable to `n^c*`: `T(n) = 2T(n/2) + n/log n` falls between cases 1
  and 2. (Answer: Θ(n log log n), by tree.)
- The regularity condition fails in case 3.

The Akra–Bazzi method handles unequal splits generally; know it exists.

### 2.3 The classic algorithms, and what each teaches

**Mergesort.** `T(n) = 2T(n/2) + Θ(n)` → Θ(n log n). Stable, Θ(n) extra space, and — the
reason it matters at scale — it works on sequential access, which makes it the basis of
**external sorting** for data larger than memory (DI-721 L03).

**Quicksort.** Partition around a pivot, recurse on both sides. `T(n) = T(k) + T(n−k−1) + Θ(n)`.

- Balanced: Θ(n log n).
- Worst case (sorted input, first-element pivot): Θ(n²).
- **Randomized pivot: Θ(n log n) expected**, and the analysis is a lovely one (L06).

In practice quicksort beats mergesort by a constant because it is in-place and cache-friendly
(PY-602 L06). Real implementations are **introsort**: quicksort, switching to heapsort when
recursion gets too deep (bounding the worst case) and to insertion sort for small subarrays
(bounding the constant). CPython's `list.sort` is Timsort — mergesort exploiting existing
runs, Θ(n) on nearly-sorted input, which is extremely common in practice.

**Binary search.** `T(n) = T(n/2) + Θ(1)` → Θ(log n). The technique generalizes far beyond
sorted arrays: **binary search on the answer**. If you can test "is the answer ≤ x?" in
polynomial time and the predicate is monotone, you can binary search x. This turns many
optimization problems into decision problems, and it is the most reusable trick in this
lesson.

**Karatsuba multiplication.** Naive n-digit multiplication is Θ(n²). Karatsuba splits each
number in half and observes that the three products it needs can be computed with **three**
multiplications instead of four:
`(a·10^m + b)(c·10^m + d) = ac·10^2m + ((a+d)(b+c) − ac − bd)·10^m + bd`.
So `T(n) = 3T(n/2) + Θ(n)` → `Θ(n^log₂3) = Θ(n^1.585)`. CPython uses Karatsuba for large
integers.

**Strassen's matrix multiplication.** The same trick: 7 multiplications instead of 8.
`T(n) = 7T(n/2) + Θ(n²)` → `Θ(n^2.807)`. Rarely used in practice — worse constants, worse
numerical stability, and worse cache behaviour than a well-tiled Θ(n³) (PY-602 L06 §2.5) —
but it is the proof that Θ(n³) is not a lower bound, and it opened a research programme still
running.

**Closest pair of points.** Θ(n log n) by divide and conquer, where the combine step is the
clever part: after solving both halves with minimum distance δ, only points within δ of the
dividing line matter, and sorted by y-coordinate each needs comparison with at most 7
successors. Worth working through as an example of a combine step that requires an insight.

**Fast Fourier Transform.** `T(n) = 2T(n/2) + Θ(n)` → Θ(n log n) for what is naively Θ(n²).
Turns polynomial multiplication and convolution into pointwise multiplication. The most
consequential divide-and-conquer algorithm in existence.

### 2.4 Why the combine step is the design decision

Given a and b, the algorithm's complexity is determined by whether f(n) is above, at, or
below `n^log_b a`. So algorithm design by divide and conquer is largely **the design of the
combine step**, and there are two moves:

**Reduce a.** Karatsuba and Strassen both do this: find algebraic identities that need fewer
subproblems. This changes the critical exponent and is the big lever.

**Reduce f.** Make the combine cheaper. The closest-pair algorithm's insight (only 7
neighbours matter) is exactly this — it keeps the combine at Θ(n) instead of Θ(n²).

A third move, which is a design smell when needed: **change b.** Splitting into more or fewer
pieces rarely helps because `log_b a` moves both ways.

### 2.5 Divide and conquer as a parallel pattern

The independence of subproblems means divide and conquer parallelizes naturally:

- **Fork-join.** Recurse in parallel, join, combine. Directly maps to a thread pool or
  `ProcessPoolExecutor` (PY-601 L08).
- **MapReduce** is one level of divide and conquer with a fixed combine (DI-721 L06).
- **Parallel reductions and scans** — sum, max, prefix-sum — are divide and conquer with an
  associative combine.

The requirement is that the combine operation be **associative** (and for some schemes,
commutative). That is a constraint on the *problem*, not the implementation, and recognizing
when your combine is associative is how you know whether a computation can be distributed at
all.

The cost model changes: the recursion depth becomes the *span* (critical path), and
`work / span` bounds the available parallelism. For mergesort, work is Θ(n log n) and span is
Θ(log² n) with a parallel merge, so parallelism is Θ(n / log n) — plenty.

Practical caveat: parallelize only above a **grain-size threshold**. Below it, task overhead
exceeds the work, and the standard structure is "if n < threshold, do it sequentially".

### 2.6 Recursion in Python, practically

- **The recursion limit** is 1,000 by default (`sys.setrecursionlimit`). A divide-and-conquer
  algorithm with depth `log n` is fine forever; one with depth n (naive quicksort on sorted
  input, or a linked-list traversal) is not.
- **CPython does not do tail-call optimization** and will not. Deep recursion must be
  converted to iteration with an explicit stack — which is exactly what quicksort
  implementations do for the larger partition, recursing only on the smaller one to bound
  depth at `log n`.
- **Function calls are expensive** (PY-602 L04 §2.5: 30–60 ns). For small subproblems the
  recursion overhead dominates, which is why the grain-size threshold matters even
  sequentially.
- **Memoization** turns exponential recursion into polynomial (L04). `functools.cache`, with
  the leak caveat of PY-501 L09 §4.

## 3. Construction: designing by divide and conquer

**Exercise A — solve these recurrences**, by whichever method applies, showing your work:

1. `T(n) = 2T(n/2) + n³`
2. `T(n) = 16T(n/4) + n²`
3. `T(n) = 2T(n/2) + n/log n`
4. `T(n) = 2T(√n) + log n`  *(hint: substitute `m = log n`)*
5. `T(n) = T(n−1) + 1/n`
6. `T(n) = T(n/2) + T(n/4) + T(n/8) + n`
7. `T(n) = 3T(n/3) + n/2`
8. `T(n) = T(n/2) + n^0.5`

**Exercise B — inversions.** Count inversions in an array (pairs i < j with a[i] > a[j]) in
Θ(n log n). Naive is Θ(n²). The insight is that the merge step of mergesort can count
inversions between the halves for free. Implement it, prove the bound, and verify against the
brute force for n ≤ 500 with random arrays.

**Exercise C — maximum subarray.** Find the contiguous subarray with the largest sum, three
ways: brute force Θ(n²), divide and conquer Θ(n log n) (the combine step handles subarrays
crossing the midpoint), and Kadane's algorithm Θ(n). Implement all three, verify they agree
on 10,000 random inputs, and measure. Then answer: **why does the Θ(n) algorithm exist at
all — what structural property does it exploit that divide and conquer does not?** (The
answer is the seed of dynamic programming, L04.)

**Exercise D — binary search on the answer.** A classic: given n files with sizes and k
machines, partition the files (preserving order) to minimize the maximum load on any machine.

- The decision version — "can we do it with maximum load ≤ x?" — is a greedy Θ(n) check.
- The predicate is monotone in x.
- So binary search x over `[max(sizes), sum(sizes)]`: Θ(n log(sum)).

Implement it. Then find two more problems in your own experience that have this shape (the
signature is: an optimization problem whose decision version is easy and monotone) and solve
one.

**Exercise E — parallel.** Take your mergesort and parallelize it with
`ProcessPoolExecutor` (PY-601 L08). Measure speedup against worker count and against the
grain-size threshold. Report the two-dimensional table, find the optimum, and explain both
edges: too small a threshold and overhead dominates; too large and parallelism is lost.

## 4. Failure modes

- **Applying the Master Theorem where it does not apply** — unequal splits, subtractive
  recurrences, or f not polynomially comparable.
- **Forgetting the regularity condition** in case 3.
- **Substitution with a hypothesis that is not strong enough**, "proving" `T(n) ≤ cn` from
  `T(n) ≤ cn + n`.
- **Quicksort with a deterministic pivot** on sorted input: Θ(n²) and Θ(n) stack depth.
- **Recursing on the larger partition first**, blowing the stack.
- **No grain-size threshold**, so the constant factor dominates for small subproblems.
- **Parallelizing a non-associative combine.**
- **Assuming a lower asymptotic always wins.** Strassen is usually slower than tiled Θ(n³).
- **Ignoring the extra space.** Mergesort's Θ(n) is often the reason quicksort is chosen.
- **Recursion depth in Python.** No TCO; the limit is 1,000.

## 5. Exercises

### Warm-up (30 min)

**W1.** Draw the recursion tree for `2T(n/2) + n`, `2T(n/2) + n²`, and `4T(n/2) + n`. Label
the work at each level and identify which of root, leaves, or all-levels dominates.

**W2.** Solve Exercise A items 1, 2, 7 with the Master Theorem, stating the case and c*.

**W3.** Demonstrate quicksort's Θ(n²) worst case with a deterministic pivot, and show that a
random pivot fixes it, by measurement at n = 10⁴.

### Core (2.5 h)

**C1 — Recurrences.** Complete Exercise A, all eight, on paper, stating the method used and
why the Master Theorem does or does not apply to each.

**C2 — Inversions and maximum subarray.** Complete Exercises B and C. Deliverable: all
implementations, the correctness verification against brute force, timing at three input
sizes confirming the predicted growth, and a 300-word answer to Exercise C's closing
question.

**C3 — Binary search on the answer.** Complete Exercise D, including finding two problems
from your own experience with the same shape and solving one. Report the shape signature you
now recognize.

**C4 — Parallel mergesort.** Complete Exercise E. Report the two-dimensional table, the
optimum, the speedup at the optimum, and the span/work analysis predicting the maximum
achievable parallelism. Compare the prediction with the measurement and explain the gap.

### Challenge

**X1.** Implement Karatsuba multiplication for big integers and benchmark against CPython's
built-in `int` multiplication across sizes from 10 to 10⁶ digits. Find the crossover where
Karatsuba beats the schoolbook method, and find where CPython switches (read
`Objects/longobject.c` — the constant is named). Then implement a Toom–Cook or FFT-based
multiplication and find the next crossover. Report the three-way plot.

**X2.** Implement the closest-pair-of-points algorithm in Θ(n log n) and prove the "at most 7
successors" bound geometrically. Compare with the Θ(n²) brute force and with a spatial-index
approach (a k-d tree or grid). Report where each wins as a function of n and of the point
distribution — the answer depends on the distribution, which is itself the lesson.

## 6. Self-check

1. Give the divide-and-conquer recurrence form and say what a, b, and f mean.
2. For each of the three Master Theorem cases, give a recurrence and say whether root,
   leaves, or all levels dominate.
3. State the regularity condition and why case 3 needs it.
4. Give three recurrences the Master Theorem cannot solve, and the method for each.
5. Explain Karatsuba's trick and the resulting exponent.
6. What is "binary search on the answer" and what two conditions does it need?
7. What property must a combine step have to parallelize, and why?
8. Why does real quicksort recurse on the smaller partition?

## 7. Primary sources

- Kleinberg & Tardos, ch. 5.
- CLRS, 4th ed., chs. 2 and 4 (including the Akra–Bazzi section).
- Karatsuba & Ofman (1962); Strassen, "Gaussian Elimination is not Optimal" (1969).
- Cooley & Tukey, "An Algorithm for the Machine Calculation of Complex Fourier Series"
  (1965).
- Peters, the Timsort description in CPython's `Objects/listsort.txt` — an unusually good
  piece of engineering writing.

---

**Previous:** [L01](L01-asymptotics-models-amortized.md) · **Next:**
[L03 — Greedy Algorithms and Exchange Arguments](L03-greedy-algorithms.md)
