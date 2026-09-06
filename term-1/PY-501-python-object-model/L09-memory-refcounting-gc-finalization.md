# PY-501 · Lesson 09 — Memory: Reference Counting, Cycles, and Finalization

**Estimated study time:** 4 hours
**Prerequisites:** L01, L07

---

## 1. Orientation

A Python service that grows from 200 MB to 4 GB over a week is not usually leaking in the C
sense. Nothing is unreachable-and-unfreed. Something is *reachable* that you did not intend
to keep — a cache without eviction, a logger holding exception objects, a closure over a
frame, a module-level list. Python's memory problems are almost always **retention**
problems, and diagnosing them requires a precise model of what keeps objects alive.

This lesson provides that model, and the tools to interrogate it.

## 2. Theory

### 2.1 Reference counting

Every CPython object header contains a reference count. It is incremented when a new
reference is created (binding a name, appending to a container, passing as an argument,
capturing in a cell) and decremented when one goes away. At zero, the object is deallocated
**immediately** — its `__del__` runs, its references are dropped (possibly cascading), and
its memory returns to the allocator.

```python
import sys
xs = [1, 2, 3]
sys.getrefcount(xs)      # 2: `xs`, plus the temporary argument reference
```

`getrefcount` always reports one more than you expect, because passing the object to the
function created a reference. This is a good first illustration of how easy it is to create
references you did not think about.

Properties of refcounting worth stating explicitly:

- **Deterministic.** Objects die at a predictable point. This is why
  `with open(...) as f` and even the sloppier `open(f).read()` usually close promptly on
  CPython — and why that code breaks on PyPy, which does not refcount. Never rely on it.
- **Incremental.** No stop-the-world pause for the common case.
- **Costly.** Every reference operation touches the object header. This is a write to
  shared memory, which is exactly what makes CPython's reference counting the central
  obstacle to removing the GIL (PY-601 L03).
- **Cannot collect cycles.** §2.2.

### 2.2 Cycles and the generational collector

```python
a = {}; b = {}
a["b"] = b; b["a"] = a
del a, b            # both refcounts are still 1 — unreachable but not freed
```

The cycle collector (`gc` module) exists for exactly this. Its algorithm:

1. Track **container objects only** — things that can participate in cycles (lists, dicts,
   sets, instances, tuples containing containers). Atomic objects (`int`, `str`, `float`)
   are never tracked, because they cannot reference anything.
2. Periodically, for a chosen generation: take the tracked objects, and compute a
   *subtractive* count — for each object, subtract the references that come from *within
   the set being examined*.
3. Objects whose adjusted count is zero are reachable only from within the set: they are
   garbage. Objects with a positive adjusted count are reachable from outside; they and
   everything they reach are live.
4. Collect the garbage, running finalizers.

**Generations.** Three of them (0, 1, 2). New objects enter generation 0. A collection of
generation *n* examines generations 0..*n*; survivors are promoted. Generation 0 is
collected often (by default after 700 more allocations than deallocations), generation 1
after 10 collections of generation 0, generation 2 after 10 of generation 1. The hypothesis
is the standard generational one: most objects die young.

```python
import gc
gc.get_threshold()      # (700, 10, 10)
gc.get_count()          # current counts per generation
gc.collect()            # force a full collection; returns number of unreachable objects found
gc.get_stats()          # per-generation collection statistics
```

**The cost model that matters in production:** a generation-2 collection walks *every
tracked object in the process*. For a service with a large long-lived object graph — a
big in-memory cache, a loaded model, a large config tree — that walk is expensive and
happens at unpredictable times, producing latency spikes correlated with nothing in your
code. Instagram's well-known result was that *disabling* the cycle collector entirely
improved both latency and memory (by avoiding copy-on-write page faults from refcount
writes during collection in forked workers). That is a legitimate technique with a real
cost: you must then have no cycles you care about, or call `gc.collect()` at controlled
points.

```python
gc.freeze()      # move all current objects out of generational tracking (call after startup, before fork)
gc.disable()     # stop automatic collection
```

