# CS-621 · Lesson 07 — Hashing, Sketches, and Streaming

**Estimated study time:** 4 hours
**Prerequisites:** L06

---

## 1. Orientation

"How many distinct users visited today?" is trivially answered with a set — until the answer
is 400 million and the set is 40 GB, one per shard, and you need it per hour and per
dimension.

**HyperLogLog answers it to within 2% using 12 KB.** That is not an optimization; it is a
different category of solution, and it exists because a mathematician asked what could be
learned from the *positions of leading zeros in hashed values*.

Sketches are the answer to a specific and increasingly common question: **what can I compute
in one pass, in space sublinear in the data, accepting a bounded error?** They are the
mathematical core of every metrics system, every stream processor, and every large-scale
analytics engine — and they are almost never taught to engineers who use them daily.

## 2. Theory

### 2.1 The streaming model

Data arrives as a sequence `x₁, x₂, …, xₙ`. You may make **one pass** (or few), and you have
**space sublinear in n** — typically polylogarithmic. You must answer a query at the end (or
continuously).

The constraints are what make it interesting: no random access, no second pass, no room to
store the input. Almost every exact answer is impossible; the question is what approximate
answer you can guarantee.

**The fundamental lower bound**: exact distinct-counting requires Ω(n) space. There is no
clever algorithm; it is information-theoretic. So approximation is not laziness, it is the
only option — and knowing that stops you from looking for the exact algorithm.

The parameters of any sketch: **ε** (relative or additive error) and **δ** (failure
probability). Space is typically `O(f(ε) · log(1/δ))`, so improving accuracy is expensive and
improving confidence is cheap. That asymmetry is worth remembering when you configure one.

### 2.2 Distinct counting: HyperLogLog

The problem: `|{distinct elements}|`, in small space.

**The idea, built up.** Hash each element to a uniform bit string. In a stream of d distinct
values, about d/2 hashes begin with `1`, d/4 with `01`, d/8 with `001`. So if the maximum
number of leading zeros seen is ρ, then d ≈ 2^ρ.

That single estimator has enormous variance — one unlucky hash ruins it. Two refinements fix
it:

- **Stochastic averaging.** Use the first b bits of the hash to select one of `m = 2^b`
  registers, and track the maximum leading-zero count in each. This is m independent
  experiments for the price of one pass.
- **Harmonic mean.** Combine the registers with a harmonic mean, which suppresses the
  influence of large outliers.

The estimate is `α_m · m² / Σ 2^(−M_j)` with a constant `α_m` correcting the bias.

**Standard error: `1.04/√m`.** With m = 16,384 registers of 6 bits each (12 KB), that is
**0.81%**. Independent of the cardinality — the same 12 KB estimates 10³ or 10¹⁰ distinct
values to the same relative accuracy. That property is what makes it so useful.

**Mergeable.** The union of two HLLs is the register-wise maximum. So you can compute per
shard, per hour, per dimension, and combine at query time — which is exactly what a metrics
or analytics system needs, and it is the property that makes HLL the industry standard rather
than a curiosity.

Practical: `datasketches`, Redis's `PFADD`/`PFCOUNT`, Presto/Trino, BigQuery, ClickHouse,
Druid all implement it. Know that HLL++ adds bias correction for small cardinalities and a
sparse representation that is exact for small sets.

### 2.3 Frequency estimation: count-min sketch

The problem: estimate `f(x)`, the count of item x, in small space.

A `d × w` array of counters and d pairwise-independent hash functions. To add x, increment
`C[i][h_i(x)]` for each i. To query, return `min_i C[i][h_i(x)]`.

Each row over-counts because of collisions; taking the minimum takes the least-polluted
estimate.

**Guarantee.** With `w = ⌈e/ε⌉` and `d = ⌈ln(1/δ)⌉`, the estimate satisfies
`f(x) ≤ f̂(x) ≤ f(x) + ε·N` with probability ≥ 1 − δ, where N is the total count.

Note the error is **additive in N**, not relative to `f(x)`. So it is accurate for *heavy
hitters* and useless for rare items — a 1% error on a stream of 10⁹ is ±10⁷, which says
nothing about an item that appeared 5 times. Understanding that shape is the difference
between using it correctly and shipping garbage.

