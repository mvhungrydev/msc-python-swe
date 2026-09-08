# PY-602 · Lesson 03 — Memory Profiling and Allocation Behaviour

**Estimated study time:** 3.5 hours
**Prerequisites:** PY-501 L09; L01, L02

---

## 1. Orientation

Memory is a performance problem twice over. Directly: a process that exceeds its limit is
killed. Indirectly, and more often: **allocation is slow, and memory that does not fit in
cache is slow to touch** (L06).

A Python program that allocates ten million small objects pays three times — once for the
allocation, once for the reference counting, and once for the cache misses when it walks
them. Reducing allocations is therefore frequently a *CPU* optimization, and it is the one
people look for last.

## 2. Theory

### 2.1 What `sys.getsizeof` does not tell you

```python
sys.getsizeof([1, 2, 3])          # 88 — the list object and its pointer array
sys.getsizeof([1, 2, 3][0])       # 28 — one int, separately
```

`getsizeof` reports **only the object itself**, not what it references. For a list of a
million ints it reports ~8 MB (the pointer array) while the true cost is ~36 MB. For nested
structures it is wildly wrong.

It also does not include: the allocator's per-block overhead, arena fragmentation, or shared
objects counted once per referrer if you naively sum.

Use it only for a single flat object, and only when you know what you are asking. For
anything real, use the tools in §2.3.

### 2.2 Where the memory goes

Four distinct consumers, and they need different tools:

1. **Python objects on the heap.** Visible to `tracemalloc` and `gc.get_objects()`.
2. **Native allocations** by extensions — NumPy buffers, database driver buffers, TLS
   contexts. *Invisible* to `tracemalloc` unless the extension uses `PyMem_*`.
3. **Interpreter and import overhead.** Modules, code objects, type objects — typically
   20–60 MB before your program does anything.
4. **Fragmentation.** Freed memory not returned to the OS (PY-501 L09 §2.6).

The practical consequence: **RSS is the only number that matters operationally, and no
Python-level tool explains all of it.** Reconcile Python-level accounting with RSS explicitly
and be suspicious of the gap.

```python
import resource, tracemalloc
rss_kb = resource.getrusage(resource.RUSAGE_SELF).ru_maxrss   # KB on Linux, bytes on macOS
current, peak = tracemalloc.get_traced_memory()
```

### 2.3 The tools

**`tracemalloc`** — Python-level allocation tracking with tracebacks. Built in.

```python
import tracemalloc
tracemalloc.start(25)                     # keep 25 frames

snap1 = tracemalloc.take_snapshot()
do_work()
snap2 = tracemalloc.take_snapshot()

for stat in snap2.compare_to(snap1, "traceback")[:10]:
    print(f"{stat.size_diff/1e6:+8.2f} MB  {stat.count_diff:+8d}")
    for line in stat.traceback.format():
        print("   ", line)
```

Sees: every `PyMem`/`PyObject` allocation, with the Python line that caused it. Does not see:
native `malloc` by extensions. Overhead: significant (roughly 2–3× slower with deep
tracebacks), so it is a development tool, though `start(1)` is cheap enough for occasional
production use.

**`memray`** (Linux/macOS) — the best available. Tracks native *and* Python allocations, with
flame graphs, a live TUI, and a temporal view.

```bash
memray run -o out.bin app.py
memray flamegraph out.bin           # allocation flame graph
memray flamegraph --leaks out.bin   # allocations never freed
memray tree out.bin
memray run --live app.py            # live TUI
```

`--leaks` is the killer feature: it shows allocations still live at exit, attributed to the
line that made them.

**`scalene`** — profiles CPU, memory, and GPU together, and distinguishes *Python* from
*native* memory, and copy volume. Its unique contribution is showing memory *growth rate* per
line, which points at the accumulation rather than the allocation.

**`objgraph`** — reference graphs. When you know *what* is leaking but not *what holds it*:

```python
import objgraph
objgraph.show_growth(limit=10)                 # types that grew since last call
objgraph.show_backrefs(obj, max_depth=5, filename="refs.png")
```

`show_backrefs` is the tool of last resort and it works when nothing else does.

**`gc.get_objects()` + `Counter`** — a type census, five lines (PY-501 L09 §3), and often
enough.

### 2.4 The diagnosis procedure

A leak is a *trend*, not a level. The procedure:

1. **Establish steady state.** Run the workload N times; the memory after run 3 is the
   baseline, not the memory after startup.
2. **Diff across a cycle.** Snapshot; run N more; snapshot; compare. Objects growing linearly
   with N are the suspects.
3. **Census by type.** *What* is growing, before *where*.
4. **Attribute by traceback.** `tracemalloc.compare_to("traceback")` or
   `memray flamegraph --leaks`.
5. **Find the retainer.** `gc.get_referrers`, `objgraph.show_backrefs`. This is the step that
   actually solves it (PY-501 L09 §2.3 lists the usual suspects).