`gc.freeze()` is specifically designed for the pre-fork case: it moves everything allocated
during startup into a permanent generation so that collections never touch it and its
refcount fields are never written, preserving copy-on-write sharing across forked workers.
CA-731 L06 returns to this.

### 2.3 What actually keeps objects alive

The practical list, in rough order of how often each causes a production problem:

1. **Module-level containers.** A dict used as a cache with no eviction. Lives forever.
2. **Class attributes.** Same, one level down (L01 §4.2).
3. **Closures and default arguments.** Both capture and hold.
4. **Exception tracebacks.** `e.__traceback__` → frames → locals → everything (L07 §2.7).
   Storing exceptions is a retention bomb.
5. **Suspended generators and coroutines.** Hold their frames and all locals. A
   partially-consumed generator in a cache retains its entire working set.
6. **Registrations.** `atexit`, signal handlers, observer lists, `weakref` callbacks that
   are themselves bound methods (a bound method holds `__self__`!), thread-locals,
   `functools.lru_cache` (which holds both arguments and results — including `self` when
   applied to a method, §4).
7. **C-level references** invisible to Python-level introspection.

`gc.get_referrers(obj)` answers "what points at this?" and is the tool of last resort. It
is slow, returns frames and internal structures, and requires the collector to be tracking
the referrer — but when you are stuck it is the only thing that works.

### 2.4 Weak references

A weak reference does not contribute to the refcount:

```python
import weakref

class Node: pass
n = Node()
r = weakref.ref(n)
r()          # the Node
del n
r()          # None
```

Uses:

- **Caches that should not keep entries alive** — `weakref.WeakValueDictionary`.
- **Attaching data to objects you do not own** — `weakref.WeakKeyDictionary` (with the
  `__eq__` hazard of L03 §2.3).
- **Breaking cycles deliberately** — parent pointers in a tree are the classic case.
- **Observer registries** that should not prevent observers from being collected.

Not everything is weak-referenceable: `int`, `str`, `tuple`, and instances of classes with
`__slots__` that do not include `__weakref__` are not. Adding `"__weakref__"` to
`__slots__` restores it at the cost of a pointer per instance.

**The bound-method trap.** `weakref.ref(obj.method)` is almost always useless: `obj.method`
creates a *new* bound method object (L03 §2.4) which dies immediately, so the weak ref is
dead on arrival. Use `weakref.WeakMethod` instead.

### 2.5 Finalization

`__del__` is called when the refcount reaches zero, or when the collector reclaims the
object. It is a bad tool and you should know exactly why:

- **Not guaranteed to run.** Not at interpreter shutdown, not if the object is alive at
  exit, not on `os._exit`, not if the process is killed.
- **Timing is not deterministic** across implementations.
- **Exceptions in `__del__` are swallowed** (printed to stderr, then ignored).
- **It can resurrect the object** by storing `self` somewhere, and since PEP 442 the
  finalizer will not run again.
- **It disrupts collection.** Historically, objects with `__del__` in a cycle were
  uncollectable (`gc.garbage`); PEP 442 (3.4+) fixed that, but finalization order within a
  cycle is still arbitrary, so a `__del__` may run after objects it depends on are already
  finalized.
- **It runs at arbitrary points**, including inside other threads, mid-allocation.

`weakref.finalize` is the better mechanism for cleanup tied to object lifetime:

```python
import weakref

class Connection:
    def __init__(self, sock):
        self._sock = sock
        self._finalizer = weakref.finalize(self, sock.close)   # note: does NOT capture self

    def close(self):
        self._finalizer()       # idempotent; also detaches
```

Critically: the callback must **not reference the object**, or it creates a strong
reference and the object never dies. Pass the resource, not `self`. `weakref.finalize` also
runs at interpreter exit by default (unlike `__del__`) and is idempotent.

**The correct answer for resources is a context manager** (L06 §2.4). Finalizers are a
safety net for the case where the caller forgot, not the primary mechanism. The standard
library follows exactly this pattern: files and sockets are context managers *and* have a
finalizer that emits a `ResourceWarning`.

### 2.6 The allocator

CPython does not call `malloc` for every object. There are layers:

- **Object-specific free lists** — small caches of recently-freed objects of a given type
  (frames, tuples of small size, small ints, floats), reused without touching the allocator
  at all.
- **pymalloc** — an arena allocator for blocks ≤512 bytes. Memory is carved into
  256 KiB *arenas*, subdivided into 4 KiB *pools*, subdivided into fixed-size *blocks*.
- **The system allocator** for anything larger.

The consequence that surprises people: **freeing objects does not necessarily return memory
to the OS.** An arena is released only when every pool in it is empty. Fragmentation —
a few long-lived objects scattered across many arenas — pins memory that is 95% free. This
is why a process that peaked at 4 GB may sit at 3 GB forever after the peak, with `gc`
showing nothing wrong.

Mitigations, in order of preference: avoid the peak (stream instead of materializing);
isolate the peak in a subprocess that exits; use `mmap`/`array`/NumPy for large
homogeneous data (which bypasses pymalloc); accept it.

Measure with `tracemalloc` (Python-level allocation attribution, with tracebacks) and
`memray` (native + Python, with flame graphs). `sys.getsizeof` reports only the object's
own size, not what it references — it is almost always the wrong tool, and knowing that is
part of C2 in L02.

## 3. Construction: a leak-finding workflow

Not a single tool — a *procedure*. Build it as a reusable module.

**Step 1 — establish the baseline.** A leak is a trend, not a level.

```python
import gc, tracemalloc, linecache

def snapshot(label: str) -> tracemalloc.Snapshot:
    gc.collect()                        # remove cycle noise
    return tracemalloc.take_snapshot().filter_traces((
        tracemalloc.Filter(False, tracemalloc.__file__),
        tracemalloc.Filter(False, linecache.__file__),
    ))
```

**Step 2 — diff across a steady-state cycle.** Run the workload N times; snapshot; run N
more; snapshot; diff. Objects that grow linearly with N are the suspects.

```python
def top_growth(a: tracemalloc.Snapshot, b: tracemalloc.Snapshot, n: int = 15):
    for stat in b.compare_to(a, "traceback")[:n]:
        print(f"{stat.size_diff/1e6:8.2f} MB  {stat.count_diff:+8d}")
        for line in stat.traceback.format():
            print("   ", line)
```

**Step 3 — census by type,** to see *what* rather than *where*:

```python
from collections import Counter
def census() -> Counter[str]:
    gc.collect()
    return Counter(type(o).__name__ for o in gc.get_objects())
```

**Step 4 — find the retainer.** Once you know the type, pick an instance and walk backwards:

```python
def retainers(obj, depth: int = 3, seen=None):
    seen = seen if seen is not None else set()
    if depth == 0 or id(obj) in seen:
        return
    seen.add(id(obj))
    for r in gc.get_referrers(obj):
        if isinstance(r, dict) and any(r is getattr(m, "__dict__", None) for m in ()):
            continue
        yield type(r).__name__, repr(r)[:100]
        yield from retainers(r, depth - 1, seen)
```

Crude and noisy — it will show you frames and the local dict of the function you are
standing in. Filtering that noise is exercise C1, and doing so teaches you what a reference
graph actually looks like.

**Step 5 — confirm the fix by re-running step 2.** Not by reasoning. Retention bugs are
where confident reasoning goes to die.

## 4. Failure modes

- **`lru_cache` on a method.** `functools.lru_cache` keys on all arguments *including
  `self`*, so every instance ever passed is retained forever. Use `cached_property` for
  per-instance memoization, or a `WeakKeyDictionary`, or accept and bound it explicitly.
  This is easily the most common accidental leak in Python codebases.
- **Storing exceptions.** §2.3 (4).
- **Unbounded caches.** Everything is fine until the cardinality of the key space is
  higher than you assumed.
- **`__del__` for resource cleanup.** §2.5.
- **`weakref.finalize` capturing `self`.** The finalizer never fires.
- **`weakref.ref(obj.method)`.** Dead immediately. Use `WeakMethod`.
- **`sys.getsizeof` for deep structures.** Reports the container, not the contents.
- **Assuming freed memory returns to the OS.** §2.6.
- **Benchmarking memory without `gc.collect()` first.** Noise swamps signal.
- **Reference cycles through `self` in a callback.** `self.timer = Timer(self.on_tick)` —
  the bound method holds `self`, `self` holds the timer. Common in GUI and async code.

