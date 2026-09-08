# PY-602 · Lesson 04 — CPython Object Costs and Data Layout

**Estimated study time:** 4 hours
**Prerequisites:** PY-501 L02, L03, L08, L09; L03

---

## 1. Orientation

```python
xs = [i for i in range(10_000_000)]
```

That is roughly 400 MB and about a second. The same data as a NumPy array is 80 MB and about
30 milliseconds.

The 5× memory and 30× time are not because NumPy is written in C. They are because of what a
Python `list` of `int` *is*: an array of eight-byte pointers to individually allocated
28-byte objects scattered across the heap, each with a reference count and a type pointer,
each requiring a pointer dereference to a cache line that is probably cold.

Understanding the layout lets you **predict** these numbers rather than discovering them, and
prediction is what lets you choose a representation before writing the code.

## 2. Theory

### 2.1 The object header

Every CPython object begins with:

```c
typedef struct _object {
    Py_ssize_t ob_refcnt;      // 8 bytes
    PyTypeObject *ob_type;     // 8 bytes
} PyObject;                     // 16 bytes minimum
```

Variable-size objects (`PyVarObject`) add `ob_size` — another 8 bytes. Objects tracked by the
cycle collector carry a `PyGC_Head` **before** the object (2 more pointers, 16 bytes) which
`sys.getsizeof` includes but which is easy to forget when reasoning.

So: **16–40 bytes of overhead before any of your data**, per object. That is the number to
carry around.

*(Free-threaded builds change the header — biased refcounting adds fields. Measure on your
build rather than assuming; the principle is unchanged.)*

### 2.2 Sizes worth memorizing

On 64-bit CPython 3.12, `sys.getsizeof`:

| Object | Bytes | Note |
|---|---|---|
| `None`, `True` | 16–28 | singletons; free |
| small `int` (< 2³⁰) | 28 | −5…256 are cached |
| large `int` | 28 + 4 per 30 bits | arbitrary precision |
| `float` | 24 | |
| `str` (ASCII, n chars) | 49 + n | compact ASCII representation |
| `str` (non-ASCII) | 74 + 2n or 4n | PEP 393: 1, 2, or 4 bytes per char |
| `bytes` (n) | 33 + n | |
| empty `tuple` | 40 | singleton |
| `tuple` (n) | 40 + 8n | pointers |
| empty `list` | 56 | |
| `list` (n) | 56 + 8n | plus over-allocation |
| empty `dict` | 64 | |
| `dict` (n small) | 64 + table | ~100 for 1–5 keys |
| `set` (n) | 216 + table | |
| plain instance | 48 + `__dict__` | |
| slotted instance (k slots) | 40 + 8k | |

**Run these yourself on your build**; they move between versions. The point is the *shape*:
containers store pointers, and the pointed-to objects are separately allocated.

**PEP 393 strings.** A string's storage depends on its widest character: pure ASCII gets
1 byte/char with a compact header; any character above U+00FF forces 2 or 4 bytes per char
for the *whole* string. A single emoji in a million-character string quadruples it. This is a
real and surprising cost in text processing.

**List over-allocation.** `list.append` grows the array geometrically (roughly 1.125× plus a
constant), so a list built by appending holds up to ~12% slack. `list(iterable)` with a known
length allocates exactly. For millions of elements, building with a comprehension over a
sized source is measurably better than repeated `append`.

**Dict layout.** Since 3.6 dicts are "compact": a dense array of entries plus a sparse index
array of small integers. Result: ordering for free, better cache behaviour, and ~20–25% less
memory than the old design. **Key-sharing dicts** (PEP 412) let instances of the same class
share their key table, so instances are much cheaper than a standalone dict — until you add
an attribute not in the shared set, which forces a private copy of the keys.

### 2.3 Predicting a data structure's cost

The method, which you should be able to do on paper:

```
one record = {"id": int, "name": str(~20 ascii), "score": float, "tags": list[str]}
```

- `dict` itself: ~232 bytes for 4 keys (a small dict with its table)
- 4 key strings: shared/interned, ~0 amortized across records
- `int`: 28
- `str` name: 49 + 20 = 69
- `float`: 24
- `list` of 3 tags: 56 + 24 = 80, plus 3 strings at ~55 each = 165
- **≈ 598 bytes per record**, so a million records ≈ 600 MB.

Alternatives, same data:

| Representation | Bytes/record | Ratio |
|---|---|---|
| `dict` | ~600 | 1.0 |
| `NamedTuple` | ~430 | 0.72 |
| plain class | ~570 | 0.95 |
| `@dataclass(slots=True)` | ~400 | 0.67 |
| columnar: 4 typed arrays + interned tags | ~60 | **0.10** |

The last row is the one to notice. Columnar wins by an order of magnitude *and* — because
iteration then touches contiguous memory — it is faster too (L06).

**Do this calculation before choosing a representation.** It takes five minutes and it is the
difference between a 600 MB service and a 60 MB one.

