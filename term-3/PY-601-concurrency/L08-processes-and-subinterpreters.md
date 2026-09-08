# PY-601 · Lesson 08 — Processes, Subinterpreters, and Shared Memory

**Estimated study time:** 4 hours
**Prerequisites:** L02, L04
**Platform note:** several exercises need `fork`, which does not exist on Windows. Use WSL2
(see `00-program/lab-setup.md`, step A9).

---

## 1. Orientation

Processes are the oldest answer to the GIL and still the most reliable one. They give you
true parallelism, complete isolation, and independent failure — at the cost of memory,
startup time, and the fact that **everything you pass between them must be serialized**.

That last cost is the one that decides most designs. A process pool that speeds up your
computation 8× and spends 90% of its time pickling is a common and entirely avoidable
outcome. This lesson is mostly about knowing where the data goes.

## 2. Theory

### 2.1 Start methods

`multiprocessing` supports three, and they behave very differently:

| Method | How | Available | Notes |
|---|---|---|---|
| `fork` | copy the parent process | Unix only | fastest; inherits everything; **unsafe with threads** |
| `spawn` | start a fresh interpreter, re-import `__main__` | all platforms | slowest; clean; the default on macOS and Windows |
| `forkserver` | fork from a small clean server process | Unix | fast and safer than `fork`; the best default on Linux |

```python
import multiprocessing as mp
mp.set_start_method("forkserver")     # once, at program start, before anything else
```

**Why `fork` is dangerous in a threaded program.** `fork` duplicates only the calling
thread. Every other thread ceases to exist in the child — but the *state* they owned is
copied. A lock held by another thread at the moment of the fork is copied in the locked
state, and there is no thread to release it. The child deadlocks the first time it touches
that lock, which may be inside `malloc`, inside the logging module, or inside an SSL
library.

This is not theoretical: it is why `fork` after starting a thread pool, an HTTP client with
a connection pool, or a logging handler with a lock is a recurring production bug. CPython
emits a `DeprecationWarning` when `fork` is used in a multi-threaded process (3.12+), and
the default on Linux is moving away from `fork`. **Use `forkserver`.**

`spawn`'s consequences you must design for:

- The child **re-imports `__main__`**, so module-level side effects run again. Hence the
  `if __name__ == "__main__":` guard, which is not a style convention — without it you get
  infinite process creation.
- Everything passed to the child is **pickled**, including the target function (by reference:
  module + qualified name). Lambdas, closures, and locally-defined classes cannot be sent.
- Startup is 10–100× more expensive than `fork` (a fresh interpreter, re-imported modules).
  For short tasks this dominates.

### 2.2 The cost of moving data

Everything crossing a process boundary is serialized. The default is `pickle`, and the cost
is real:

- Round-trip a 100 MB NumPy array: pickling copies it, writes it to a pipe, reads it, and
  reconstructs — hundreds of milliseconds and 200 MB of transient memory.
- Round-trip a large dict of Python objects: far worse, because pickle walks the object
  graph.

The design rules that follow:

1. **Send parameters, not data.** Give the worker a file path, an offset, a query — let it
   read what it needs.
2. **Chunk coarsely.** `pool.map(f, items)` with a million tiny items pickles a million
   times. `chunksize` exists for this; set it deliberately (a good starting point is
   `len(items) // (4 * n_workers)`).
3. **Return summaries, not data.** A worker that returns a 50 MB result has undone the
   parallelism.
4. **Use shared memory for large arrays** (§2.4).
5. **Consider a faster serializer** — `pickle` protocol 5 with out-of-band buffers
   (PEP 574) avoids copies for buffer-supporting objects; `msgspec` or `orjson` for
   structured data.

**Measure the serialization fraction before optimizing anything else.** In most disappointing
process-pool results, it is over half the wall clock.

### 2.3 `concurrent.futures.ProcessPoolExecutor`

```python
from concurrent.futures import ProcessPoolExecutor

if __name__ == "__main__":
    with ProcessPoolExecutor(max_workers=8, mp_context=mp.get_context("forkserver")) as ex:
        for result in ex.map(work, items, chunksize=64):
            ...
```

