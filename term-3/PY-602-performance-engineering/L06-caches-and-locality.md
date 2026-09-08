# PY-602 · Lesson 06 — Caches, Locality, and Data-Oriented Design

**Estimated study time:** 4 hours
**Prerequisites:** L04, L05

---

## 1. Orientation

Two loops over the same 10,000 × 10,000 array:

```python
for i in range(n):
    for j in range(n):
        total += a[i][j]        # row-major traversal

for j in range(n):
    for i in range(n):
        total += a[i][j]        # column-major traversal
```

Identical instruction count, identical arithmetic, identical results. The second is 3–10×
slower.

The difference is entirely the **memory hierarchy**. The first loop touches consecutive
addresses; each 64-byte cache line fetched serves eight `float64`s and the hardware
prefetcher predicts the next one. The second jumps 80,000 bytes per access; every access is a
cache miss, every fetched line serves one element, and the prefetcher cannot help.

This is why *data layout* often dominates *algorithm* for constant-factor performance, and
why the same principle appears in three places in this program — NumPy strides (L05),
columnar storage (DI-721 L05), and here.

## 2. Theory

### 2.1 The hierarchy

Approximate, modern server CPU. Learn the *ratios*, not the numbers:

| Level | Size | Latency | Relative |
|---|---|---|---|
| Register | ~1 KB | 0 cycles | 1 |
| L1 data | 32–48 KB | ~4 cycles | ~4 |
| L2 | 512 KB–2 MB | ~14 cycles | ~14 |
| L3 (shared) | 8–64 MB | ~40–70 cycles | ~50 |
| DRAM | GBs | ~200–300 cycles | **~250** |
| NVMe SSD | TBs | ~100 µs | ~300,000 |
| Network (same DC) | — | ~500 µs | ~1,500,000 |
| Disk (spinning) | — | ~10 ms | ~30,000,000 |

**A DRAM access costs roughly 250 cycles** — enough time to execute several hundred
instructions. Which is why, on modern hardware, *most programs are memory-bound*, and why
counting instructions is a poor model of performance.

Jeff Dean's "Latency Numbers Every Programmer Should Know" is the canonical version of this
table. Memorize the orders of magnitude; they let you sanity-check any performance claim in
seconds.

### 2.2 Cache lines

Memory moves in **64-byte lines**. Consequences:

- **Touching one byte fetches 64.** If you use all 64, the fetch cost is amortized 64-fold.
  If you use 8, you have wasted 87% of the bandwidth.
- **Alignment matters.** A structure straddling a line boundary costs two fetches.
- **False sharing.** Two threads writing to *different* variables in the *same* line cause
  the line to ping-pong between cores. Each write invalidates the other core's copy. This can
  make a parallel program slower than the sequential one, with no logical sharing at all.

In Python you rarely control layout directly — but you control it completely through NumPy
dtypes and structured arrays, and you meet false sharing in extensions and in the
free-threaded interpreter.

### 2.3 The three kinds of locality

**Spatial locality** — accessing nearby addresses. Sequential array traversal is perfect;
pointer chasing (a linked list, a tree of Python objects) is terrible.

**Temporal locality** — reusing recently accessed data. Loop tiling exploits it.

**Instruction locality** — a tight loop's code fits in the L1 instruction cache. Massive
inlining and megamorphic call sites hurt it.

The design consequence for Python: **a list of Python objects has almost no spatial
locality.** The pointer array is contiguous, but every dereference goes to a separately
allocated object somewhere else in the heap. Iterating a million-object list is a million
potential cache misses. A NumPy array of the same data has perfect spatial locality. This is
the largest part of L05's 24×, and L04's AoS/SoA argument restated in hardware terms.

### 2.4 Measuring cache behaviour

```bash
perf stat -e cycles,instructions,cache-references,cache-misses,LLC-load-misses \
          -e L1-dcache-loads,L1-dcache-load-misses python bench.py
```

What to look for:

- **IPC (instructions/cycles).** < 1.0 means stalls. Modern cores can retire 4+ per cycle,
  so 0.3 means you are waiting on memory ~90% of the time.
- **L1 miss rate.** Above ~5% is worth investigating.
- **LLC miss rate.** Every LLC miss is a DRAM trip. Above ~1% of loads is a memory-bound
  program.
- **Stalled cycles.** `perf stat` reports frontend and backend stalls; backend stalls
  dominated by memory is the signature.

`perf mem` and `perf c2c` go further — the latter specifically detects false sharing and is
the only practical way to find it.

**Estimate before measuring**, and check: working-set size versus cache size. If your hot
data is 200 MB and L3 is 32 MB, you *will* be DRAM-bound and no amount of instruction
tuning will help. The fix is to make the working set smaller (L04) or to restructure so that
you make one pass instead of many.

### 2.5 Techniques

**Tiling (blocking).** Restructure nested loops so each block's working set fits in cache:

```python
# naive matrix multiply: C = A @ B, streams all of B per row of A
for i in range(n):
    for k in range(n):
        for j in range(n):
            C[i, j] += A[i, k] * B[k, j]

# tiled: process T×T blocks, so the working set is 3T² elements
for ii in range(0, n, T):
    for kk in range(0, n, T):
        for jj in range(0, n, T):
            for i in range(ii, min(ii+T, n)):
                for k in range(kk, min(kk+T, n)):
                    for j in range(jj, min(jj+T, n)):
                        C[i, j] += A[i, k] * B[k, j]
```

Choose T so that `3T²·sizeof(elem)` fits in L1 or L2. This is what BLAS does, and it is why
a naive triple loop is 50× slower than `np.dot` — most of the gap is tiling, not SIMD.

Write it in Numba (a Python triple loop is too slow to show the effect) and measure the
speedup as a function of T. The curve has a clear optimum at the cache boundary, and seeing
it is the point of the exercise.

**Structure of arrays.** L04 §2.6. If you scan one field, do not fetch the others' bytes.

**Reduce the working set.** Smaller dtypes (`float32`, `int32`, categorical codes) mean more
elements per line and more of the data in cache. Halving the element size often gives more
than 2× on bandwidth-bound work, because it also improves cache residency.

**Fuse passes.** Three separate NumPy operations make three passes over memory. One Numba
loop makes one. On data larger than L3, this is often 2–3× (L05 §2.3).

**Prefetching.** Hardware prefetchers detect sequential and constant-stride patterns
automatically. They cannot predict pointer chasing or random access. So: *make your access
patterns predictable*, which usually means sorting your accesses.

**Sort for locality.** If you must do random lookups, sorting the *queries* by key turns
random access into near-sequential access. For a large batch this can be several times
faster including the sort — a genuinely counterintuitive and very useful trick.

**Padding to avoid false sharing.** In an extension or a free-threaded build, pad per-thread
counters to 64 bytes so they occupy separate lines.

### 2.6 What you can actually control in Python

The honest inventory:

| Lever | Available in pure Python? |
|---|---|
| Array-of-structs → struct-of-arrays | **yes** — the biggest lever |
| dtype width | **yes**, via NumPy/`array` |
| Access order (row vs column) | **yes** |
| Fusing passes | yes, via Numba or `numexpr` |
| Tiling | via Numba/Cython |
| Sorting for locality | **yes** |
| Working-set reduction | **yes** — L04's ladder |
| Alignment, padding, prefetch hints | no — C/Rust extension only |
| SIMD | indirectly — NumPy, Numba (`fastmath`), or an extension |

The first, third, sixth, and seventh are available with no new tools and are where the
returns are. **Most of the achievable cache win in Python comes from choosing a
representation, not from clever loop transformations.**

### 2.7 The wider principle

The pattern recurs at every level of the storage hierarchy, with the same reasoning and
different constants:

| Level | Unit | Locality principle |
|---|---|---|
| CPU cache | 64 B line | contiguous access; SoA |
| Page / TLB | 4 KB page | reduce working-set pages; huge pages |
| SSD | 4 KB block | sequential I/O; batch |
| Database | 8 KB page | clustered index; column store (DI-721 L02) |
| Network | packet / RTT | batch; avoid chatty protocols (SE-521 L09) |
| Distributed | partition | co-locate data with computation (DI-721 L06) |

The unifying statement: **the cost of moving data dominates the cost of computing on it, at
every scale.** Design so that the data you need next is near the data you have.

This is one of the few genuinely universal principles in systems, and recognizing it in a new
context — the first time you see that a chatty microservice API and a column-major array
traversal are the *same mistake* — is a real step in engineering maturity.

## 3. Construction: making a computation cache-friendly

Take a computation over a large dataset: an aggregation, a join, a simulation step, a
scoring pass.

**Step 1 — measure the working set.** How many bytes does the hot loop touch per iteration,
and in total? Compare with L1, L2, and L3 sizes (`lscpu`, or `sysctl hw` on macOS). Predict
whether you are cache-resident or DRAM-bound *before* measuring.

**Step 2 — measure the counters.** `perf stat -d`. Report IPC, L1 miss rate, LLC miss rate.
Confirm or refute your prediction from step 1. A wrong prediction here is the most useful
outcome; work out why.

**Step 3 — traversal order.** If the data is 2-D, measure both orders. Report the ratio and
the miss rates for each.

**Step 4 — AoS → SoA.** Restructure to columns. Predict the improvement from the fraction of
each cache line you were previously wasting (if you touch one 8-byte field of a 64-byte
record, you were using 12.5% of each line, so predict up to 8×). Measure. Explain the gap
between prediction and reality — there always is one, and it is usually the prefetcher or
the fact that you touch more fields than you thought.

**Step 5 — shrink the dtype.** `float64` → `float32`, or `int64` → `int32`/`int16`, where
the range permits. Measure time *and* the numerical difference. Report both.