### 2.4 `__slots__`, precisely

```python
class P:            __slots__ = ("x", "y")
class Q:            pass
```

`P` instances: 40 bytes header + 2 pointers = 56 bytes. `Q` instances: 48 bytes plus a
`__dict__`, which for a key-sharing dict is ~104 bytes → ~152 bytes. Roughly **2.5×**.

Plus: slot access is an offset through a descriptor rather than a dict lookup — modestly
faster, and much more cache-friendly since the values are inline in the object.

The constraints (PY-501 L02 §2.5) matter in practice:

- No `__dict__`, so no arbitrary attributes, and no `functools.cached_property`.
- No `__weakref__` unless you add it to `__slots__`.
- Multiple inheritance from two classes with non-empty slots raises `TypeError`.
- A subclass without `__slots__` reintroduces `__dict__` and the saving vanishes silently.
- `dataclass(slots=True)` returns a *new class*.

**Use `__slots__` by default for any class you will have many of.** The cost is a line; the
saving is 40–60%.

### 2.5 The cost model for operations

From PY-501 L08 §2.7, refined with numbers you should verify yourself (order of magnitude,
modern hardware, CPython 3.12):

| Operation | ~ns |
|---|---|
| `LOAD_FAST` (local read) | 1–3 |
| Attribute read, specialized | 5–15 |
| Attribute read, unspecialized | 20–40 |
| Dict lookup (str key, hit) | 20–40 |
| Function call (Python→Python) | 30–60 |
| Method call | 40–80 |
| Small object allocation | 30–80 |
| `int` addition producing a new object | 20–40 |
| List append (amortized) | 20–40 |
| Exception raise + catch | 200–500 |
| `isinstance` against a class | 30–60 |
| `isinstance` against an ABC | 200–1000 |
| NumPy elementwise op, per element | 0.5–2 |

Two conclusions that should change how you write hot code:

- **Allocation and dispatch dominate.** Instruction count is secondary. Removing an
  allocation per iteration beats removing five bytecodes.
- **The gap between per-element Python and per-element NumPy is 20–100×**, and it comes from
  amortizing dispatch and from contiguity, not from C being magic.

### 2.6 Data-oriented thinking

The reframing that produces the large wins:

> **Array of structures** (a list of objects) versus **structure of arrays** (one array per
> field).

```python
# AoS — one object per record
points = [Point(x, y, z) for ...]
total = sum(p.x for p in points)         # touches 3 fields' worth of cache per point

# SoA — one array per field
xs, ys, zs = np.array(...), np.array(...), np.array(...)
total = xs.sum()                          # touches only xs, contiguously
```

SoA wins when you process one field at a time — which is what analytics, aggregation,
filtering, and most data processing do. AoS wins when you process whole records one at a
time, and when the record is the natural unit of the domain.

The costs of SoA are real: it is less natural to write, harder to keep consistent (four
arrays that must stay the same length and order), and awkward for insertion and deletion.
The mitigation is to keep an SoA *store* with a thin record *view* — which is exactly what
a DataFrame is, and why DataFrames exist.

This is the same idea as columnar storage in databases (DI-721 L05), reached from the
memory-layout direction rather than the I/O direction. Noticing that they are the same idea
is worth a lot.

### 2.7 When Python objects are the right answer

Not always the wrong choice. Keep them when:

- The collection is small (< ~10⁴). The overhead is irrelevant; readability wins.
- Records are heterogeneous or sparse.
- The domain logic lives on the objects and the code's clarity depends on it (SE-521 L06).
- You process one record at a time and never scan.

The redesign is worth it when: the collection is large, the access pattern is columnar, or
the memory is a real constraint. **Measure and predict first** (L01) — an eloquent
data-oriented rewrite of something that was never the bottleneck is a common and expensive
mistake.

## 3. Construction: predicting and measuring a redesign

Take a real in-memory data structure — a loaded dataset, a cache, an index.

**Step 1 — predict.** On paper, using §2.2 and §2.3, compute the expected bytes per record
for the current representation, and the total. Write the number down *before* measuring.

**Step 2 — measure.** `tracemalloc` peak, RSS delta, and a type census (L03 §3). Compare with
your prediction and explain the discrepancy. Common causes: interning you did not account
for, key-sharing dicts, list over-allocation, and objects shared between records.

**A prediction within 20% means you understand the model.** Iterate until you do; this is the
central skill of the lesson.

**Step 3 — the alternatives table.** Predict *and* measure the same data as: dict,
`NamedTuple`, plain class, slotted dataclass, `array.array`/NumPy columns, and a columnar
store with interned categoricals. Report both columns.

**Step 4 — measure the access patterns.** For each representation, time: full scan of one
field, full scan of all fields, random access by index, lookup by key, and appending a
record. Report the table. You will find the ranking *changes by operation*, which is the
whole point — there is no single best representation, only a best one for an access pattern.

