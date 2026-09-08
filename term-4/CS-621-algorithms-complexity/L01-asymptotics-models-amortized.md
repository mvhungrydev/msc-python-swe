# CS-621 · Lesson 01 — Asymptotics, Models, and Amortized Analysis

**Estimated study time:** 4 hours
**Prerequisites:** `appendices/mathematical-notation.md` if set-builder notation and
quantifiers are unfamiliar

---

## 1. Orientation

Everyone knows "big-O". Fewer can prove a bound from the definition, and fewer still can say
what the definition actually claims — which matters, because the informal version supports
false inferences.

Two things this lesson fixes:

- **Big-O is an upper bound, not a description.** `n = O(n²)` is *true*. Saying "this
  algorithm is O(n²)" tells you nothing about whether it is also O(n log n). To make a claim
  about the algorithm's actual growth you need Θ, and most people say O when they mean Θ.
- **Amortized is not average-case.** They are different guarantees with different assumptions,
  and confusing them leads to trusting a bound that does not hold for your workload.

## 2. Theory

### 2.1 The definitions

Let f, g be functions from ℕ to ℝ⁺.

**O (upper bound).** `f(n) = O(g(n))` iff ∃ c > 0, n₀ ∈ ℕ such that ∀ n ≥ n₀: `f(n) ≤ c·g(n)`.

**Ω (lower bound).** `f(n) = Ω(g(n))` iff ∃ c > 0, n₀ such that ∀ n ≥ n₀: `f(n) ≥ c·g(n)`.

**Θ (tight bound).** `f(n) = Θ(g(n))` iff `f = O(g)` and `f = Ω(g)`.

**o (strictly smaller).** `f(n) = o(g(n))` iff ∀ c > 0 ∃ n₀ ∀ n ≥ n₀: `f(n) < c·g(n)`.
Equivalently `lim f/g = 0`.

**ω (strictly larger).** The dual of o.

Note the quantifier structure: O has "∃c ∀n", o has "∀c ∃n₀". That difference is the whole
difference between "at most, up to a constant" and "eventually negligible compared to".

The `=` is an abuse of notation — it means ∈, since O(g) is a *set* of functions. Hence
`n = O(n²)` is fine and `O(n²) = n` is meaningless. Write `f ∈ O(g)` if the abuse bothers
you; everyone writes `=`.

**Proving a bound.** Two lines, and you should be able to do it fluently:

> **Claim.** `3n² + 5n + 2 = O(n²)`.
> **Proof.** For n ≥ 1: `3n² + 5n + 2 ≤ 3n² + 5n² + 2n² = 10n²`. Take c = 10, n₀ = 1. ∎

> **Claim.** `n log n ≠ O(n)`.
> **Proof.** Suppose `n log n ≤ cn` for all n ≥ n₀. Then `log n ≤ c` for all n ≥ n₀, which
> fails at `n = 2^(c+1)`. Contradiction. ∎

**Useful facts, worth internalizing:**

- `log_a n = Θ(log_b n)` for any bases — logarithm bases vanish into the constant, which is
  why we write `log n` without a base.
- `n^a = o(n^b)` for a < b; `(log n)^k = o(n^ε)` for every k and every ε > 0; `n^k = o(c^n)`
  for c > 1; `c^n = o(n!)`; `n! = o(n^n)`.
- Polynomial beats polylogarithmic; exponential beats polynomial; factorial beats
  exponential. Always, eventually.

### 2.2 What asymptotics hide

Three things, and each has bitten someone:

**Constants.** `1000n` versus `n²`: the linear algorithm wins only above n = 1000. Below
that, the "worse" algorithm is better. Real examples: insertion sort beats merge sort for
n ≲ 32 (which is why Timsort switches to insertion sort for small runs); linear scan beats a
hash table for small collections because of hashing and cache behaviour.

**Which operations you counted.** A "comparison-based" bound counts comparisons; if your
comparison is a string compare on long strings, the model is wrong for your case. Always
state the unit.

**The machine.** Asymptotic analysis in the RAM model treats every memory access as O(1),
which — after PY-602 L06 — you know is false by a factor of 250 between L1 and DRAM. Two
Θ(n) algorithms can differ by 10× from locality alone. The **external memory** and
**cache-oblivious** models exist to capture this and are the right models for large data.

The correct posture: **asymptotics tell you how the cost scales; measurement tells you what
it is.** You need both, and PY-602 L01 §2.8 puts the algorithm check *before* the
measurement precisely because a better asymptotic beats every constant factor eventually,
and "eventually" is often at production scale.

### 2.3 Models of computation

Analysis is always relative to a model, and being sloppy about the model is how bogus bounds
arise.

