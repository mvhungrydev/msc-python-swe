# PY-602 · Lesson 05 — Vectorization and NumPy's Memory Model

**Estimated study time:** 4 hours
**Prerequisites:** L04

---

## 1. Orientation

```python
result = [a[i] * b[i] + c[i] for i in range(len(a))]     # ~1.2 s for 10⁷
result = a * b + c                                       # ~0.05 s for 10⁷
```

24×. And the naive explanation — "C is faster than Python" — is only a third of the story.
The three real reasons:

1. **Amortized dispatch.** One type check and one loop setup for ten million elements
   instead of ten million of each.
2. **Contiguous memory.** The data is one block, so the prefetcher works and every cache line
   fetched is fully used (L06).
3. **No per-element objects.** No allocation, no refcounting, no pointer chasing.

But notice something in the second line: `a * b` allocates a full temporary array, then
`+ c` allocates another. For 10⁷ float64 that is 160 MB of transient allocation, and on
large arrays *that* becomes the bottleneck. Vectorization is not free, and knowing its
memory model is what separates a 24× win from a 3× win.

## 2. Theory

### 2.1 The `ndarray`

An `ndarray` is a small header plus a pointer to a flat buffer:

```python
a = np.arange(12).reshape(3, 4)
a.dtype         # int64      — element type and size
a.shape         # (3, 4)
a.strides       # (32, 8)    — bytes to step per axis
a.flags         # C_CONTIGUOUS, F_CONTIGUOUS, OWNDATA, WRITEABLE
a.base          # None if it owns its data; the parent if it is a view
a.nbytes        # 96
```

**Strides are the key idea.** Element `(i, j)` is at
`data + i*strides[0] + j*strides[1]`. Because indexing is arithmetic on strides, an enormous
number of operations are **views** — a new header pointing into the same buffer, costing
nothing:

```python
a.T                 # view: strides reversed
a[::2]              # view: doubled stride
a[1:3, 2:]          # view: offset pointer, same strides
a.reshape(4, 3)     # view IF compatible with existing strides, else a copy
a.ravel()           # view if contiguous, else a copy
np.broadcast_to(a, (5, 3, 4))   # view with a 0 stride — no memory for the repeat
```

And which are **copies**:

```python
a[[0, 2]]           # fancy indexing — ALWAYS a copy
a[a > 5]            # boolean mask — always a copy
a.flatten()         # always a copy (contrast ravel)
a.astype(np.int32)  # copy
np.ascontiguousarray(a.T)   # copy, if a.T is not contiguous
```

**Know which is which.** A view that you mutate changes the parent — a source of real bugs.
A copy in a loop is a hidden allocation — a source of real slowness. `a.base is not None`
tells you it is a view; `np.shares_memory(a, b)` answers it directly.

### 2.2 Broadcasting

Rules, applied right to left:

1. Missing dimensions are treated as 1.
2. Dimensions of size 1 are stretched (stride set to 0 — no memory).
3. Any other mismatch is an error.

```python
a = np.zeros((1000, 3))
b = np.array([1.0, 2.0, 3.0])
a + b                     # b is broadcast across rows — no copy of b
```

The trap: **broadcasting the wrong pair creates an enormous intermediate.**

```python
x = np.arange(10_000)
d = x[:, None] - x[None, :]     # 10⁸ elements = 800 MB
```

Perfectly reasonable-looking code; it allocates 800 MB. Whenever you write `[:, None]`,
compute the resulting shape in your head first. This one habit prevents most NumPy OOMs.

### 2.3 Temporaries and how to avoid them

```python
d = a * b + c        # two temporaries: (a*b), then the result
```

For arrays that fit comfortably in cache this is fine. For large arrays it costs allocation
plus a full extra pass over memory, and memory bandwidth is the binding constraint.

Mitigations, in increasing order of effort:

**`out=` parameters.** Every ufunc accepts one:

```python
np.multiply(a, b, out=d)
np.add(d, c, out=d)          # in-place, no temporaries
```

**In-place operators.** `a *= b` uses no temporary (and mutates `a` — including any view
sharing its buffer).

**`numexpr`.** Evaluates an expression in blocks that fit in cache, fusing the operations:

```python
import numexpr as ne
d = ne.evaluate("a * b + c")     # single pass, multi-threaded
```

Typically 2–4× on large arrays, purely from avoiding temporaries and improving locality.

**Numba.** JIT-compile a loop, fusing everything and avoiding all temporaries:

```python
from numba import njit
@njit(fastmath=True, cache=True)
def f(a, b, c):
    out = np.empty_like(a)
    for i in range(a.size):
        out[i] = a[i] * b[i] + c[i]
    return out
```

A hand-written loop that is *faster* than the vectorized expression, because it makes one
pass. This is the counterintuitive result worth internalizing: **on large arrays, a compiled
loop beats chained NumPy operations**, because NumPy's per-operation passes are
bandwidth-bound.

### 2.4 Memory order and why it matters

C order (row-major, the default) versus Fortran order (column-major).