## 5. Exercises

### Warm-up (25 min)

**W1.** Create a two-object cycle, `del` both names, and show with `gc.collect()`'s return
value that the collector found them. Then add `__del__` to both and repeat, reporting any
difference on your version.

**W2.** Demonstrate the `lru_cache`-on-a-method leak with a measurable memory increase.

**W3.** Show that `weakref.ref(obj.method)()` returns `None` immediately, and fix it.

### Core (2.5 h)

**C1 — A usable retainer walker.** Complete §3 step 4 so that it produces a readable chain
from a leaked object back to a root (module global, frame, or class attribute), filtering
out its own machinery. Test it on four planted leaks of different shapes: a module cache, a
closure, a retained traceback, and a suspended generator. Deliverable: the tool plus a
one-page "how to read the output" guide.

**C2 — The GC latency experiment.** Build a service-shaped workload with a large long-lived
object graph (say 5 million small objects) and a per-request allocation pattern. Measure
p50/p99/p999 request latency with: default GC, tuned thresholds, `gc.freeze()` after
warm-up, and GC disabled with periodic manual collection. Report a table. Then state the
conditions under which you would deploy each configuration, and what monitoring you would
add first.

**C3 — Fragmentation.** Construct a workload that allocates 2 GB, frees 95% of it, and does
not return memory to the OS. Prove it with RSS measurements. Then change the data structure
(to `array`, `bytes`, or NumPy) so that memory *is* returned, and explain why, referring to
§2.6.

**C4 — Resource lifetime, three ways.** Implement a `Connection` class three ways: context
manager only, `__del__`, and `weakref.finalize` + context manager. For each, write tests
covering: normal use, exception during use, caller forgets to close, and interpreter
shutdown with the object still alive. Report which combinations behave correctly. Then
find the equivalent pattern in the standard library (`socket`, `tempfile.TemporaryDirectory`,
`subprocess.Popen`) and compare.

### Challenge

**X1.** Read PEP 442 (safe object finalization). Construct a cycle containing an object with
`__del__` whose finalizer observes another cycle member *after* that member has been
finalized. Show the resulting misbehaviour and explain why PEP 442 does not — and cannot —
prevent it.

**X2.** Instrument a real service (yours, or a substantial open-source one) with
`tracemalloc` and produce a memory-attribution report for a realistic workload. Identify
the top three retention sources, propose fixes, and measure. Write it up as an incident
report: symptom, hypothesis, evidence, fix, verification. This format is exactly what
CA-731 L07 will ask for at scale.

## 6. Self-check

1. Why can reference counting not collect cycles?
2. Describe the subtractive-count algorithm the cycle collector uses.
3. What does a generation-2 collection cost, and why does that matter for a service?
4. What does `gc.freeze()` do and when would you call it?
5. List five things that commonly keep objects alive unintentionally.
6. Give three reasons not to use `__del__` for cleanup, and the correct alternative.
7. Why must a `weakref.finalize` callback not reference the object?
8. Why does freeing 95% of a large allocation often not reduce RSS?

## 7. Primary sources

- `gc`, `weakref`, `tracemalloc` module documentation.
- PEP 442 — Safe object finalization. PEP 445 — customizing memory allocators.
- CPython `Objects/obmalloc.c` header comment — the definitive description of pymalloc's
  arena/pool/block structure.
- Instagram engineering, "Dismissing Python Garbage Collection at Instagram" (2017) and
  "Copy-on-write friendly Python garbage collection" (2018) — the origin of `gc.freeze()`.
- Ramalho, *Fluent Python* 2e, ch. 6 (the "Del and Garbage Collection" section).

---

**Previous:** [L08](L08-bytecode-and-the-evaluation-loop.md) · **Next:**
[L10 — Exceptions, Control Flow, and Cleanup Semantics](L10-exceptions-and-cleanup-semantics.md)