**RAM (random access machine).** The default. Unit-cost arithmetic on words, unit-cost memory
access, a word large enough to index the input (typically O(log n) bits). Assumptions worth
noticing: adding two numbers is O(1) *only if the numbers fit in a word* — an algorithm on
big integers must count bit operations.

**Comparison model.** Only comparisons count. Gives the Ω(n log n) sorting lower bound, and
that lower bound is *only about this model* — radix sort is Θ(n) and does not contradict it,
because it does not compare.

**External memory model.** Memory of size M, transfers in blocks of size B. Cost is the
number of block transfers. This is the model that explains why B-trees exist (DI-721 L02) and
why cache-friendly layouts win (PY-602 L06).

**Cell-probe, decision tree, circuit, Turing machine** — for lower bounds and complexity
theory (L08, L10).

### 2.4 The sorting lower bound

Worth doing once because it is the cleanest lower-bound argument you will meet.

**Claim.** Any deterministic comparison-based sorting algorithm makes Ω(n log n) comparisons
in the worst case.

**Proof.** Model the algorithm as a binary decision tree: each internal node is a comparison,
each branch an outcome, each leaf a permutation. Correctness requires a distinct leaf for
each of the n! possible input orderings, so the tree has ≥ n! leaves. A binary tree of height
h has ≤ 2^h leaves, so `2^h ≥ n!`, giving `h ≥ log₂(n!)`. By Stirling, `log₂(n!) = Θ(n log n)`.
The worst case is the tree's height. ∎

Note what the argument *assumes*: comparisons only, and deterministic. Both matter, and
noticing which assumption a lower bound rests on is how you find the way around it.

### 2.5 Amortized analysis

The question: what is the cost of a *sequence* of n operations, divided by n?

This is not average-case analysis. **Average-case averages over a distribution of inputs and
assumes that distribution. Amortized analysis is a worst-case bound over any sequence, with
no probabilistic assumption at all.** An amortized bound holds for an adversary; an
average-case bound does not.

Three methods, and you should be fluent in all three.

**Aggregate method.** Bound the total cost of n operations, divide by n.