```python
a = np.zeros((10_000, 10_000))     # C order
a.sum(axis=1)                      # sum along rows — contiguous, fast
a.sum(axis=0)                      # sum along columns — strided, slower
```

The difference can be 3–10×, entirely from cache behaviour: a strided access touches one
element per cache line and discards the rest (L06).

Rules:

- **Iterate along the last axis** for C-order arrays.
- Transposes are free, but *operations on a transposed array* may be strided. If you will do
  many operations, `np.ascontiguousarray` once and pay one copy.
- Check `a.flags` when performance is unexpectedly poor. A non-contiguous input to a
  library function often triggers an internal copy you did not know about.

### 2.5 dtypes

The dtype decides memory *and* speed:

| dtype | Bytes | Note |
|---|---|---|
| `float64` | 8 | the default; often unnecessary |
| `float32` | 4 | half the memory, ~2× the throughput for bandwidth-bound work |
| `int64`/`int32`/`int8` | 8/4/1 | choose by actual range |
| `bool` | 1 | not 1 bit |
| `object` | 8 (pointer) | **an array of Python objects — no vectorization at all** |

The `object` dtype is the trap. A NumPy array of Python strings or `Decimal` is a pointer
array with all of Python's per-element costs *plus* NumPy's overhead — often slower than a
plain list. If your pandas DataFrame shows `object` columns, that is where the time is going.

`float32` versus `float64` is a real decision: half the memory and roughly double the
throughput when bandwidth-bound, at the cost of ~7 decimal digits of precision instead of
~16. For accumulations over millions of elements, `float32` accumulation error is real —
`np.sum` uses pairwise summation to mitigate it, but a hand-written loop does not. Decide
deliberately and document it.

### 2.6 When vectorization does not apply

Honest limits, so you do not force it:

- **Sequential dependencies.** A recurrence `x[i] = f(x[i-1])` cannot be vectorized directly.
  Some have parallel formulations (prefix sums, scans); most do not. Use Numba.
- **Irregular control flow.** Vectorizing an algorithm with data-dependent branching often
  means computing both branches and masking — which is a slowdown unless the branches are
  cheap.
- **Small arrays.** Below a few hundred elements, NumPy's per-call overhead (~1–5 µs)
  dominates. A Python loop over a list of 20 numbers is faster than NumPy.
- **Ragged data.** Lists of varying-length sequences. Use Arrow's list types, or pad, or
  don't.
- **Non-numeric work.** Strings and objects: use Arrow, `polars`, or purpose-built libraries.

And the honest observation: **vectorized code is often less readable**. An expression with
three `[:, None]`s and a `np.where` is harder to verify than the loop it replaced. Numba lets
you keep the loop and get the speed, which is frequently the better trade.

### 2.7 The ecosystem

- **NumPy** — the foundation; the array and the ufunc protocol.
- **Numba** — JIT for numeric Python loops. `@njit`, `parallel=True` for automatic
  multi-threading, `nogil=True` to release the GIL (PY-601 L02 §2.5). The best
  effort/benefit ratio in this list for loop-shaped code.
- **Cython** — compile annotated Python to C. More control, more ceremony (L07).
- **`numexpr`** — expression fusion for large arrays.
- **Polars / DuckDB** — columnar query engines. For *tabular* work these usually beat
  hand-vectorized NumPy, because they add query optimization, lazy evaluation, and
  parallelism on top of the same layout ideas. Consider them before writing your own
  vectorized pipeline.
- **Arrow** — the columnar memory standard; zero-copy interchange between processes and
  languages.
- **JAX / PyTorch** — if you need autodiff or a GPU; the array APIs are similar enough that
  the concepts transfer.

The strategic point: **most "vectorize this" problems in industry are actually tabular query
problems**, and a columnar engine solves them better than array code. Reach for NumPy when
the computation is genuinely array-shaped (signal processing, linear algebra, simulation) and
for a query engine when it is table-shaped.

## 3. Construction: vectorizing a real computation

Take something loop-shaped and numeric — a rolling statistic, a physics step, a scoring
function, a distance computation.

**Stage 1 — the baseline loop.** Pure Python over lists. Measure with L01's harness at three
input sizes so you can see the scaling.

**Stage 2 — naive NumPy.** Convert lists to arrays, express with array operations. Measure.
Then count the temporaries: for each intermediate expression, note its shape and bytes.
Report the total transient allocation and compare with `tracemalloc`'s peak.

**Stage 3 — eliminate temporaries.** `out=` parameters and in-place operations. Measure. On
large arrays expect a further 1.5–3×; on small arrays expect nothing, and understand why.

**Stage 4 — check the memory order.** Is your access along the contiguous axis? If not, try
both orders and measure. Run `perf stat -d` for cache misses on each (L02 §2.4).

**Stage 5 — `numexpr` and Numba.** Both. Report the speedup of each over stage 3. Expect
Numba to win on anything with a loop or a branch, and `numexpr` to win on pure elementwise
expressions over very large arrays.