Mergeable by element-wise addition. Uses: heavy hitters, per-key rate limiting at scale,
approximate joins, feature hashing.

The **count-sketch** variant gives an unbiased estimate with error proportional to the L2
norm rather than L1, which is better for skewed distributions.

### 2.4 Heavy hitters

"Which items appear more than n/k times?"

**Misra–Gries / Space-Saving.** Keep `k−1` counters. On each item: if it has a counter,
increment; else if a counter is free, claim it; else decrement all counters (Misra–Gries) or
evict the minimum and take its count (Space-Saving).

Guarantee: every item with frequency > n/k is reported, and each count is within n/k of the
truth. Θ(k) space, and it is deterministic.

Space-Saving is what most "top-K" implementations use. The failure mode to know: **the result
may include items that are not actually heavy** (false positives), so a second pass is needed
for exactness — and in a streaming context you do not get one, which is why these results are
labelled "approximate top K".

### 2.5 Quantiles

Harder than means, because a quantile depends on the *order statistics*, not on a sum.

- **t-digest** — clusters weighted by distance from the median, so the tails are represented
  far more precisely than the middle. Accuracy at q = 0.999 is very high, at q = 0.5 it is
  moderate. This is exactly the right trade for latency monitoring, which is why it is used
  everywhere for that.
- **HDR histogram** — fixed relative-precision buckets over a stated range. Deterministic
  error bound, very fast, mergeable, and the standard for latency because you can state the
  precision in advance.
- **KLL sketch** — the theoretically optimal one: `O((1/ε) log log(1/δ))` space for
  ε-approximate quantiles, with a proved guarantee.

**All three are mergeable, which is the property that matters** and which the naive
alternative lacks. Recall PY-602 L01 §2.3: you cannot average percentiles across instances.
You *can* merge sketches, and that is the correct way to compute a fleet-wide p99. A
monitoring system that reports the mean of per-instance p99s is reporting a number with no
meaning, and this is why.

### 2.6 Similarity: MinHash and SimHash

**MinHash.** For sets A and B, the Jaccard similarity `|A∩B| / |A∪B|` is estimated by: hash
all elements, keep the minimum hash of each set. `P(min(h(A)) = min(h(B))) = J(A,B)` exactly.
Use k independent hashes (or the k smallest of one hash) and the fraction of matches
estimates J with standard error `1/√k`.

Uses: near-duplicate detection at web scale (its original use at AltaVista), document
clustering, recommendation.

**Locality-sensitive hashing (LSH).** Hash so that *similar* items collide with high
probability. Banding MinHash signatures gives a tunable S-curve threshold: items above the
similarity threshold are found with high probability, items below are rarely retrieved.
This turns "find all similar pairs" from Θ(n²) into approximately linear.

**SimHash** does the same for cosine similarity on vectors — the basis of Google's
near-duplicate detection for web pages, and of many modern vector-search filters.

### 2.7 Streaming algorithm design

Recurring patterns:

- **Sliding windows.** Exponential histograms give ε-approximate counts over a sliding window
  in `O((1/ε) log² n)` space. Exact sliding windows require storing the window.
- **Sampling.** Reservoir sampling (L06 §2.6) with variants for weighted and
  distinct sampling.
- **Sketch + exact hybrid.** Keep exact counts for the top-K (which fit) and a sketch for the
  long tail. Best of both, and what most production systems actually do.
- **Two-pass when you can afford it.** Often the honest answer: if the data fits on disk, a
  second pass is cheap and exact. Sketches are for when it does not, or when the answer is
  needed continuously.

**The design questions**, in order:

1. What error is acceptable? Get a number from the person who will use the answer. "2% on
   distinct users" is usually fine; "2% on the billing total" is not.
2. Must it be mergeable? Almost always yes in a distributed setting.
3. Must it support deletion? Most sketches do not. (Counting Bloom filters and count sketches
   do; HLL does not.)
4. Is the distribution skewed? Count-min is fine for heavy hitters, useless for the tail.
5. Can you afford a second pass? If yes, consider whether you need a sketch at all.

### 2.8 Where sketches are wrong to use

The honest limits:

- **When the exact answer is required.** Billing, compliance, financial reconciliation. A 2%
  error on revenue is not an engineering trade-off, it is a misstatement.
- **When the data fits.** A HashSet of 10⁶ items is 100 MB and exact. Reach for a sketch at
  10⁸, not 10⁶.
- **When the error is not understood by the consumer.** A dashboard labelled "unique users"
  that is actually ±2% will be used for a decision that assumes exactness, unless the label
  says so. **Label your approximations in the UI**, not just in the code.
- **Small cardinalities.** Plain HLL is badly biased below a few thousand; HLL++ fixes this
  with a sparse exact mode. Know which your implementation uses.
- **Set intersection.** HLL supports union well; intersection via inclusion–exclusion has
  error proportional to the *union* size, which for two large nearly-disjoint sets is
  catastrophic. This is a real and common mistake: "distinct users who did A **and** B"
  computed from HLLs is often meaningless.

That last one deserves emphasis because it is the most common sketch misuse in analytics
systems, and the error is invisible — the number comes back and looks plausible.

## 3. Construction: building and validating sketches

**Exercise A — implement from scratch**, then validate empirically against the theoretical
guarantee:

1. **HyperLogLog.** Registers, stochastic averaging, harmonic mean, the α correction. Test at
   cardinalities from 10² to 10⁸ (generate, do not store). Plot relative error against
   cardinality; verify it sits near `1.04/√m`. Then implement the small-range correction and
   show it fixes the bias below ~2.5m.
2. **Count-min sketch.** Verify the `f ≤ f̂ ≤ f + εN` guarantee on a Zipf-distributed stream.
   Plot estimation error against true frequency and show the additive-in-N shape — the error
   is constant in absolute terms, so it is negligible for heavy items and total for rare ones.
3. **Space-Saving.** Top-K over a Zipf stream. Verify all true heavy hitters are found, and
   report the false positives.
4. **MinHash + LSH.** For a corpus of documents (shingled), estimate pairwise Jaccard and
   compare with exact. Then band the signatures and report precision and recall against
   ground truth at a chosen threshold. Plot the S-curve.
5. **t-digest or HDR histogram.** Estimate quantiles of a heavy-tailed distribution. Report
   error at q = 0.5, 0.9, 0.99, 0.999 and show the accuracy profile differs by quantile.

**Exercise B — the merge property.** For HLL, count-min, and your quantile sketch:

- Split a stream into 10 shards.
- Compute a sketch per shard.
- Merge.
- Compare with the sketch of the whole stream.

Verify the merged result matches. Then demonstrate the *wrong* way for quantiles: average the
per-shard p99s and compare with the true p99. Report the error. This is the concrete
demonstration of PY-602 L01 §2.3's claim, and it is worth doing because you will meet a
dashboard doing exactly this.

**Exercise C — the intersection trap.** Build two HLLs over sets with a known small
intersection. Estimate the intersection by inclusion–exclusion
(`|A| + |B| − |A ∪ B|`). Report the error as the sets become larger and more disjoint. Show
that it becomes worse than useless. Then implement the correct approach (MinHash, or exact
intersection of the underlying sets if feasible) and compare.

**Exercise D — a real metrics pipeline.** Build one:

- Ingest events with a user id, an endpoint, and a latency.
- Maintain, per minute per endpoint: distinct users (HLL), request count, top-K user agents
  (Space-Saving), and a latency histogram (HDR).
- Support querying over an arbitrary time range by merging the per-minute sketches.
- Compare with exact computation on a day of synthetic data.

Report: memory used by sketches versus exact, query latency for a 30-day range, and the error
in each statistic. This is the architecture of every metrics system, built once so you know
what is inside.

**Exercise E — the error budget.** For a real system you know, list every approximate number
on a dashboard. For each: what sketch produces it, what is the error, and does the consumer
know? Report the ones where the label implies exactness. Propose the labelling.

## 4. Failure modes

