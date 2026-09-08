# PY-602 · Lesson 07 — Extending Python: C API, Cython, and Rust

**Estimated study time:** 4 hours
**Prerequisites:** PY-501 L09; PY-601 L02; L01, L04

---

## 1. Orientation

Writing an extension is the last step of L01's optimization ladder, and it should be. It
costs: a build toolchain, wheels for every platform, a class of bugs (segfaults, refcount
errors, memory corruption) that Python does not have, a barrier to contribution, and a
maintenance burden that outlives your enthusiasm.

It is nonetheless sometimes correct. This lesson is about doing it well and — more
importantly — about knowing when the answer is "no, use Numba" or "no, restructure the data".

## 2. Theory

### 2.1 The options

| Approach | Effort | Speed | Safety | Best for |
|---|---|---|---|---|
| **Numba** | very low | high | safe | numeric loops over arrays |
| **Cython** | low–medium | high | manual | gradual typing of existing Python; C library wrapping |
| **`ctypes`/`cffi`** | low | medium | manual | calling an existing shared library |
| **PyO3 (Rust)** | medium | high | memory-safe | new native code, especially concurrent |
| **C API** | high | high | manual | maximum control; CPython internals |
| **pybind11 (C++)** | medium | high | manual | wrapping an existing C++ codebase |

The decision procedure:

1. **Is it a numeric loop over arrays?** → Numba. One decorator, no build system, no wheels.
   Try this first, always.
2. **Do I need to call an existing C library?** → `cffi` (in API mode) or Cython.
3. **Am I writing new native code?** → Rust with PyO3. Memory safety, fearless concurrency,
   a genuinely good build story (`maturin`), and no segfaults.
4. **Am I wrapping existing C++?** → pybind11 or nanobind.
5. **Do I need CPython internals?** → the C API, and know what you are taking on.

Raw C API code is rarely the right answer for new work in 2026. It remains essential
knowledge for reading CPython and for debugging other people's extensions.

### 2.2 What the C API requires you to get right

Three things, and each has a characteristic failure:

**Reference counting.** Every `PyObject*` you obtain is either a *new* reference (you own it,
you must `Py_DECREF`) or a *borrowed* one (you do not own it, and it may die). The
documentation states which for every function, and you must read it every time.

- Leak a reference → a memory leak, silent.
- Over-decref → a use-after-free, a crash somewhere unrelated, later.
- Use a borrowed reference after its owner released it → the same.

`Py_INCREF`/`Py_DECREF`, and the `Py_NewRef`/`Py_XDECREF` helpers. Since 3.12 there are
`PyObject_GetOptionalAttr`-style APIs that reduce the ambiguity, but the burden is
fundamental.

**Error handling.** A C function returning `NULL` (or `-1`) means an exception is set. You
must propagate it. Forgetting to check a return value means continuing with a null pointer
and an exception pending — a crash, or worse, a silently swallowed error.

**GIL discipline.** You hold the GIL on entry from Python. Release it around long
computations or blocking calls:

```c
Py_BEGIN_ALLOW_THREADS
    long_computation();      /* no Python API calls in here */
Py_END_ALLOW_THREADS
```

Touching any Python object without the GIL is undefined behaviour. This is where extension
bugs become impossible to debug.

For free-threaded builds, an extension must additionally declare
`Py_mod_gil = Py_MOD_GIL_NOT_USED` and actually be thread-safe, or the interpreter re-enables
the GIL at import (PY-601 L02 §2.4).

### 2.3 Cython

Python-like syntax compiled to C.

```cython
# fast.pyx
cimport cython
import numpy as np
cimport numpy as cnp

@cython.boundscheck(False)      # skip bounds checks
@cython.wraparound(False)       # skip negative-index handling
@cython.cdivision(True)         # C division semantics (no zero-division check)
def weighted_sum(cnp.float64_t[::1] values, cnp.float64_t[::1] weights) -> float:
    cdef Py_ssize_t i, n = values.shape[0]
    cdef double total = 0.0
    with nogil:
        for i in range(n):
            total += values[i] * weights[i]
    return total
```

The pieces that matter:

- **`cdef` typed locals.** Untyped Cython is roughly as slow as Python; the speed comes from
  static types. This is the single most common Cython disappointment — people compile
  unannotated code and see 1.2×.
- **Typed memoryviews** (`double[::1]`) give direct, bounds-checkable buffer access with no
  Python-object overhead. The `::1` means C-contiguous, which enables the fastest indexing.
- **`nogil`** blocks release the GIL — the whole point for parallel numeric work. Nothing
  Python-level may appear inside.
- **The directives** turn off safety checks. `boundscheck(False)` with an off-by-one is a
  segfault, not an `IndexError`. Turn them off only after the code is correct, and consider
  keeping them on in a debug build.

**`cython -a file.pyx`** produces an annotated HTML file: yellow lines are those still
interacting with the Python C API. **The workflow is: annotate, look for yellow in the hot
loop, add types until it is white.** That feedback loop is Cython's real advantage.

### 2.4 Rust with PyO3

```rust
use pyo3::prelude::*;
use rayon::prelude::*;

#[pyfunction]
fn weighted_sum(py: Python<'_>, values: Vec<f64>, weights: Vec<f64>) -> PyResult<f64> {
    if values.len() != weights.len() {
        return Err(pyo3::exceptions::PyValueError::new_err("length mismatch"));
    }
    // release the GIL for the computation
    Ok(py.allow_threads(|| {
        values.par_iter().zip(&weights).map(|(v, w)| v * w).sum()
    }))
}

#[pymodule]
fn fastlib(m: &Bound<'_, PyModule>) -> PyResult<()> {
    m.add_function(wrap_pyfunction!(weighted_sum, m)?)?;
    Ok(())
}
```

What Rust buys over C:

- **No segfaults, no use-after-free, no data races** — the borrow checker eliminates the
  three worst extension bug classes at compile time.
- **`py.allow_threads`** is the GIL release, and the type system prevents touching Python
  objects inside it. The `Py_BEGIN_ALLOW_THREADS` mistake is *unrepresentable*.
- **`rayon`** gives data parallelism in one line, with real parallelism because the GIL is
  released.
- **`maturin`** builds wheels, including manylinux, in one command.
- **Cargo** — dependency management that works.

What it costs: learning Rust (real, several weeks to productivity), longer compile times,
and a second language in your build.

The ecosystem has voted: `pydantic-core`, `polars`, `ruff`, `uv`, `orjson`, `tokenizers`, and
`cryptography` are Rust. For new native code, this is now the default choice.

**The conversion cost is the thing to watch.** `Vec<f64>` in the signature above *copies*
the data from Python. For large arrays use `numpy` crate's `PyReadonlyArray1<f64>`, which
borrows the buffer with no copy. Getting this wrong turns a 50× speedup into a 2× one, and
it is the most common PyO3 performance mistake.

### 2.5 The buffer protocol

PEP 3118. The mechanism by which NumPy, `bytes`, `memoryview`, Arrow, and your extension
share memory **without copying**.

```python
mv = memoryview(some_bytes)       # no copy
arr = np.frombuffer(mv, dtype=np.uint8)   # no copy
sock.recv_into(bytearray_buffer)  # no copy
```

An extension that accepts a buffer (`Py_buffer` in C, `PyReadonlyArray` in PyO3,
`double[::1]` in Cython) can be called with a NumPy array, a `bytes`, an `mmap`, or an Arrow
buffer, and touch the memory directly.

**Design your extension's interface in terms of buffers, not lists.** Accepting
`list[float]` forces a conversion of every element; accepting a buffer is free. This single
decision often matters more than anything inside the function.

### 2.6 Packaging and the real cost

The part people underestimate.

- **Wheels per platform.** `cibuildwheel` in CI builds Linux (manylinux/musllinux), macOS
  (x86_64 + arm64), and Windows, across Python versions. Set this up on day one; retrofitting
  it is miserable.
- **The ABI.** The **stable ABI** (`Py_LIMITED_API`) lets one wheel work across Python
  versions, at the cost of a restricted API and some performance. `abi3` wheels dramatically
  reduce your build matrix and are worth the constraint for most extensions.