6. **Fix, then re-run step 2.** Not "reason that it is fixed" — measure.

Always `gc.collect()` before a snapshot, or cycle garbage adds noise that looks like a leak.

### 2.5 Peak versus steady state

Different problems with different fixes:

**High steady state** — you are holding too much. Fix: bound the caches, use smaller
representations (L04), store less.

**High peak** — a transient spike. Usually one of:

- Materializing a whole file or query result (`.read()`, `list(cursor)`, `df = pd.read_csv`).
- `sorted()` or `set()` over a stream.
- A copy inside a library you did not know copied (NumPy fancy indexing, pandas operations,
  `json.dumps` building the whole string).
- Serialization: `pickle.dumps` builds the entire bytes object before writing.

Peak is what gets you OOM-killed, and it is invisible in steady-state monitoring sampled
every 15 seconds. Measure peak explicitly (`tracemalloc.get_traced_memory()[1]`, or
`ru_maxrss`).

**Fixes for peak, in order of preference:** stream instead of materialize (PY-502 L05);
chunk; process out-of-core; use a memory-mapped file; move the peak into a subprocess that
exits.

### 2.6 Allocation as a CPU cost

Every Python object allocation costs:

- A pymalloc block acquisition (fast — a free-list pop — but not free).
- Header initialization: refcount, type pointer, and for GC-tracked objects a GC header.
- GC tracking: container objects are added to generation 0, and every 700 net allocations
  triggers a gen-0 collection.
- Eventually, deallocation and possibly a cascade of decrefs.

Consequences you can act on:

- **A loop creating a temporary object per iteration** is doing all of the above a million
  times. Reusing a buffer, or using a generator instead of building a list, removes it.
- **`gc.freeze()` after startup** (PY-501 L09 §2.2) moves the permanent object graph out of
  gen-2 scanning. For a service with a large loaded model or cache, this reduces both pause
  time and — in forked workers — copy-on-write page faults.
- **Tuning `gc.set_threshold()`** upward reduces collection frequency at the cost of larger
  peaks. Measure both.
- **Object reuse / pooling** is worth it only for expensive-to-construct objects. Pooling
  small objects usually loses to pymalloc's free lists, and it is worth measuring rather than
  assuming.

### 2.7 Reducing memory: the ladder

In order of value per unit effort:

1. **Do not hold it.** Stream, bound the cache, use a generator. The largest wins.
2. **`__slots__`.** 40–60% off small instances (PY-501 L02 §2.5). Nearly free.
3. **Better containers.** `array.array` or NumPy for homogeneous numbers: a list of a million
   floats is ~8 MB of pointers plus ~24 MB of float objects; a NumPy array is 8 MB total.
   `bytes`/`bytearray` for byte data. `deque` where you need a queue.
4. **Interning and deduplication.** `sys.intern` for repeated strings; a dict of canonical
   values for repeated small objects. A dataset with a million rows and 12 distinct country
   strings should hold 12 strings, not a million.
5. **Compact representations.** Integer codes instead of strings; a categorical dtype;
   packed structs.
6. **Columnar rather than row-oriented.** One array per field instead of one object per row.
   This is L06's data-oriented design and it usually wins by 5–20× *and* speeds up
   iteration.
7. **Out-of-core.** `mmap`, memory-mapped arrays, or a database.

### 2.8 Reconciling with RSS

Python-level accounting will not add up to RSS, and the gap is informative:

| Gap source | How to see it |
|---|---|
| Interpreter + imports | RSS right after startup, before any work |
| Native allocations | `memray` (native mode), or RSS minus `tracemalloc` |
| Fragmentation | RSS stays high after freeing; `/proc/self/smaps` |
| Allocator arenas not returned | same |
| Thread stacks | 8 MB virtual each, small resident |
| Memory-mapped files | RSS counts touched pages |

Rule: **report RSS in your findings, always**, and state how much of it you can attribute.
An analysis that accounts for 60% of RSS and says so is honest; one that reports only the
Python-level number is misleading.

## 3. Construction: an allocation study

Take a memory-heavy program — a data pipeline, an import job, a service handling large
payloads.

**Step 1 — measure the shape.** RSS over time (sample `ru_maxrss` or read
`/proc/self/statm` every 100 ms) and peak versus steady state. Plot it. The shape tells you
which problem you have: a rising line is a leak; a sawtooth is peak allocation; a step is a
cache filling.

**Step 2 — reconcile.** At steady state, compare `tracemalloc` current with RSS. Report the
gap and attribute as much as you can using §2.8's table. Report the unattributed remainder
honestly.

**Step 3 — the census.** Type counts from `gc.get_objects()` at steady state. Which types
dominate? For each of the top three, estimate the per-instance cost (`getsizeof` plus what it
references, computed properly) and multiply.

**Step 4 — attribute the peak.** `tracemalloc` peak with tracebacks, or
`memray flamegraph`. Find the single line responsible for the largest transient. It is
usually a `.read()`, a `list(...)`, or a library copy.