**Step 5 — the cache story.** For the largest two representations, run `perf stat -d` on the
one-field scan and report cache miss rates. Explain the difference in terms of what is
contiguous. (L06 goes deeper; this is the appetizer.)

**Step 6 — choose and justify.** Pick a representation, and write the justification in terms
of the *actual* access pattern of your program, not in terms of which was fastest overall.

**Step 7 — the hybrid.** Implement the SoA store with an AoS view: a `Records` object holding
columns, plus a `Record` view class returning a lightweight object for one row. Measure the
view's cost. Decide whether the ergonomics are worth it.

## 4. Failure modes

- **Reasoning about memory without the header cost.** Every object is 16–40 bytes before
  your data.
- **Forgetting the GC header** on tracked objects.
- **Ignoring PEP 393**: one non-ASCII character quadruples a string.
- **Building huge lists by `append`** and paying over-allocation.
- **Not using `__slots__`** on a class with millions of instances.
- **A subclass silently reintroducing `__dict__`.**
- **AoS for a columnar access pattern.** The single largest structural win, most often
  missed.
- **SoA for a record-at-a-time pattern.** The reverse mistake; less common and equally real.
- **Redesigning before measuring.** L01.
- **Trusting the numbers in §2.2 without running them** on your build.
- **`isinstance` against an ABC in a hot loop.** Ten to thirty times more expensive than
  against a class.

## 5. Exercises

### Warm-up (25 min)

**W1.** Compute `sys.getsizeof` for the full table in §2.2 on your interpreter. Report the
differences from the table and explain any you can.

**W2.** Show that adding one emoji to a 1,000,000-character ASCII string quadruples its size.

**W3.** Measure list over-allocation: build a list of 10⁶ elements by `append` and by
`list(range(...))`, and compare `getsizeof`.

### Core (2.5 h)

**C1 — The redesign.** Complete §3, all seven steps. Deliverable: the prediction with its
error, the six-way alternatives table (predicted and measured), the access-pattern timing
table, the cache-miss comparison, the justified choice, and the hybrid implementation with
its measured cost.

**C2 — Slots everywhere.** Take a codebase and add `__slots__` to every class with more than
1,000 live instances. Measure RSS before and after on a realistic workload. Report the saving
and every place it broke (there will be some: `cached_property`, weakrefs, multiple
inheritance, dynamic attributes) and how you resolved each.

**C3 — The cost model.** Measure every row of §2.5's table on your machine, with proper
methodology (L01). Report your numbers alongside the table's and explain any large
discrepancies. This becomes a reference you will consult for years.

**C4 — Interning and categoricals.** For a dataset with high-cardinality and
low-cardinality string columns, implement: naive strings, `sys.intern`, and an integer
categorical with a lookup table. Measure memory and the time to filter by that column.
Report the break-even cardinality.

### Challenge

**X1.** Implement a columnar record store with an AoS view, supporting: typed columns,
categorical columns, null masks, append, filtered scan, and a `__getitem__` returning a row
view. Benchmark it against a list of slotted dataclasses and against pandas/polars on the
same operations. Report where each wins and why, in terms of layout rather than
implementation language.

**X2.** Read PEP 393 and PEP 412, and CPython's `Objects/dictobject.c` header comment. Write
1,200 words explaining compact dicts and key-sharing dicts: the layout, what they save, and
the specific circumstances in which key sharing is *lost* (constructing the attribute set in
different orders across instances, adding an attribute after the class's shared keys are
established). Then demonstrate the loss experimentally and measure it.

## 6. Self-check

1. What is in a CPython object header, and how large is it including the GC header?
2. Give the approximate size of: a small int, a 20-char ASCII string, a 5-element tuple, an
   empty dict, and a slotted instance with 3 slots.
3. Explain PEP 393 and the emoji effect.
4. How much does `__slots__` save, and name four things it costs.
5. Give the order-of-magnitude cost of: a local read, a dict lookup, a function call, an
   allocation, an ABC `isinstance`.
6. What dominates the cost of a hot Python loop, and what follows for optimization?
7. Explain AoS versus SoA and say which access patterns favour each.
8. Work through the per-record cost prediction for a 4-field dict record.

## 7. Primary sources

- CPython `Include/object.h`, `Objects/dictobject.c` (the header comment), `Objects/listobject.c`
  (`list_resize` for the growth factor), `Objects/unicodeobject.c`.
- PEP 393 (flexible string representation), PEP 412 (key-sharing dictionaries),
  PEP 3118 (buffer protocol).
- Bryant & O'Hallaron, *CSAPP*, ch. 6 (the memory hierarchy) — the reason layout matters.
- Acton, "Data-Oriented Design and C++" (CppCon 2014). The best available statement of the
  AoS/SoA argument, and it transfers entirely.

---

**Previous:** [L03](L03-memory-profiling.md) · **Next:**
[L05 — Vectorization and NumPy's Memory Model](L05-vectorization-and-numpy.md)