*Dynamic array (Python's `list`).* `append` is O(1) unless the array is full, in which case
it allocates a new array of double the size and copies. Over n appends, the copies happen at
sizes 1, 2, 4, …, and total `1 + 2 + 4 + … + n < 2n`. So n appends cost O(n), and the
amortized cost per append is **O(1)**.

Crucially: **doubling is what makes this work.** With a growth factor of 1 (grow by a
constant c), the copies total `c + 2c + … + n = Θ(n²/c)`, so each append is amortized Θ(n) —
quadratic overall. Any factor > 1 gives O(1) amortized; the factor trades memory slack
against copy frequency. (CPython uses roughly 1.125, which is a memory-conscious choice, and
PY-602 L04 §2.2 notes the resulting slack.)

**Accounting method.** Charge each operation a fixed "amortized cost", storing the surplus as
credit on data-structure elements, and pay for expensive operations from stored credit.
Requires: credit never goes negative.

*Dynamic array.* Charge 3 per `append`: 1 pays for the insertion, 1 is stored on the new
element to pay for copying it at the next resize, and 1 is stored to pay for copying one of
the older elements that has already spent its credit. At a resize from size k to 2k, the k
elements each carry the credit needed to move themselves. Credit never goes negative, so the
amortized cost is 3 = O(1). ∎

**Potential method.** The most powerful and the one to master. Define a potential function
Φ mapping a data structure state to a non-negative real, with Φ(D₀) = 0. Define

> amortized cost of operation i := actual costᵢ + Φ(Dᵢ) − Φ(Dᵢ₋₁)

Then `Σ amortized = Σ actual + Φ(Dₙ) − Φ(D₀) ≥ Σ actual` (given Φ ≥ 0 and Φ(D₀) = 0), so the
sum of amortized costs bounds the total actual cost.

*Dynamic array.* Let `Φ = 2·(num_items) − (capacity)` when the array is at least half full,
0 otherwise. A non-resizing append: actual 1, Φ rises by 2, amortized 3. A resizing append
from size k (capacity k): actual k+1, Φ goes from `2k − k = k` to `2(k+1) − 2k = 2`, so ΔΦ =
`2 − k`, amortized = `k + 1 + 2 − k = 3`. Constant either way. ∎

The art is choosing Φ. The heuristic: **Φ should be high when the structure is "about to be
expensive"** — nearly full, unbalanced, or with much deferred work — so that the expensive
operation is paid for by the drop in potential.

### 2.6 Amortized guarantees and when they are not enough

An amortized O(1) `append` can still take Θ(n) on one call. That matters when:

- **You have a latency SLO.** A p999 latency requirement is violated by the resize, even
  though the average is fine. This is a real cause of tail latency (PY-602 L09 §2.3's
  variance argument).
- **Real-time systems.** A hard deadline cannot be met by an amortized bound.
- **The structure is shared** and the expensive operation holds a lock (PY-601 L03 §2.4).

The alternatives: **worst-case bounds** (a real-time deque, incremental rehashing), or
**deamortization** — spreading the expensive work across subsequent cheap operations. Most
databases and garbage collectors do this: incremental rehashing in Redis, incremental
compaction in LSM trees (DI-721 L02), incremental GC.

Say which guarantee you have. "Amortized O(1)" and "worst-case O(1)" are different products.

### 2.7 The complexities you must know cold

| Structure | Access | Search | Insert | Delete | Note |
|---|---|---|---|---|---|
| Array | Θ(1) | Θ(n) | Θ(n) | Θ(n) | |
| Dynamic array | Θ(1) | Θ(n) | Θ(1) amort. | Θ(n) | append only for the insert bound |
| Linked list | Θ(n) | Θ(n) | Θ(1)* | Θ(1)* | *given the node |
| Hash table | — | Θ(1) avg | Θ(1) avg | Θ(1) avg | Θ(n) worst case |
| Balanced BST | Θ(log n) | Θ(log n) | Θ(log n) | Θ(log n) | ordered |
| Binary heap | Θ(1) min | Θ(n) | Θ(log n) | Θ(log n) | |
| Trie | Θ(k) | Θ(k) | Θ(k) | Θ(k) | k = key length |
| B-tree | Θ(log_B n) | | | | in block transfers |

And Python's, which you should know because you use them daily:

| Operation | Complexity |
|---|---|
| `list[i]`, `len`, `append`, `pop()` | Θ(1) (append amortized) |
| `list.insert(0, x)`, `list.pop(0)`, `del list[0]` | Θ(n) |
| `x in list` | Θ(n) |
| `x in set` / `x in dict` | Θ(1) average |
| `dict[k]`, `set.add` | Θ(1) average |
| `deque.appendleft/popleft` | Θ(1) |
| `heapq.heappush/heappop` | Θ(log n) |
| `bisect.insort` | Θ(n) (the search is Θ(log n), the insert is Θ(n)) |
| `sorted` | Θ(n log n) |
| `list.sort` on nearly-sorted data | Θ(n) (Timsort exploits runs) |
| `str + str` in a loop | Θ(n²) total |
| `"".join(list)` | Θ(n) |

The `list.pop(0)` in a loop is the single most common accidental quadratic in Python code.
`bisect.insort` is the second — people see the Θ(log n) search and forget the Θ(n) shift.

## 3. Construction: analysis in practice

**Exercise A — prove the bounds.** For each, prove or disprove from the definitions, on paper:

1. `n log n = O(n^1.01)`
2. `2^(n+1) = O(2^n)`
3. `2^(2n) = O(2^n)`
4. `log(n!) = Θ(n log n)`
5. `Σ_{i=1}^{n} 1/i = Θ(log n)`
6. If `f = O(g)` and `g = O(h)` then `f = O(h)`
7. `max(f, g) = Θ(f + g)`
8. `f = O(g)` implies `2^f = O(2^g)` — true or false?

(3 and 8 are the interesting ones; both are false, and understanding why is the point.)

**Exercise B — the potential method.** Derive an amortized bound for each, choosing Φ
yourself:

1. A binary counter under increment (n increments; show O(1) amortized bit flips).
2. A stack with a `multipop(k)` operation.
3. A dynamic array with both grow-on-full and shrink-when-quarter-full. **Why quarter and not
   half?** (Halving on half-empty gives Θ(n) per operation on an alternating
   append/pop sequence — construct that sequence and show it.)
4. Two stacks implementing a queue.

**Exercise C — find the hidden quadratic.** In each, identify the complexity and fix it:

```python
# 1
result = ""
for s in strings: result += s

# 2
while queue: item = queue.pop(0); process(item)

# 3
for x in a:
    if x in b: matches.append(x)        # b is a list

# 4
for i, x in enumerate(xs):
    if x in xs[:i]: duplicates.append(x)

# 5
seen = []
for x in stream:
    if x not in seen: seen.append(x); yield x

# 6
sorted_items = []
for x in stream: bisect.insort(sorted_items, x)
```

For each: state the current complexity with a one-line justification, give the fix, and state
the new complexity. Then *measure* both at n = 10³, 10⁴, 10⁵ and confirm the growth matches
your analysis. That confirmation step is what makes the analysis real.

**Exercise D — the model matters.** Implement two Θ(n) algorithms with very different
constants: summing a Python list of ints, and summing a NumPy array. Measure both at
increasing n. Plot time against n on log-log axes; both are lines of slope 1, separated by a
constant. Then plot the *ratio* against n and explain its shape in terms of PY-602 L06's
cache effects. This is the concrete demonstration that Θ is not the whole story.

## 4. Failure modes

- **Saying O when you mean Θ.**
- **Confusing amortized with average-case.** Different assumptions, different guarantees.
- **Trusting an amortized bound under a latency SLO.**
- **Ignoring constants at realistic n.** The asymptotically better algorithm may lose at your
  scale — and you should know your scale.
- **Forgetting the model.** Unit-cost arithmetic on big integers; unit-cost memory on data
  larger than cache.
- **Applying the comparison lower bound to non-comparison sorts.**
- **`list.pop(0)` and `list.insert(0, x)` in a loop.**
- **`in` on a list where a set was meant.**
- **String concatenation in a loop.**
- **Analysing the wrong operation.** The complexity that matters is the one in the loop.
- **Assuming `dict` is O(1) worst case.** It is O(1) *average*; adversarial keys with
  colliding hashes make it O(n) — which is the DoS that PEP 456's hash randomization defends
  against (PY-501 L05 §2.3).

## 5. Exercises

### Warm-up (30 min)

**W1.** Do Exercise A, items 1–4, on paper. Time yourself; fluency is the goal.

**W2.** Order these by growth: `n`, `n log n`, `n^1.5`, `2^n`, `n!`, `log n`, `n^log n`,
`(log n)^n`, `2^(log n)`, `n^(1/log n)`. Justify the two you found hardest.

**W3.** For Exercise C, predict each complexity before running anything, then confirm by
measurement.

### Core (2.5 h)

**C1 — Proofs.** Complete Exercise A in full, on paper, then typed up. Items 3 and 8 require
you to state *why* the false ones are false, with counterexamples.

**C2 — Amortized analysis.** Complete Exercise B, all four, with an explicit Φ and the
verification that Φ ≥ 0 and Φ(D₀) = 0. Then implement the shrinking dynamic array and
*demonstrate* the Θ(n)-per-operation behaviour of the half-empty shrink policy with the
adversarial sequence, measuring it.

**C3 — The quadratic hunt.** Complete Exercise C with measurements confirming each analysis.
Then find a genuine accidental quadratic in a real codebase (yours or open source) and report
it: the code, the analysis, the fix, and the measured improvement at realistic n.

**C4 — Model comparison.** Complete Exercise D. Then extend it: implement binary search over
a sorted Python list and over a NumPy array, and measure. Explain why the crossover happens
where it does, referring to both the asymptotics and the cache model.

### Challenge

**X1.** Prove the Ω(n log n) comparison-sorting lower bound in full, including the Stirling
step. Then research and write 800 words on how radix sort, counting sort, and the
"sorting networks for small n" used inside Timsort each evade it — being precise about which
assumption each drops.

**X2.** Implement a *deamortized* dynamic array: worst-case O(1) append, achieved by
incrementally copying elements from the old array during subsequent appends. Prove the
worst-case bound. Then measure the latency distribution of appends for the amortized and
deamortized versions and plot the p999. Report the throughput cost of the worst-case
guarantee.

## 6. Self-check

1. State the definitions of O, Ω, Θ, and o, and give the quantifier difference between O
   and o.
2. Prove `3n² + 5n + 2 = O(n²)` and `n log n ≠ O(n)`.
3. What three things do asymptotics hide?
4. Name four models of computation and one result that depends on each.
5. Prove the comparison-sorting lower bound, and state its two assumptions.
6. Distinguish amortized from average-case analysis precisely.
7. Give the potential method's definition and the heuristic for choosing Φ.
8. Give three situations where an amortized bound is not good enough, and the two remedies.

## 7. Primary sources

- Kleinberg & Tardos, ch. 2.
- CLRS, 4th ed., chs. 3 (asymptotics) and 16 (amortized analysis) — the potential method
  especially.
- Tarjan, "Amortized Computational Complexity" (SIAM J. Alg. Disc. Meth., 1985) — the paper
  that introduced the framework.
- Aggarwal & Vitter, "The Input/Output Complexity of Sorting and Related Problems" (1988) —
  the external memory model.
- Python's `TimeComplexity` wiki page, and `Objects/listobject.c` for the actual growth
  factor.

---

**Next:** [L02 — Divide and Conquer, and Recurrences](L02-divide-and-conquer.md)
