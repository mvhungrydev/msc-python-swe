# CS-621 · Lesson 06 — Randomized Algorithms and Concentration

**Estimated study time:** 4.5 hours
**Prerequisites:** L01–L05; basic probability

---

## 1. Orientation

Randomization buys three things that determinism cannot:

1. **Simplicity.** Randomized quicksort is a page; deterministic median-of-medians
   quickselect is three pages and slower in practice.
2. **Robustness against adversarial input.** A deterministic algorithm has a worst case that
   an adversary can construct. A randomized one has a worst case that *the adversary cannot
   force*, because the adversary does not know your coin flips.
3. **Speed with a bounded error probability.** For many problems, accepting a 2⁻⁵⁰ chance of
   being wrong buys an enormous speedup — and 2⁻⁵⁰ is far below the probability of a hardware
   fault, which makes it a better guarantee than "deterministically correct on hardware that
   might flip a bit".

The tools are expectation, linearity, and — the ones worth real study — the **concentration
inequalities**, which tell you how unlikely it is that a random variable strays far from its
mean. Those are what turn "expected fast" into "fast with overwhelming probability", and they
are the mathematical core of this lesson.

## 2. Theory

### 2.1 Two kinds of randomized algorithm

**Las Vegas.** Always correct; the *running time* is random. Randomized quicksort, hash
tables with random hashing, randomized quickselect.

**Monte Carlo.** Running time is bounded; the *answer* may be wrong with bounded probability.
Miller–Rabin primality, Bloom filters, min-hash, most sketches.

A Monte Carlo algorithm with one-sided error is often converted to arbitrarily-small error by
repetition: if it errs with probability ≤ ½, running it k times independently and taking the
consensus errs with probability ≤ 2⁻ᵏ. Fifty repetitions gives 2⁻⁵⁰.

A Las Vegas algorithm can be converted to Monte Carlo by cutting it off at a time bound; the
reverse conversion needs a way to *verify* an answer.

### 2.2 The probability toolkit

**Linearity of expectation.** `E[X + Y] = E[X] + E[Y]`, **regardless of dependence**. This is
the single most useful fact in the subject, because it lets you decompose a complicated random
variable into indicators and sum them without worrying about how they interact.

*Example.* How many fixed points does a random permutation of n elements have? Let `X_i = 1`
if element i is fixed. `E[X_i] = 1/n`. So `E[ΣX_i] = n · (1/n) = 1`. Exactly one, in
expectation, for every n. The `X_i` are dependent, and it does not matter.

**Indicator variables.** Turn "count the number of things with property P" into a sum of
0/1 variables, then use linearity. This is the standard move and it solves most problems in
this lesson.

**Union bound.** `P(A₁ ∪ … ∪ Aₙ) ≤ ΣP(Aᵢ)`. Crude, requires no independence, and enormously
useful: to show *nothing* bad happens, bound each bad event and sum.

**Markov's inequality.** For `X ≥ 0`: `P(X ≥ a) ≤ E[X]/a`. Very weak, needs only the mean.

**Chebyshev's inequality.** `P(|X − μ| ≥ kσ) ≤ 1/k²`. Needs the variance; polynomial decay.

**Chernoff bounds.** For a sum X of independent 0/1 variables with mean μ:

> `P(X ≥ (1+δ)μ) ≤ exp(−δ²μ / (2+δ))` and `P(X ≤ (1−δ)μ) ≤ exp(−δ²μ / 2)`

**Exponential** decay. This is the workhorse. It is why load balancing works, why sampling
works, why hash tables have short chains, and why "with high probability" claims are
believable.

**Hoeffding's inequality** generalizes Chernoff to bounded (not just 0/1) variables, which is
what you need for measurement and sampling.

The progression Markov → Chebyshev → Chernoff is a progression in **how much you assume**
(mean → variance → independence) and **how much you get** (1/a → 1/k² → e^−μ). Knowing which
you can apply is the practical skill.

### 2.3 Randomized quicksort

Choose the pivot uniformly at random.

> **Claim.** Expected comparisons = Θ(n log n).
> **Proof.** Let the sorted order be `z₁ < … < zₙ`, and `X_ij = 1` if `z_i` and `z_j` are ever
> compared. Two elements are compared exactly once or never, and they are compared **iff one
> of them is the first pivot chosen from the range `{z_i, …, z_j}`** — because if some other
> element of that range is chosen first, they are separated into different partitions and
> never meet again.
> That range has `j − i + 1` elements, each equally likely to be chosen first, and two of them
> cause a comparison. So `P(X_ij = 1) = 2/(j − i + 1)`.
> By linearity: `E[comparisons] = Σ_{i<j} 2/(j−i+1) = Σ_{i=1}^{n} Σ_{k=2}^{n-i+1} 2/k
> ≤ 2n·H_n = Θ(n log n)`. ∎