- **Debug builds.** Users file segfault reports; you need symbols to act on them.
- **The contribution barrier.** A pure-Python project can be contributed to by anyone. Add
  Rust and your contributor pool shrinks. This is a real organizational cost.
- **Free-threaded support.** Another axis in the build matrix, and a correctness audit.
- **Maintenance.** Someone must keep it building in three years.

**Rule: an extension must be justified by a measurement, and the justification should include
the packaging cost.** "It's 3× faster" is not sufficient if the function is 5% of runtime.

### 2.7 The alternatives you should rule out first

Before writing any extension:

- **Numba.** For a numeric loop, it usually gets within 2× of hand-written C for a fraction
  of the effort and none of the packaging.
- **A better algorithm.** L01 §2.8 step 3.
- **NumPy/Polars/DuckDB.** Someone has already written the extension you were about to
  write.
- **A different data representation.** L04, L06.
- **PyPy.** For pure-Python hot loops with no C dependencies, its JIT is often 5–20× on
  exactly the code you were about to rewrite. Try it; it costs an afternoon.
- **Caching or precomputation.**

If a measurement shows the hot code is a tight numeric loop *and* Numba does not apply
(because it must be callable without a JIT warm-up, or must be thread-safe and
GIL-releasing, or must integrate with a C library), then write the extension.

## 3. Construction: extending, measured

Take a genuinely hot function identified by profiling (L02) — not a guess.

**Step 0 — rule out the alternatives.** Try Numba, PyPy, and a representation change. Record
the speedup of each. **If Numba gets you 80% of the way, stop and write that up as the
result.** That is a successful outcome, not a failure to do the exercise.

**Step 1 — the baseline.** The pure-Python version, measured with L01's harness. Record the
fraction of total runtime this function represents — that caps your achievable end-to-end
improvement (Amdahl).

**Step 2 — Cython, untyped.** Compile the Python source as `.pyx` with no changes. Measure.
Typically 1.1–1.5×. This is the lesson that Cython's speed comes from types, not from
compilation.

**Step 3 — Cython, typed.** Add `cdef` types and memoryviews. Run `cython -a` and drive the
hot loop white. Measure after each annotation and record the progression — the annotation
that produces the jump is instructive.

**Step 4 — Cython, unsafe.** Add `boundscheck(False)`, `wraparound(False)`,
`cdivision(True)`. Measure the additional gain. Then deliberately introduce an off-by-one and
observe the segfault, so that you have felt what you traded away.

**Step 5 — release the GIL.** `with nogil:` and measure two-thread scaling. Report the
speedup and compare with the process-based version (PY-601 L08).

**Step 6 — Rust with PyO3.** Same function. First with `Vec<f64>` parameters (copying), then
with `PyReadonlyArray1<f64>` (borrowing). **Measure both** — the copy cost is the point.
Then add `rayon` for parallelism and measure again.

**Step 7 — the end-to-end number.** All this time you have measured the function. Now measure
the *program*. Apply Amdahl: if the function was 30% of runtime and you made it 20× faster,
the program is 1.4× faster. Report that number, because it is the one that matters and it is
usually sobering.

**Step 8 — the packaging.** Set up `cibuildwheel` and build wheels for three platforms. Time
how long that took you. Include it in the cost side of the recommendation.

**Step 9 — the recommendation.** With: the end-to-end speedup, the effort, the packaging
burden, the contributor cost, and the maintenance commitment. A recommendation of "use Numba
and do not ship an extension" is a full-credit answer and is often the right one.

## 4. Failure modes

- **Writing an extension before profiling.** Optimizing 3% of runtime.
- **Ignoring Amdahl.** A 50× function speedup in a function that is 5% of runtime gives 5%.
- **Untyped Cython**, then concluding Cython is slow.
- **Turning off bounds checking before the code is correct.** Segfaults instead of
  exceptions.
- **Copying data at the boundary.** `Vec<f64>` or `list[float]` parameters on large inputs.
- **Not releasing the GIL** during a long computation — you have written a serialization
  point.