- **HLL intersection via inclusion–exclusion.** §2.8. The most common serious misuse.
- **Count-min for rare items.** The error is additive in N.
- **Averaging percentiles instead of merging sketches.**
- **Plain HLL at small cardinality** without the sparse/small-range correction.
- **Assuming sketches support deletion.** Most do not.
- **Correlated hash functions**, invalidating the independence assumption and the guarantee.
- **Using a sketch when the data fits.** Complexity for nothing.
- **Using a sketch where exactness is required.** Billing, compliance.
- **Unlabelled approximations** in a UI.
- **Not validating the guarantee empirically.** Every implementation of these has subtle
  bugs; the theoretical bound does not validate your code.
- **Choosing ε without asking the consumer** what error is acceptable.

## 5. Exercises

### Warm-up (30 min)

**W1.** Estimate distinct count with the naive leading-zeros estimator (no averaging) and
measure its variance over 1,000 trials. Then add stochastic averaging and measure again.

**W2.** Compute the memory for exact distinct counting of 10⁸ 16-byte ids as a Python set,
as a `bytes`-keyed set, and as an HLL. Report the ratios.

**W3.** Demonstrate the averaging-percentiles error with 10 shards of a skewed latency
distribution.

### Core (3 h)

**C1 — Implement and validate.** Complete Exercise A, all five. Deliverable: implementations,
the empirical error plots against the theoretical guarantee for each, and — where your
measured error exceeds the bound — the bug you found (there will usually be one; that is the
point of validating).

**C2 — Mergeability.** Complete Exercise B. Deliverable: the merge verification for three
sketch types and the quantified error of the wrong quantile-averaging method.

**C3 — The intersection trap.** Complete Exercise C. Deliverable: the error curve, and a
400-word explanation you could give to an analyst about why "distinct users who did A and B"
cannot be computed from two HLLs.

**C4 — The metrics pipeline.** Complete Exercise D. Deliverable: the pipeline, the
memory/latency/error comparison against exact, and a statement of the error budget for each
statistic.

### Challenge

**X1.** Read Flajolet et al. (2007) on HyperLogLog and Heule et al. (2013) on HLL++.
Implement HLL++ including the sparse representation and the bias correction table. Validate
against your plain HLL across the full cardinality range and reproduce the paper's accuracy
figures. Report where your implementation deviates and why.

**X2.** Design and implement a sketch for a problem in your own domain that is not in the
standard catalogue — an approximate distinct-count-per-key, a decaying frequency estimate, a
sliding-window quantile. State the guarantee you are claiming, prove it or clearly state that
it is heuristic, and validate empirically. Then write 800 words on what you would need to
prove to make the guarantee rigorous. Being honest that a sketch is a heuristic without a
proof is a legitimate and valuable outcome.

## 6. Self-check

1. State the streaming model's constraints and the fundamental lower bound for exact distinct
   counting.
2. Explain HyperLogLog's estimator, its two refinements, and its standard error formula.
3. Why is HLL's accuracy independent of cardinality, and why does that matter?
4. State count-min's guarantee and explain why it is useless for rare items.
5. Why can percentiles not be averaged, and what is the correct alternative?
6. What does MinHash estimate, and what does LSH add?
7. Give five design questions to ask before choosing a sketch.
8. Give five situations where a sketch is the wrong choice, and the most common serious
   misuse.

## 7. Primary sources

- Flajolet, Fusy, Gandouet & Meunier, "HyperLogLog: the analysis of a near-optimal
  cardinality estimation algorithm" (2007).
- Heule, Nunkesser & Hall, "HyperLogLog in Practice" (EDBT 2013) — Google's engineering
  refinements; an unusually good paper on the gap between an algorithm and a production
  implementation.
- Cormode & Muthukrishnan, "An Improved Data Stream Summary: The Count-Min Sketch" (2005).
- Metwally, Agrawal & El Abbadi, "Efficient Computation of Frequent and Top-k Elements in
  Data Streams" (2005) — Space-Saving.
- Broder, "On the Resemblance and Containment of Documents" (1997) — MinHash.
- Dunning & Ertl, "Computing Extremely Accurate Quantiles Using t-Digests" (2019).
- Karnin, Lang & Liberty, "Optimal Quantile Approximation in Streams" (FOCS 2016) — KLL.

---

**Previous:** [L06](L06-randomized-algorithms.md) · **Next:**
[L08 — NP-Completeness and Reductions](L08-np-completeness.md)