**Stage 6 — the dtype experiment.** `float32` versus `float64`. Measure time and memory, and
*measure the numerical difference in the result*. Decide, and write the justification —
this is a correctness decision wearing a performance costume.

**Stage 7 — the query-engine comparison.** If the computation is table-shaped, implement it
in Polars or DuckDB and compare with your best NumPy version on time, memory, and lines of
code. This is often humbling and always instructive.

**Stage 8 — the report.** A table of all versions: time at three input sizes, peak memory,
lines of code, and readability (your own honest one-to-five rating). Then a recommendation
that weighs all four, not just time.

## 4. Failure modes

- **Object-dtype arrays.** All the costs, none of the benefits.
- **Broadcasting into an enormous intermediate.** `x[:, None] - x[None, :]`.
- **Chained operations on huge arrays**, allocating a temporary per operation.
- **Fancy indexing in a loop.** Every one is a copy.
- **Mutating a view and surprising the parent.**
- **Assuming `reshape` is free.** It is a copy when strides do not permit a view.
- **Strided access along the wrong axis.**
- **Passing a non-contiguous array to a library** that silently copies it.
- **NumPy for small arrays.** Per-call overhead dominates.
- **`float64` by default** when `float32` would do — twice the memory and half the
  throughput for nothing.
- **`float32` accumulation** without considering precision.
- **Vectorizing something a database or query engine should do.**
- **Unreadable vectorized code** where Numba would have kept the loop and the speed.

## 5. Exercises

### Warm-up (25 min)

**W1.** For fifteen NumPy operations, predict view-or-copy, then check with `a.base` and
`np.shares_memory`. Report your error rate.

**W2.** Write a broadcasting expression that allocates more than 1 GB from two 10,000-element
arrays. Then compute the intermediate's shape by hand before running it.

**W3.** Time `a.sum(axis=0)` versus `a.sum(axis=1)` on a 10,000×10,000 C-order array. Explain
the ratio and confirm with `perf stat -d`.

### Core (2.5 h)

**C1 — The vectorization study.** Complete §3, all eight stages. Deliverable: the full table
(time at three sizes, peak memory, LOC, readability), the temporary-allocation accounting from
stage 2, the cache-miss comparison from stage 4, the dtype decision with its numerical
justification, and a recommendation weighing all four columns.

**C2 — Numba versus NumPy.** Find a computation where a Numba loop beats the vectorized NumPy
version, and one where it does not. Explain both in terms of passes over memory and
per-operation overhead. Report the crossover input size.

**C3 — Find the object columns.** Take a real pandas or NumPy workload. Find every
`object`-dtype array or column. For each, determine what it should be (categorical, string
dtype, fixed-width, or a different structure), convert it, and measure memory and time before
and after.

**C4 — Precision.** For an accumulation over 10⁸ elements, compare `float32` naive,
`float32` with Kahan summation, `float64`, and `np.sum` (pairwise). Report the error against
an exact (`Fraction` or `math.fsum`) result, and the time for each. Then state the rule you
would give a colleague.

### Challenge

**X1.** Implement a rolling-window computation four ways: Python loop, NumPy with
`sliding_window_view` (a stride trick — no copy), `numexpr`, and Numba with
`parallel=True`. Benchmark across window sizes and array sizes. Explain the crossovers.
Then explain what `sliding_window_view` actually does to the strides and why it uses no extra
memory but can be *slower* than a copy for some window sizes.

**X2.** Take a data-processing pipeline currently written with pandas. Rewrite it in Polars
(lazy) and in DuckDB SQL. Compare time, peak memory, and lines of code at three data sizes,
including one that does not fit in memory. Write 800 words on what the query engines do that
hand-written array code cannot — predicate pushdown, projection pruning, streaming execution,
and parallelism — and when that stops being an advantage.

## 6. Self-check

1. Give the three reasons vectorized code is faster, in order of importance.
2. Explain strides, and give five operations that are views and five that are copies.
3. State the broadcasting rules and the trap they create.
4. Why can a Numba loop beat chained NumPy operations on large arrays?
5. Why is `a.sum(axis=0)` slower than `a.sum(axis=1)` for a C-order array?
6. What is wrong with an `object`-dtype array?
7. Give five situations where vectorization does not apply.
8. When should you reach for a columnar query engine instead of NumPy?

## 7. Primary sources

- NumPy documentation: "Internal organization of NumPy arrays", "Broadcasting", and the
  `ndarray` reference. The internals page is short and repays careful reading.
- Harris et al., "Array programming with NumPy" (Nature, 2020).
- van der Walt, Colbert & Varoquaux, "The NumPy Array: A Structure for Efficient Numerical
  Computation" (2011).
- Numba documentation, "Performance tips" and the `nogil`/`parallel` chapters.
- Apache Arrow columnar format specification — for the interchange story.

---

**Previous:** [L04](L04-object-costs-and-layout.md) · **Next:**
[L06 — Caches, Locality, and Data-Oriented Design](L06-caches-and-locality.md)
