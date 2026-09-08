# PY-601 · Lesson 02 — Threads, the GIL, and the Free-Threaded Build

**Estimated study time:** 4 hours
**Prerequisites:** L01; PY-501 L08, L09

---

## 1. Orientation

The Global Interpreter Lock is the most discussed and least understood aspect of Python.
Two claims are both common and both wrong:

- *"The GIL makes Python code thread-safe."* It does not. `counter += 1` is a race under
  the GIL (L03), and every check-then-act sequence is a bug waiting for a switch interval.
- *"The GIL makes threads useless."* It does not. Every blocking I/O call releases it, as do
  NumPy, `hashlib`, compression, and most C extensions during their work.

What the GIL actually is: **a mutex protecting CPython's internal interpreter state — most
importantly reference counts — so that only one thread executes Python bytecode at a time.**
Understanding that sentence precisely determines what threads can and cannot do for you, and
it explains why removing the GIL took thirty years.

## 2. Theory

### 2.1 What the GIL protects

Every CPython object has a reference count updated on essentially every operation
(PY-501 L09 §2.1). Without synchronization, two threads incrementing a refcount concurrently
can lose an update, and a lost decrement means a use-after-free — a segfault, not an
exception.

Options for protecting refcounts:

1. **One global lock.** Cheap in the uncontended single-threaded case, which is the case
   CPython optimizes for. This is the GIL.
2. **A lock per object.** Enormous overhead: two lock operations per attribute access, and
   deadlock risk between objects.
3. **Atomic increments.** Cheaper than a lock but still an atomic RMW on a shared cache line
   for *every* reference operation — measured at roughly 20–40% single-thread slowdown in
   the historical attempts, plus severe cache-line contention when threads touch the same
   objects.
4. **Biased reference counting + deferred/immortal objects.** The approach PEP 703 takes.

The GIL also protects other interpreter state: the memory allocator, internal caches, and
the C-level structures of built-in types.

### 2.2 How the GIL is scheduled

A thread holds the GIL and releases it in two circumstances:

- **Voluntarily**, around a blocking call. Every I/O operation in the standard library wraps
  its syscall in `Py_BEGIN_ALLOW_THREADS` / `Py_END_ALLOW_THREADS`. So do `time.sleep`, and
  most well-written C extensions during long computations.
- **Involuntarily**, when another thread requests it and the **switch interval** elapses.
  Default 5 ms, settable with `sys.setswitchinterval()`.