Study this proof. It is the model for the whole subject: define an indicator, find its
probability by a *combinatorial* argument, sum by linearity. The insight — "compared iff one
of them is the first pivot in their range" — is the kind of observation that makes these
proofs work, and it is found by asking *when exactly does this event happen?*

A Chernoff-style argument extends this to "Θ(n log n) with high probability", not merely in
expectation.

### 2.4 Randomized selection

Find the k-th smallest element. Partition around a random pivot, recurse into one side only.

`E[T(n)] = T(3n/4) + Θ(n)` in expectation (the pivot lands in the middle half with probability
½), giving **Θ(n) expected**.

The deterministic alternative — median of medians — is also Θ(n) worst case, and is
substantially slower in practice due to its constant. This is the standard illustration of
randomization buying simplicity and speed at the cost of a worst-case guarantee.

### 2.5 Hashing

**Universal hashing.** A family H of hash functions is *universal* if for any distinct x, y:
`P_{h∈H}[h(x) = h(y)] ≤ 1/m`. Choosing h at random from a universal family guarantees an
expected O(1) chain length **regardless of the input**, because the adversary cannot know h.

This is the theoretical justification for Python's hash randomization (PEP 456,
PY-501 L05 §2.3): without it, an attacker who knows the hash function can craft keys that all
collide, turning every dict operation into Θ(n) — an algorithmic complexity denial of service.

**The balls-in-bins result**, which you should know because it appears everywhere: throwing n
balls into n bins uniformly at random, the maximum load is `Θ(log n / log log n)` with high
probability. Not Θ(1), and not Θ(log n) — the answer is in between and it is provable with a
Chernoff bound plus a union bound.

**The power of two choices.** Choose *two* bins at random and place the ball in the less
loaded one. The maximum load drops to `Θ(log log n)` — an exponential improvement from one
extra random choice.

This is not a curiosity. It is the theoretical basis of the "two random choices" load
balancing used in real systems, and it is dramatically better than pure random assignment
while requiring far less coordination than a global least-loaded policy. If you take one
practical result from this lesson, take this one.

**Consistent hashing.** Map both keys and nodes onto a circle; a key goes to the next node
clockwise. Adding or removing a node moves only `K/n` keys instead of nearly all of them.
With **virtual nodes** (each physical node placed at multiple points), the load variance
drops. This is how distributed caches and sharded stores assign data (DS-701 L06,
DI-721 L06), and it is a randomized algorithm.

### 2.6 Random sampling

**Reservoir sampling.** Sample k items uniformly from a stream of unknown length, in one pass
and Θ(k) space:

```python
def reservoir(stream, k):
    res = []
    for i, item in enumerate(stream):
        if i < k:
            res.append(item)
        else:
            j = random.randrange(i + 1)
            if j < k:
                res[j] = item
    return res
```

> **Claim.** Every item is in the final reservoir with probability k/n.
> **Proof sketch.** By induction on n. Item n is included with probability k/n by
> construction. An earlier item, in the reservoir before step n, survives with probability
> `1 − (k/n)(1/k) = 1 − 1/n`, so its overall probability is `(k/(n−1))(1 − 1/n) = k/n`. ∎

Prove it properly; it is a good exercise in conditioning.

**Sampling for estimation.** To estimate a proportion p within ±ε with confidence 1−δ, you
need `n ≈ ln(2/δ) / (2ε²)` samples — by Hoeffding. Note what is **not** in that formula: the
population size. Sampling 1,000 items estimates a proportion equally well for a population of
10⁵ or 10⁹. This surprises people constantly and it is the basis of every "we sampled 1% of
traffic" claim.

### 2.7 Randomized data structures

**Skip lists.** A linked list with randomized express lanes; each node is promoted to the
next level with probability ½. Expected Θ(log n) search, insert, delete. Simpler than a
balanced tree, easier to make concurrent (which is why Redis uses them for sorted sets and
why lock-free skip lists are common).

**Treaps.** A BST by key, a heap by a random priority. The random priorities make it balanced
in expectation, with no rotations logic beyond the heap property.

**Bloom filters.** A bit array plus k hash functions. `add` sets k bits; `contains` checks
them. **No false negatives; false positives with probability `(1 − e^(−kn/m))^k`.** For a 1%
false-positive rate you need about 9.6 bits per element — *regardless of the element size*.
That is the striking property: a Bloom filter for a billion 100-byte strings is 1.2 GB, not
100 GB.