- **Touching Python objects without the GIL.** Undefined behaviour, intermittent crashes.
- **Refcount errors.** Leaks or use-after-free, both silent until they are not.
- **Not checking a C API return value.** Continuing with an exception set.
- **No `abi3`**, producing a wheel matrix of 30 builds.
- **No CI wheel building.** "It works on my machine" as a distribution strategy.
- **Underestimating maintenance.** The person who wrote it leaves; the build breaks on the
  next Python release.

## 5. Exercises

### Warm-up (30 min)

**W1.** Compile an unmodified `.py` as Cython and measure. Then add types to the hot loop and
measure again. Report both.

**W2.** Write a Cython function with `boundscheck(False)` and an off-by-one. Observe the
segfault. Then turn bounds checking on and observe the `IndexError`.

**W3.** Write a PyO3 function taking `Vec<f64>` and one taking `PyReadonlyArray1<f64>`.
Measure both on a 10⁷-element array and report the copy cost.

### Core (3 h)

**C1 — The full study.** Complete §3, all ten steps. Deliverable: the alternatives-ruled-out
table from step 0, the Cython progression with `cython -a` screenshots or notes, the
GIL-release scaling, both PyO3 variants, the **end-to-end** number with the Amdahl
calculation, the packaging time, and the recommendation with its full cost side.

**C2 — Read an extension.** Take a real Rust or Cython extension (`orjson`, `pydantic-core`,
`polars`' Python bindings, or a Cython module in scipy). Read enough to answer: how does it
receive data, where does it release the GIL, how does it handle errors, and what does its
build matrix look like? Write 800 words. Then find one thing you would do differently.

**C3 — Buffer protocol.** Write a function that accepts any buffer and computes a checksum
without copying. Verify it works with `bytes`, `bytearray`, `memoryview`, a NumPy array, an
`mmap`, and a slice of each. Measure against a version that accepts `bytes` and copies.

**C4 — Free-threaded readiness.** Take an extension (yours or a small open-source one) and
audit it for free-threaded safety: global mutable state, non-atomic refcount assumptions,
caches, and module state. Report the findings and what declaring
`Py_MOD_GIL_NOT_USED` would require.

### Challenge

**X1.** Implement the same non-trivial algorithm in Numba, Cython, and Rust/PyO3, all with
GIL release and parallelism. Benchmark across input sizes and thread counts. Report: peak
performance, performance per hour of development, lines of code, build complexity, and the
platform matrix. Then write 1,000 words recommending one *for a library you would maintain
for five years*, which is a different question from "which is fastest".

**X2.** Write a CPython C extension using the raw C API — a type with methods, correct
refcounting, and error handling. Then deliberately introduce (a) a leaked reference and (b)
an over-decref, and diagnose each: the leak with `sys.gettotalrefcount()` on a debug build,
the over-decref with a debug build's assertions and `valgrind`. Write up the diagnosis
procedure. This exercise is why Rust exists, and doing it once makes that argument concrete.

## 6. Self-check

1. Give the six extension approaches and the decision procedure for choosing.
2. Name the three things the C API requires you to get right and the failure mode of each.
3. Why is untyped Cython slow, and what does `cython -a` show you?
4. What do `boundscheck(False)` and `wraparound(False)` trade away?
5. What does PyO3's `py.allow_threads` make *unrepresentable* that C allows?
6. What is the most common PyO3 performance mistake?
7. What does the buffer protocol let your extension accept, and why design for it?
8. Give five costs of shipping an extension beyond writing it.

## 7. Primary sources

- Python C API Reference, "Reference Counting" and "Extending and Embedding" — read the
  reference-counting chapter in full once.
- PEP 3118 (buffer protocol), PEP 384/652 (stable ABI), PEP 489 (multi-phase init),
  PEP 703 (free-threading, and what extensions must declare).
- Cython documentation: "Typed Memoryviews", "Compiler Directives", "Using Parallelism".
- PyO3 user guide, and the `maturin` documentation.
- `cibuildwheel` documentation.

---

**Previous:** [L06](L06-caches-and-locality.md) · **Next:**
[L08 — I/O, Syscalls, and Zero-Copy](L08-io-and-syscalls.md)