The mechanism since Python 3.2 (Antoine Pitrou's rewrite): a waiting thread sets a
"gil_drop_request" flag; the holder checks it at bytecode boundaries and drops the GIL.
This replaced the pre-3.2 scheme of releasing every N bytecodes, which produced pathological
behaviour — Beazley's famous demonstration that two CPU-bound threads could run *slower*
than one, especially on multicore, because of a convoy effect.

Three things follow that matter in practice:

- **A switch can happen between any two bytecode instructions.** So any operation compiling
  to more than one bytecode is not atomic (L03 §2.2).
- **A CPU-bound thread can starve an I/O thread's response latency** by up to the switch
  interval. A thread that has just received data may wait 5 ms to be scheduled. For a
  latency-sensitive service with any CPU-bound background work, this is a real and
  frequently-missed source of tail latency. Lowering the switch interval reduces it at the
  cost of more switching overhead.
- **The GIL is not fair.** There is no queue; a released GIL may be immediately reacquired
  by the same thread. Starvation is possible.

### 2.3 What this means for your workload

| Work | GIL held? | Threads help? |
|---|---|---|
| `socket.recv`, `file.read`, `time.sleep` | released | **yes** — this is the main case |
| `hashlib`, `zlib`, `bz2` on large data | released | yes |
| NumPy elementwise ops, BLAS calls | released | yes |
| `json.loads` (C accelerated) | **held** | no |
| `re` matching | held | no |
| Pure Python loops | held | no |
| `pickle.dumps` | mostly held | no |

The rule: **threads help when the work is done outside the interpreter.** The
counterintuitive entries — `json.loads` and `re` — are C code that does not release the
GIL, because the operations are usually short and releasing/reacquiring has a cost. For a
service parsing large JSON payloads, this is a real serialization point and a reason to
consider a process pool or `msgspec`/`orjson` (which also hold it, but for less time).

### 2.4 Free-threading (PEP 703)

PEP 703, accepted in 2023, adds a build of CPython with the GIL removed. It shipped as an
experimental build in 3.13 (`--disable-gil`), and PEP 779 defined the criteria for it to
become officially supported, which landed in the 3.14 cycle. *Check your version's release
notes rather than relying on this paragraph — this area is moving.*

How it makes reference counting safe without a global lock:

- **Biased reference counting.** Each object has an "owning" thread; that thread updates its
  count with cheap non-atomic operations. Other threads use a separate, atomic shared count.
  The two are merged when needed. Most objects are only ever touched by one thread, so most
  refcount operations stay cheap.
- **Immortal objects** (PEP 683). `None`, `True`, `False`, small ints, interned strings, and
  type objects get a sentinel refcount and are never counted at all. This removes contention
  on the hottest shared objects.
- **Deferred reference counting** for objects reachable from the interpreter's internals.
- **Per-object locks** for mutable containers, plus internally lock-free reads for `dict`
  and `list` in the common case.
- **A different memory allocator** (mimalloc) that is thread-safe and supports the above.

What changes for you:

- **True parallel execution of Python bytecode.** CPU-bound multithreading finally works.
- **Data races become possible at the Python level.** Under the GIL, `list.append` is atomic
  because it is one bytecode holding the GIL. Free-threaded builds keep individual
  container operations safe via per-object locks — but *sequences* of operations were never
  atomic and now the windows are much wider and genuinely concurrent. Code that was
  accidentally correct will break.
- **Single-thread performance cost.** The early figures were roughly 5–10% for the
  free-threaded build versus the default build, improving over releases. Measure on your own
  workload.
- **C extensions must opt in** (`Py_MOD_GIL_NOT_USED`) or the interpreter re-enables the GIL
  at runtime. Ecosystem readiness is the real constraint, not the interpreter.

**The critical mental shift:** the GIL was never a correctness feature, but it *was* an
accident that made some incorrect code work. Free-threading removes the accident. Everything
in L03 becomes load-bearing.

### 2.5 The other approaches to parallelism

Free-threading is not the only path, and knowing the alternatives is part of choosing well.

**Multiple processes.** Total isolation, true parallelism, and the cost of serialization and
memory (L08).

**Per-interpreter GIL (PEP 684) and subinterpreters in the stdlib (PEP 734).** Multiple
interpreters in one process, each with its own GIL and its own object space. Parallelism
without process overhead, and with cheaper (though not free) data transfer. The
`interpreters` module exposes this. Constraints: no object sharing between interpreters
except through defined channels; and many C extensions are not per-interpreter-safe.

**Release the GIL in C/Rust.** Cython's `nogil` blocks, `PyO3`'s `allow_threads`, and any
extension doing long work outside the interpreter. This is how NumPy, `hashlib`, and
`polars` get parallelism today, and it remains the highest-leverage option for numeric work
(PY-602 L07).

**Do the work elsewhere.** A vectorized NumPy operation, a database aggregate, or a query
pushed down to a columnar engine avoids the question entirely, and usually wins by more than
any threading scheme (PY-602 L05).

### 2.6 Threads in Python: the practical API

```python
from concurrent.futures import ThreadPoolExecutor, as_completed

with ThreadPoolExecutor(max_workers=32) as pool:
    futures = {pool.submit(fetch, url): url for url in urls}
    for fut in as_completed(futures):
        try:
            result = fut.result()
        except Exception:
            log.exception("failed: %s", futures[fut])
```

Things to know that the tutorials skip:

- **`Executor.map` swallows nothing but reorders nothing either** — it yields in input
  order, so a slow first item blocks the rest. `as_completed` yields in completion order.
- **Exceptions are captured in the Future** and re-raised at `.result()`. A future whose
  result is never retrieved swallows its exception *silently*. Always retrieve, or use
  `add_done_callback` to log.
- **`shutdown(wait=True)` is the default on context exit**, and it does not cancel running
  work. `cancel_futures=True` (3.9+) cancels queued-but-not-started items.
- **Thread pool sizing** for I/O is not "number of cores". It is bounded by the concurrency
  the *downstream* can absorb, by memory per thread (~8 MB of stack address space, though
  usually little resident), and by the file-descriptor limit. Deriving it from Little's Law
  (L09 §2.2) is the principled approach.
- **`threading.local()`** gives per-thread storage. In a thread pool it is per-*worker*, not
  per-task, so state leaks between tasks. Use `contextvars` instead (PY-502 L07 §2.7).
- **Daemon threads are killed abruptly at interpreter exit** with no cleanup, mid-operation.
  Do not use them for anything holding a resource.
- **`threading.excepthook`** — an uncaught exception in a thread does not terminate the
  process and, by default, only prints. Set the hook, or you will lose failures.

## 3. Construction: measuring the GIL

**Experiment 1 — CPU-bound threads.** A pure-Python `fib(30)` in 1, 2, 4, 8 threads. Measure
wall clock and total CPU. You should see wall clock roughly constant or *worse* with more
threads, and CPU time growing (the extra is switching overhead and contention). Explain the
overhead.

**Experiment 2 — I/O-bound threads.** The same shape with `time.sleep(0.1)`. Near-perfect
scaling. Explain why in one sentence.

**Experiment 3 — the switch interval and latency.** Run one CPU-bound thread and one thread
that receives from a socket and timestamps the delay between data arriving and the thread
processing it. Measure the p99 of that delay at `sys.setswitchinterval()` values of 0.005
(default), 0.001, and 0.05. Plot latency and total throughput against the interval. **This
experiment explains a class of production tail-latency mystery** and almost nobody runs it.

**Experiment 4 — what releases the GIL.** For each of `hashlib.sha256`, `json.loads`,
`re.match`, `zlib.compress`, and a NumPy operation, run two threads doing the operation on
large inputs, and compare wall clock with the sequential case. Build the table of §2.3 from
your own measurements rather than trusting it.

**Experiment 5 — free-threaded.** If you can obtain a free-threaded build (§2.4), repeat
experiments 1 and 4. Then run a deliberately racy program — an unsynchronized
`counter += 1` across eight threads, ten million increments — on both builds and report the
error magnitude. Under the GIL you will lose some updates; free-threaded you will lose far
more. Neither is correct; the point is that the GIL made a bug *rare* rather than absent,
which is the worst possible property for a bug.

## 4. Failure modes

- **Believing the GIL provides thread safety.** L03 is about this.
- **Using threads for CPU-bound Python and expecting speedup.**
- **Not retrieving a Future's result**, and losing its exception silently.
- **`Executor.map` with a slow first item**, blocking everything behind it.
- **Unbounded thread pools.** Memory and file descriptors are finite; the downstream is
  finite too.
- **`threading.local` in a pool**, leaking state between tasks.
- **Daemon threads holding resources.**
- **No `threading.excepthook`**, so thread failures are invisible.
- **Ignoring switch-interval latency** in a service with CPU-bound background work.
- **Assuming C extension calls release the GIL.** `json` and `re` do not.
- **Porting to free-threaded without auditing shared state.** Accidentally-correct code
  breaks.

## 5. Exercises

### Warm-up (25 min)

**W1.** Demonstrate that two CPU-bound threads are not faster than one, and that two
I/O-bound threads are.

**W2.** Submit a task that raises to a `ThreadPoolExecutor` and never call `.result()`. Show
that the exception vanishes. Then fix it three ways.

**W3.** Show that `threading.local` in a pool leaks state between tasks, and that
`contextvars` does not.

### Core (2.5 h)

**C1 — The five experiments.** Complete §3, including experiment 3 (the switch-interval
latency curve) and experiment 5 if a free-threaded build is available. Deliverable: plots,
tables, machine details, and a 600-word note on which result most changed your mental model.

**C2 — Pool sizing.** For a real I/O-bound workload, derive the right pool size from Little's
Law (arrival rate × service time), then measure throughput and p99 latency at 0.5×, 1×, 2×,
and 4× that size. Report the curve and where it turns down (L01 §2.7's coherency term).
Explain the physical cause.

**C3 — GIL release in an extension.** Write a small Cython or Rust (PyO3) function performing
a long computation, once holding the GIL and once releasing it. Measure two-thread scaling
for each. Report the code difference and the speedup.

**C4 — Free-threading readiness audit.** Take a module with shared state and audit it as if
porting to a free-threaded build: every module-level mutable, every class attribute mutated
after import, every lazily-initialized cache, every check-then-act. Produce the list and the
fix for each. Report how many were relying on GIL atomicity.

### Challenge

**X1.** Read PEP 703 in full. Write 1,500 words explaining: biased reference counting, why
immortal objects matter, what per-object locking buys and costs, and the single-thread
performance trade. Then predict — and, if you can, measure — which of your own workloads
would benefit and which would regress.

**X2.** Reproduce Beazley's "Understanding the GIL" experiments on a modern interpreter and
report what has and has not changed since 3.2. In particular, look for convoy effects with
mixed CPU and I/O threads and for the effect of thread pinning. Write it up as a short
paper with plots.

## 6. Self-check

1. State precisely what the GIL protects and why refcounting made it necessary.
2. Give the two circumstances in which a thread releases the GIL.
3. What is the switch interval, and what latency effect does it cause?
4. Name three C-accelerated operations that do *not* release the GIL.
5. Explain biased reference counting and immortal objects.
6. What breaks when the GIL is removed, and why is "it made a bug rare" the worst property?
7. Name four approaches to parallelism in Python other than free-threading.
8. Why does an unretrieved Future swallow its exception, and what do you do about it?

## 7. Primary sources

- PEP 703 (making the GIL optional), PEP 683 (immortal objects), PEP 684 (per-interpreter
  GIL), PEP 734 (multiple interpreters in the stdlib), PEP 779 (free-threaded support
  criteria).
- Beazley, "Understanding the Python GIL" (PyCon 2010) — still the clearest exposition of
  the pre-3.2 problem and the fix.
- Pitrou's 3.2 GIL rewrite notes (python-dev, 2009).
- CPython `Python/ceval_gil.c`.

---

**Previous:** [L01](L01-concurrency-parallelism-correctness.md) · **Next:**
[L03 — Shared State, Locks, and the Memory Model](L03-shared-state-and-locks.md)