Uses: avoiding a disk read for a key that is absent (every LSM-tree storage engine does this
— DI-721 L02); a first-pass filter before an expensive check; distributed set reconciliation.
Variants: counting Bloom filters (support deletion), cuckoo filters (better space, support
deletion), quotient filters (better locality).

**Count-min sketch, HyperLogLog** — L07.

### 2.8 Derandomization and the practical caveats

Some randomized algorithms can be *derandomized* — the method of conditional expectations,
pairwise-independent families using O(log n) random bits, or exhaustive search over a small
sample space. Usually at a cost in simplicity and constants. The theoretical question of
whether randomness fundamentally helps (**P vs BPP**) is open, and the prevailing belief is
that it does not — that P = BPP — which would mean randomization buys convenience and speed
constants, not computational power.

**Practical caveats you must respect:**

- **`random` is not cryptographically secure.** Mersenne Twister's state is recoverable from
  624 consecutive outputs. Use `secrets` for anything an adversary might predict — session
  tokens, password resets, and, notably, *any randomization intended to defend against an
  adversary*.
- **Seeding.** Reproducibility requires a fixed seed; adversarial robustness requires an
  unpredictable one. These conflict, and you must decide per use. The usual resolution: fixed
  seeds in tests, unpredictable seeds in production, and log the seed so a production failure
  is reproducible.
- **"With high probability" needs a stated probability.** `1 − n^−c` for a specified c, not a
  vibe.
- **Independence assumptions.** Chernoff requires independence. If your "independent" hash
  functions are correlated, the bound does not hold. This is a real failure mode in sketches
  built with cheap hash functions.
- **Testing randomized code.** Fix the seed for determinism; then *separately* run with many
  random seeds to check the distributional properties. Both, not one (SE-511 L04; PY-601 L10).

## 3. Construction: implementing and verifying

**Exercise A — the proofs.** On paper:

1. Prove the randomized quicksort bound (§2.3) in full, including the harmonic sum.
2. Prove reservoir sampling's uniformity by induction.
3. Prove that n balls in n bins has maximum load `O(log n / log log n)` w.h.p., using a
   Chernoff bound and a union bound.
4. Derive the Bloom filter false-positive formula and the optimal k for given m and n.
5. Use Hoeffding to derive the sample size for ±ε at confidence 1−δ, and note the absence of
   the population size.

**Exercise B — implement and verify empirically.** For each, implement it *and* verify the
probabilistic claim by simulation:

1. **Randomized quicksort.** Verify Θ(n log n) expected comparisons by measuring the mean and
   the distribution over 1,000 runs at n = 10⁵. Compare with deterministic-pivot quicksort on
   sorted input.
2. **Reservoir sampling.** Run 10⁶ times on a stream of 100 items with k = 10, and verify
   each item appears with frequency ≈ 0.1 (chi-squared test).
3. **Balls in bins.** Simulate n = 10⁶ and measure maximum load. Compare with the predicted
   `log n / log log n`. Then implement the power of two choices and measure the improvement.
   The gap should be dramatic.
4. **Bloom filter.** Implement with a configurable m and k. Measure the actual false-positive
   rate against the formula for several (m, n, k). Plot predicted versus measured.
5. **Consistent hashing.** With and without virtual nodes. Measure: load distribution across
   nodes (report the ratio of max to mean), and the fraction of keys that move when a node is
   added or removed. Show that virtual nodes reduce the load variance and quantify by how
   much.
6. **Skip list.** Verify expected Θ(log n) search by measuring the average number of nodes
   visited at increasing n.

**Exercise C — the adversarial input.** Construct an input that makes a deterministic
algorithm quadratic and show that the randomized version is unaffected:

- Sorted input to first-element-pivot quicksort.
- Colliding keys to a hash table with a known hash function. (In Python, demonstrate this by
  fixing `PYTHONHASHSEED` and constructing colliding strings — then show that randomization
  prevents it. This is the PEP 456 attack, made concrete.)

Report the runtime ratio in each case.

**Exercise D — sampling in practice.** Take a real dataset. Estimate a statistic (a
proportion, a mean, a quantile) from a random sample of increasing size, and measure the
actual error against the true value. Plot error against sample size and compare with the
Hoeffding bound. Report where the bound is loose and why (it is a worst-case bound; the
actual error is usually much smaller).

Then estimate a **quantile** and note the difference: quantile estimation from a sample is
harder than mean estimation, and the standard practical answer is a sketch (L07).

## 4. Failure modes