Things to know:

- **`max_workers` defaults to the CPU count** (and is capped at 61 on Windows). For CPU-bound
  work that is right; for anything that also waits, it is not.
- **`initializer=` runs once per worker** — the place to open a database connection, load a
  model, or import a heavy module, so the cost is paid per worker rather than per task.
- **A worker that dies** (segfault, OOM kill) raises `BrokenProcessPool` and the entire pool
  becomes unusable. There is no per-task recovery. If your work can crash a worker, run it
  behind a supervisor that recreates the pool, or use a real task queue.
- **Exceptions are pickled back.** A custom exception whose `__init__` signature does not
  match `args` fails to unpickle and you get a confusing error instead of the real one
  (PY-501 L10 §2.7). Test that your exceptions round-trip.
- **`ex.map` with a generator input** consumes it eagerly. For a huge input, submit in
  batches instead, or you will materialize everything.

### 2.4 Shared memory

`multiprocessing.shared_memory` (3.8+) gives a named block of memory mapped into several
processes, with no copying:

```python
from multiprocessing import shared_memory
import numpy as np

# parent
a = np.arange(10_000_000, dtype=np.float64)
shm = shared_memory.SharedMemory(create=True, size=a.nbytes)
buf = np.ndarray(a.shape, dtype=a.dtype, buffer=shm.buf)
buf[:] = a[:]
# pass shm.name, a.shape, a.dtype to the workers

# worker
shm = shared_memory.SharedMemory(name=name)
arr = np.ndarray(shape, dtype=dtype, buffer=shm.buf)
...
shm.close()          # every process that opened it
# parent only, at the end:
shm.unlink()         # destroy the block
```

The rules, each of which corresponds to a real failure:

- **Every process `close()`s; exactly one `unlink()`s.** Forgetting `unlink` leaks a block
  that survives the process — on Linux it sits in `/dev/shm` until reboot.
- **`close()` before the parent `unlink()`s**, or you get a use-after-free.
- **No synchronization is provided.** Concurrent writes to the same region are a data race,
  in the C sense — undefined behaviour, not a Python-level surprise. Use
  `multiprocessing.Lock` or partition the array so each worker owns a disjoint slice
  (much better).
- **The resource tracker** may warn about leaked blocks at exit; those warnings are usually
  telling you the truth.

For a pipeline of NumPy arrays this converts a copy-per-task into a pointer-per-task, and
the speedup is often larger than the parallelism itself.

Alternatives worth knowing: memory-mapped files (`mmap`) when the data has a file backing;
Arrow IPC / Plasma for columnar data shared across languages; and `multiprocessing.Array`
/`Value` for small shared scalars with built-in locks.

### 2.5 Subinterpreters

PEP 684 gave each interpreter its own GIL; PEP 734 added a stdlib `interpreters` module.
This is the middle ground between threads and processes:

```python
import interpreters                       # names/API vary by version — check the docs

interp = interpreters.create()
interp.exec("import mymodule; mymodule.work()")
```

| | Threads | Subinterpreters | Processes |
|---|---|---|---|
| Parallel Python bytecode | only free-threaded | **yes** | yes |
| Memory per unit | ~8 MB stack | ~2–10 MB | ~20–60 MB + imports |
| Startup | ~50 µs | ~1 ms | ~30–100 ms (spawn) |
| Shared objects | yes (dangerous) | **no** | no |
| Crash isolation | none | partial | full |
| C extension support | universal | limited | universal |

The constraints are real: objects cannot be shared between interpreters (data crosses via
channels, with copying or via buffers), and a C extension must be built to be
per-interpreter-safe (`Py_mod_multiple_interpreters`) or it cannot be imported in a
subinterpreter at all. As of the mid-2020s much of the scientific stack was not yet ready.

Where subinterpreters win: many isolated, CPU-bound, pure-Python tasks in one process —
a plugin host, a template renderer, a rules engine. Where they lose: anything depending on
a heavy C extension, and anything needing to share large data.

### 2.6 Choosing