**Step 5 — climb the ladder.** Apply §2.7 in order, measuring after each change:

| Change | RSS before | RSS after | Runtime before | Runtime after |
|---|---|---|---|---|

Report the table. Note especially whether memory reductions *also* improved runtime — they
usually do, via allocation and cache effects, and that surprises people.

**Step 6 — the columnar rewrite.** For the row-oriented part of your data, restructure to
one array per field. Predict the memory reduction before measuring (you can: count the fields,
their dtypes, and the per-object overhead you remove). Then measure. Report the prediction
error — a prediction within 20% means you understand the model.

**Step 7 — GC tuning.** Measure the pause-time distribution (instrument `gc.callbacks`) and
total collection time. Then try: `gc.freeze()` after warm-up, raised thresholds, and GC
disabled with manual collection. Report pause p99 and RSS for each, and say which you would
deploy.

## 4. Failure modes

- **`sys.getsizeof` on a nested structure.** Reports the container only.
- **Snapshotting without `gc.collect()`.** Cycle garbage looks like a leak.
- **One snapshot instead of a trend.**
- **Ignoring peak** because monitoring samples steady state.
- **`tracemalloc` only, with a large native allocator in play.** The real memory is invisible.
- **Not reporting RSS.**
- **Assuming freed memory returns to the OS.**
- **Pooling small objects** without measuring against pymalloc.
- **`lru_cache` on methods** (PY-501 L09 §4) — the most common accidental leak.
- **Storing exceptions or generators** (PY-501 L07 §2.7).
- **Optimizing memory before checking whether it is a problem.** L01 applies here too.

## 5. Exercises

### Warm-up (25 min)

**W1.** Show that `sys.getsizeof` under-reports a list of a million ints by 4×. Then write a
correct deep-size function and state its limitations (shared objects, cycles).

**W2.** Plot RSS over time for a program with a leak, one with a peak, and one with a filling
cache. Show that the three shapes are distinguishable.

**W3.** Measure the memory of one million small records as: dicts, tuples, plain classes,
slotted classes, dataclasses, `NamedTuple`, and NumPy structured arrays. Report the table.

### Core (2.5 h)

**C1 — The allocation study.** Complete §3, all seven steps. Deliverable: the RSS shape plot,
the reconciliation with the honest unattributed remainder, the census, the peak attribution,
the ladder table with both memory *and* runtime columns, the columnar prediction and its
error, and the GC tuning results.

**C2 — Find a real leak.** Plant three leaks of different shapes in a service (a module-level
cache, an `lru_cache` on a method, a retained traceback) and diagnose each using only the §2.4
procedure, timing yourself. Then apply the procedure to a real codebase and report what you
find.

**C3 — Peak elimination.** Take a program that materializes a large collection. Rewrite it to
stream (PY-502 L05). Measure peak RSS and runtime for both at three input sizes. Identify the
one stage that still decides the memory profile and state why it must.

**C4 — Interning study.** For a realistic dataset with repeated strings, measure memory with
and without deduplication, and measure the cost of the deduplication pass. Report the
break-even repetition ratio. Then check whether CPython already interned some of them and
explain which and why.

### Challenge

**X1.** Build a memory-attribution report for a real service that accounts for at least 90%
of RSS: interpreter baseline, imports (per top-level module), Python heap by type, native
allocations, thread stacks, and fragmentation. Publish the method. Very few teams have this
and it is what makes a memory budget possible.

**X2.** Instrument GC pauses in a production-shaped service (`gc.callbacks` with timestamps),
correlate pause times with request latency spikes, and quantify what fraction of p99 latency
is GC. Then apply `gc.freeze()` and threshold tuning and re-measure. Report the before/after
p99 and the memory cost.

## 6. Self-check

1. What does `sys.getsizeof` measure, and what does it omit?
2. Name the four consumers of a Python process's memory and which tools see which.
3. Give the six-step leak diagnosis procedure, and the step people skip.
4. Distinguish a peak problem from a steady-state problem, and give the fix ladder for each.
5. Explain why reducing allocations is often a CPU optimization.
6. Give the seven-rung memory-reduction ladder in order.
7. Why will Python-level accounting never equal RSS, and what do you do about it?
8. What does `gc.freeze()` do and when would you call it?

## 7. Primary sources

- `tracemalloc`, `gc`, and `resource` documentation.
- `memray` documentation, particularly the "Memory leaks" and "Native mode" chapters.
- Berger et al., "Scalene" (2020).
- CPython `Objects/obmalloc.c` header comment — arenas, pools, and blocks.
- Instagram engineering, "Copy-on-write friendly Python garbage collection" (2018) —
  the origin of `gc.freeze()`.

---

**Previous:** [L02](L02-profiling.md) · **Next:**
[L04 — CPython Object Costs and Data Layout](L04-object-costs-and-layout.md)