- **`random` where `secrets` is needed.** Predictable tokens.
- **Assuming independence** where it does not hold, invalidating a Chernoff bound.
- **"With high probability" with no stated probability.**
- **A fixed seed in production**, making the algorithm deterministic and therefore
  adversarially attackable.
- **No seed logging**, making a production failure unreproducible.
- **Testing only with a fixed seed**, so distributional bugs are invisible.
- **Confusing expected with worst case** under a latency SLO — the same issue as amortized
  bounds (L01 §2.6).
- **Bloom filter false negatives.** There are none; if you observe one, your implementation
  is wrong (usually a mutable or non-deterministic hash).
- **Consistent hashing without virtual nodes**, producing badly uneven load.
- **Sampling a biased population** and reporting the confidence interval as if the sample were
  random. The interval measures sampling error, not selection bias, and this is the most
  common misuse of statistics in engineering.

## 5. Exercises

### Warm-up (30 min)

**W1.** Compute the expected number of fixed points in a random permutation, using indicators
and linearity. Verify by simulation.

**W2.** Demonstrate quicksort's Θ(n²) on sorted input with a deterministic pivot, and Θ(n log
n) with a random pivot. Report the ratio at n = 10⁴.

**W3.** Compute the required sample size for ±1% at 95% confidence. Verify empirically on a
population of 10⁴ and one of 10⁸, and note that the answer does not change.

### Core (3 h)

**C1 — The proofs.** Complete Exercise A, all five, on paper. Items 1 and 3 are the ones that
teach the technique.

**C2 — Implement and verify.** Complete Exercise B, all six. Deliverable: implementations,
the empirical verification of each probabilistic claim with the statistical test used, and
plots for the Bloom filter (predicted versus measured FP rate) and consistent hashing (load
distribution with and without virtual nodes).

**C3 — Adversarial inputs.** Complete Exercise C. Deliverable: both attacks constructed, the
runtime ratios, and a 400-word note on why hash randomization is a *security* feature and
what it costs (the answer includes: `hash()` is not stable across runs, which forbids
persisting it — PY-501 L05 §2.3).

**C4 — Sampling.** Complete Exercise D. Deliverable: the error-versus-sample-size plot against
the Hoeffding bound, the quantile experiment, and a 300-word note on the difference between
sampling error and selection bias with an example from your own work.

### Challenge

**X1.** Implement a lock-free concurrent skip list (in Rust via PyO3, after PY-602 L07, or in
Python with a lock for the structure and per-node locks for comparison). Benchmark against a
lock-guarded balanced tree at increasing thread counts. Report the scalability curves and
relate them to PY-601 L01's USL.

**X2.** Read Mitzenmacher's "The Power of Two Choices in Randomized Load Balancing" and
implement a load balancer using it. Compare against random assignment and against
least-loaded (which requires global state) on a realistic workload with variable request
costs. Report maximum load, p99 latency, and the coordination cost of each. Then write 800
words on why "two choices" is the right engineering answer for a distributed load balancer —
this is one of the clearest cases in systems where a theoretical result directly determines
a design.

## 6. Self-check

1. Distinguish Las Vegas from Monte Carlo, and say how to convert between them.
2. State linearity of expectation and why the "regardless of dependence" clause matters.
3. Give Markov, Chebyshev, and Chernoff, and say what each assumes and gives.
4. Reproduce the randomized quicksort proof's key insight about when two elements are
   compared.
5. State the balls-in-bins maximum load and the power-of-two-choices improvement.
6. What does universal hashing guarantee, and what attack does it prevent?
7. Give the Bloom filter's properties and its bits-per-element for a 1% false-positive rate.
8. Why does the sample size for a given error not depend on the population size?

## 7. Primary sources

- Mitzenmacher & Upfal, *Probability and Computing*, 2nd ed., chs. 1–4, 5, 13. The primary
  text for this material.
- Motwani & Raghavan, *Randomized Algorithms* — the classical reference.
- Carter & Wegman, "Universal Classes of Hash Functions" (1979).
- Mitzenmacher, "The Power of Two Choices in Randomized Load Balancing" (1996 thesis; the
  survey version is more readable).
- Bloom, "Space/Time Trade-offs in Hash Coding with Allowable Errors" (CACM, 1970).
- Vitter, "Random Sampling with a Reservoir" (TOMS, 1985).
- Karger et al., "Consistent Hashing and Random Trees" (STOC 1997).

---

**Previous:** [L05](L05-graph-algorithms.md) · **Next:**
[L07 — Hashing, Sketches, and Streaming](L07-sketches-and-streaming.md)