```
Is the work I/O-bound?
├── yes → asyncio (or threads); processes are wrong
└── no  → is it in C (NumPy, hashlib, a Rust extension)?
          ├── yes → threads work; the GIL is released
          └── no  → is the data large?
                    ├── yes → processes + shared memory
                    └── no  → is a free-threaded build available and are your deps ready?
                              ├── yes → threads
                              └── no  → is the task pure-Python and isolated?
                                        ├── yes → subinterpreters
                                        └── no  → processes
```

And one branch above all of these: **can you avoid the computation?** A vectorized NumPy
operation, a database aggregate, a better algorithm (CS-621), or a cache beats every
parallelism scheme, and PY-602 L01 insists you check before parallelizing.

### 2.7 Process supervision

Once you have processes, you have process management:

- **Zombie processes.** A child that exits and is never `wait()`ed stays in the process
  table. `multiprocessing` handles this for its own children; `subprocess.Popen` does not
  unless you `wait()` or `poll()`.
- **Orphans.** If the parent dies, children keep running. On Linux,
  `prctl(PR_SET_PDEATHSIG, SIGTERM)` makes a child die with its parent; there is no portable
  equivalent, so a process-group kill at shutdown is the usual answer.
- **Signal propagation.** `SIGTERM` to a process group hits everyone; to a single PID it
  does not reach children. Use `os.setsid()` / `start_new_session=True` and kill the group.
- **OOM.** The Linux OOM killer chooses a victim by score, and it is frequently not the
  process that allocated. A worker killed by the OOM killer disappears with no Python-level
  error — `BrokenProcessPool` with no traceback is the usual symptom, and `dmesg` is where
  the evidence is.
- **Resource limits.** `resource.setrlimit(RLIMIT_AS, ...)` in a worker's initializer bounds
  its memory so that a runaway task fails cleanly instead of taking down the machine.

## 3. Construction: a parallel pipeline that is actually faster

Take a genuinely CPU-bound job over large data — image resizing, feature extraction, parsing
a large corpus.

**Stage 1 — sequential baseline.** Measure. Profile (PY-602 L02). Find the actual hot spot.
Then ask the L01 question: is this CPU-bound in *Python*, or in a C library that already
releases the GIL?

**Stage 2 — the naive process pool.** `ProcessPoolExecutor`, `ex.map(work, items)`, no
`chunksize`, data passed as arguments and returned as results.

Measure. Common outcome: 2× speedup on 8 cores, or slower than sequential. Then find out
why:

- Time the serialization explicitly: pickle one item, time it, multiply by the item count.
- Compare `chunksize=1` (the default for `map` with an iterable of unknown length) against
  a computed chunk size.
- Measure worker startup: how long before the first result?

**Stage 3 — fix the data movement.** Apply §2.2's rules. Typically: pass file paths and
offsets instead of data; return a summary; set `chunksize`; use `initializer` to load the
model once per worker. Re-measure after each change and record the individual contributions
— that table is the deliverable.

**Stage 4 — shared memory.** For the array-shaped part of the data, move to
`shared_memory` with each worker owning a disjoint slice. Measure again. Verify with
`unlink` discipline that nothing leaks (`ls /dev/shm` before and after).

**Stage 5 — the scalability curve.** Throughput at 1, 2, 4, 8, 16, 32 workers on an 8-core
machine. Find where it turns down and explain it in terms of L01 §2.7's coherency term.
Candidates: memory bandwidth, the parent process's serialization becoming the bottleneck,
page cache pressure, or hyperthread contention.

**Stage 6 — robustness.** Kill a worker mid-run (`kill -9`) and observe `BrokenProcessPool`.
Then build a supervisor that recreates the pool and re-queues the lost work — which requires
you to know *which* work was lost, which requires tracking submissions. Report what this
costs.

**Stage 7 — compare with the alternatives.** Same workload with: threads (to demonstrate the
GIL), a free-threaded build if available, subinterpreters if your dependencies permit, and —
most importantly — a *vectorized or algorithmically better* sequential version. Report all
of them. The last one winning is a common and instructive outcome.

## 4. Failure modes