**Step 6 — fuse the passes.** Count the passes over memory in your current implementation.
Rewrite in Numba as one pass. Measure. Compare with the arithmetic-intensity prediction
(bytes moved / operations performed).

**Step 7 — tile.** For a nested-loop computation, implement tiling in Numba and sweep the
tile size. Plot time against T. Identify the optimum and relate it to your cache sizes.

**Step 8 — sort for locality.** If there is random access (a lookup table, a hash join),
sort the accesses by key first and re-measure, *including* the sort cost. Report whether it
paid.

**Step 9 — the report.** For each change: predicted improvement, measured improvement, the
cache counters, and an explanation of any large discrepancy. The discrepancies are the
learning.

## 4. Failure modes

- **Counting instructions as a model of speed.** On memory-bound code it predicts nothing.
- **Ignoring the working-set size.** No loop transformation saves a 200 MB working set with
  a 32 MB L3.
- **Column-major traversal of a row-major array.**
- **AoS for a single-field scan.**
- **`float64` and `int64` everywhere by default.**
- **Multiple passes** where one would do.
- **Random access without sorting** when a batch is available.
- **False sharing** in threaded counters — a parallel program slower than sequential, for
  reasons invisible in the source.
- **Micro-optimizing arithmetic in a memory-bound loop.**
- **Assuming the prefetcher will save you** with an unpredictable access pattern.
- **Benchmarking with a working set that fits in cache** when production's does not. This
  invalidates the entire measurement and is extremely common.

## 5. Exercises

### Warm-up (30 min)

**W1.** Time row-major and column-major traversal of a 10,000 × 10,000 array. Report the
ratio and the L1/LLC miss rates for each.

**W2.** Find your machine's cache sizes. Write a benchmark that reads arrays of increasing
size and plot bandwidth against size. Identify the L1, L2, and L3 boundaries on the plot.

**W3.** Demonstrate false sharing: four threads incrementing adjacent elements of an array
(in Numba with `nogil`, or in a C/Rust extension), versus four threads incrementing elements
64 bytes apart. Report the ratio.

### Core (2.5 h)

**C1 — The cache study.** Complete §3, all nine steps. Deliverable: the predictions and
measurements for each step, the cache counters, the tile-size curve with its optimum related
to your cache sizes, and the explanation of the largest prediction/measurement gap.

**C2 — Tiled matrix multiply.** Implement naive and tiled matmul in Numba. Sweep the tile
size and plot. Compare both with `np.dot` (BLAS). Report the ratio and account for the
remaining gap between your best tiled version and BLAS (the answer involves SIMD, register
blocking, and packing — name them and estimate each).

**C3 — Sort for locality.** Build a workload with a million random lookups into a 1 GB table.
Measure: unsorted lookups, sorted lookups plus the sort, and sorted lookups excluding the
sort. Report all three and the break-even batch size.

**C4 — The universal principle.** For a system you know, identify the same locality mistake
at three different levels: cache (a data layout), storage (a query or file access pattern),
and network (a chatty API). Write 600 words showing they are the same mistake, and estimate
the cost of each.

### Challenge

**X1.** Read Drepper, "What Every Programmer Should Know About Memory", parts 1–3 and 5.
Reproduce three of its measurements on modern hardware (the cache-size staircase, the effect
of TLB misses with large strides, and the prefetcher's behaviour with different access
patterns). Report what has and has not changed since 2007.

**X2.** Take a real analytical workload and implement it three ways: row-oriented Python
objects, NumPy SoA, and Arrow/Polars. Measure time, memory, and cache counters for each.
Then write 1,000 words connecting the results to DI-721's column-store argument — that the
reason columnar databases win on analytics is *the same reason* SoA wins in memory, one level
of the hierarchy down.

## 6. Self-check

1. Give the latency hierarchy from register to network, in relative terms.
2. What is a cache line, and what are the three consequences of its size?
3. Distinguish spatial, temporal, and instruction locality.
4. Why does a list of Python objects have almost no spatial locality?
5. What does IPC below 1.0 tell you, and what would you check next?
6. Explain tiling and how to choose the tile size.
7. Why does sorting random accesses help, and when does it not pay?
8. State the universal locality principle and give three levels at which it applies.

## 7. Primary sources

- Drepper, "What Every Programmer Should Know About Memory" (2007). Long, free, still the
  reference.
- Bryant & O'Hallaron, *CSAPP*, ch. 6.
- Jeff Dean, "Latency Numbers Every Programmer Should Know" — memorize the orders of
  magnitude.
- Acton, "Data-Oriented Design and C++" (CppCon 2014).
- Fog, "The Microarchitecture of Intel, AMD and VIA CPUs" — for depth.
- `perf` documentation, and Gregg on `perf c2c` for false sharing.

---

**Previous:** [L05](L05-vectorization-and-numpy.md) · **Next:**
[L07 — Extending Python: C API, Cython, and Rust](L07-extending-python.md)