- **`fork` in a threaded process.** Deadlock in the child, in a library, at random.
- **Missing `if __name__ == "__main__":` with `spawn`.** Infinite process creation.
- **Pickling large data per task.** The parallelism is consumed by serialization.
- **`chunksize=1` on a million small items.**
- **Lambdas or closures as pool targets.** Not picklable.
- **Returning large results.** Undoes the win.
- **Shared memory not `unlink`ed.** Leaks past process exit.
- **Concurrent writes to shared memory with no partitioning or lock.** A real data race.
- **A worker crash killing the pool** with no recovery path.
- **Custom exceptions that do not round-trip through pickle.**
- **No `RLIMIT_AS`**, so one runaway worker takes down the host.
- **Zombie or orphan processes** at shutdown.
- **Parallelizing before profiling.** The most expensive mistake in this lesson.

## 5. Exercises

### Warm-up (30 min, Linux/WSL/macOS)

**W1.** Demonstrate the fork-with-threads deadlock: start a thread that holds a lock, fork,
and have the child try to acquire it. Then show `forkserver` avoids it.

**W2.** Measure the startup cost of `fork`, `forkserver`, and `spawn` by timing 100 trivial
tasks under each.

**W3.** Time pickling a 100 MB NumPy array round-trip. Compare with passing it through
`shared_memory`.

### Core (2.5 h)

**C1 — The pipeline.** Complete §3, all seven stages. Deliverable: the per-change measurement
table from stage 3, the shared-memory result, the scalability curve with the turn-down
explained, the crash-recovery implementation, and the seven-way comparison from stage 7 with
a recommendation.

**C2 — Serialization budget.** For a realistic workload, measure what fraction of wall clock
is serialization at chunk sizes of 1, 10, 100, and 1000. Plot. Derive a rule for choosing
`chunksize` and state its assumptions.

**C3 — Supervision.** Build a worker pool that survives: a worker segfault, a worker OOM
kill, a worker that hangs, and `SIGTERM` to the parent. For each, the lost work must be
detected and re-queued, and the process group must exit cleanly. Test all four by injecting
the failure.

**C4 — Subinterpreters.** If your Python version supports them, run a pure-Python CPU-bound
workload across subinterpreters and compare with threads and processes on: throughput,
memory, and startup. Report which of your project's dependencies can be imported in a
subinterpreter and which cannot.

### Challenge

**X1.** Build a shared-memory ring buffer for zero-copy transfer of frames between a producer
process and N consumers, with correct synchronization (a lock plus condition variables in
shared memory, or an index-based lock-free scheme with the ABA problem addressed). Test for
correctness under kill -9 of a consumer. Report what you could not make safe and why.

**X2.** Take a real workload currently using a process pool. Instrument it to attribute wall
clock to: worker startup, serialization out, computation, serialization back, and parent-side
aggregation. Produce the breakdown. Then optimize the largest non-computation component and
report the improvement. Publish the attribution method — it is reusable and almost nobody
has one.

## 6. Self-check

1. Give the three start methods with one property each, and say which should be the default
   on Linux.
2. Explain precisely why `fork` in a threaded process can deadlock the child.
3. Why does `spawn` require the `__main__` guard?
4. Give five rules for reducing data movement between processes.
5. State the `shared_memory` close/unlink discipline and what happens if you get it wrong.
6. Compare threads, subinterpreters, and processes on five dimensions.
7. What happens when a pool worker is OOM-killed, and where is the evidence?
8. Give the decision tree from §2.6, including the branch above all of it.

## 7. Primary sources

- PEP 684 (per-interpreter GIL), PEP 734 (multiple interpreters in the stdlib), PEP 574
  (pickle protocol 5 with out-of-band data), PEP 703.
- `multiprocessing` documentation — the "Contexts and start methods" and "Programming
  guidelines" sections in full; the latter is a list of real bugs.
- Kerrisk, *The Linux Programming Interface*, chs. 24–28 (process creation, signals) and 49
  (mmap).
- CPython `Lib/multiprocessing/` and `Lib/concurrent/futures/process.py`.

---

**Previous:** [L07](L07-cancellation-and-shutdown.md) · **Next:**
[L09 — Backpressure, Rate Limiting, and Flow Control](L09-backpressure-and-flow-control.md)
