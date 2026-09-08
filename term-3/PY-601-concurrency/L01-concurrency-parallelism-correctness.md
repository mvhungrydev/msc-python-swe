# PY-601 · Lesson 01 — Concurrency, Parallelism, and Models of Correctness

**Estimated study time:** 3.5 hours
**Prerequisites:** none

---

## 1. Orientation

Rob Pike's formulation is the one to keep:

> **Concurrency is about dealing with lots of things at once. Parallelism is about doing
> lots of things at once.**

Concurrency is a *structuring* property: the program is composed of independently
progressing activities. Parallelism is an *execution* property: multiple things happen
simultaneously on multiple processors.

A single-core machine can run a concurrent program. A parallel program is necessarily
concurrent. And — the point that matters for Python — **the two have different bottlenecks
and different solutions**, so mistaking one for the other leads directly to reaching for the
wrong tool.

## 2. Theory

### 2.1 Why the distinction decides your tool

| Workload | Bottleneck | Right tool in Python |
|---|---|---|
| Many network calls | waiting | `asyncio` (or threads) — concurrency, no parallelism needed |
| Many file reads | waiting on the kernel | threads (the GIL is released during I/O) |
| Heavy numeric computation | CPU | processes, or a C/Rust extension releasing the GIL, or NumPy |
| Many independent CPU-light tasks | scheduling overhead | `asyncio` |
| Mixed | both | async front end + process pool for the CPU stage |

The common failure: using threads for CPU-bound Python work and finding it *slower* than
sequential (contention plus GIL handoff overhead), or using multiprocessing for I/O-bound
work and paying serialization and process overhead for nothing.

The diagnostic is not "how many things", it is **what is the program waiting for?** Measure
before choosing (PY-602 L01). A CPU profile that shows 90% of time in `socket.recv` says
concurrency; one that shows 90% in your own bytecode says parallelism.

### 2.2 What makes concurrency hard

Three things, and every concurrency bug is one of them:

**Non-determinism.** The interleaving differs run to run. A bug may appear once in 10⁶ runs
and then three times in a minute under production load. Consequences: you cannot reproduce
it by rerunning, and a passing test is weak evidence.

**Shared mutable state.** If nothing is shared, or nothing is mutable, there is no problem.
Every concurrency bug involves state that is both.

**Composition failure.** Two individually thread-safe operations composed are not thread-safe:

```python
if key not in d:        # thread-safe
    d[key] = value      # thread-safe
                        # together: a race
```

This is the deepest of the three. Thread safety is not a property that composes, which means
"we use a thread-safe dict" is not an argument that your code is correct.

### 2.3 Safety and liveness

Every correctness property of a concurrent system is one of two kinds (Lamport, 1977):

- **Safety** — "nothing bad ever happens". Mutual exclusion. No lost update. No two
  bookings for one seat. A safety property is violated by a *finite* execution prefix, so
  a counterexample is a finite trace you can exhibit.
- **Liveness** — "something good eventually happens". Every request eventually gets a
  response. No deadlock. No starvation. A liveness violation requires an *infinite*
  execution, which is why testing rarely finds them and why model checking (FM-751 L03) is
  the tool that does.

Every specification you write should say which properties are safety and which are liveness.
Most bugs people fear are safety violations; most production incidents involving concurrency
are liveness failures (a deadlock, a starving queue, a livelock of retries).

### 2.4 The correctness conditions for concurrent objects

**Linearizability** (Herlihy & Wing, 1990) is the gold standard. An execution is
linearizable if each operation appears to take effect **instantaneously at some point
between its invocation and its response**, and that point-ordering is consistent with a
correct sequential execution of the object.

Two properties make it the right definition:

- **It is a local property**: if every object in a system is linearizable, the system is
  linearizable. This is enormously useful — you can reason object by object.
- **It is non-blocking**: a pending operation never needs to wait for another to complete in
  order to be linearized.

Weaker conditions exist and are used deliberately:

- **Sequential consistency** — operations appear in *some* sequential order consistent with
  each thread's program order, but not necessarily respecting real time. Not composable.
- **Serializability** — the database analogue, over transactions rather than single
  operations. A transaction schedule is equivalent to *some* serial order.
- **Eventual consistency** — replicas converge if updates stop. Says nothing about what you
  read in the meantime.

DS-701 L04 takes these into the distributed setting. Here the point is: **when you say "my
queue is thread-safe", say what you mean.** Linearizable? Or merely "does not corrupt its
internal state"? Those are very different claims and only the first supports composition.

### 2.5 The classic hazards, precisely

**Race condition** — the result depends on timing. Note the generality: not all races
involve data corruption; a check-then-act race (§2.2) corrupts nothing at the memory level
and is still a bug.

**Data race** — a narrower, technical term: two threads access the same memory location
concurrently, at least one writes, and there is no synchronization ordering them. In
languages with a formal memory model (C++, Java, Rust, Go) a data race is *undefined
behaviour*. CPython with the GIL has no data races at the Python level, which is exactly why
Python programmers are unprepared for the free-threaded build.

**Deadlock** — a cycle of threads each holding a resource another needs. Requires all four
Coffman conditions simultaneously: mutual exclusion, hold-and-wait, no preemption, and
circular wait. Break any one and deadlock is impossible; the practical lever is almost
always **circular wait**, broken by a global lock ordering.

**Livelock** — threads are active but make no progress. Retry storms are the common form.

**Starvation** — a thread never gets a resource. Unfair locks and priority schemes cause it.

**Priority inversion** — a low-priority thread holding a lock blocks a high-priority one.
Famous for nearly ending the Mars Pathfinder mission.

**The lost wakeup** — a thread checks a condition, is about to wait, another thread signals,
and then the first waits forever. This is why condition variables require the lock to be
held across the check-and-wait, and why the wait must be in a `while` loop (L03 §2.5).

### 2.6 The models

Four ways to structure concurrent programs. Knowing all four means you notice when you are
forcing one where another fits.

**Shared memory + locks.** Threads share state; mutual exclusion protects invariants.
Ubiquitous, efficient, and the hardest to get right because the invariants are implicit and
the compiler will not check them.

**Message passing (CSP)** — Hoare, 1978. Processes communicate over channels; no shared
state. "Do not communicate by sharing memory; share memory by communicating." Go's
goroutines and channels; Python's `queue.Queue` and `asyncio.Queue` approximate it. Much
easier to reason about, at the cost of copying and of a less direct expression of some
algorithms.

**Actors** — Hewitt, 1973. Each actor has private state and a mailbox, processes one message
at a time, and may create actors and send messages. Erlang/Elixir's model, with supervision
trees as the fault-handling story. Excellent for stateful distributed systems.

**Data parallelism.** Apply the same operation to many data elements. NumPy, SIMD, GPU
kernels, MapReduce. No shared mutable state by construction; the parallelism is implicit in
the operation.

The practical guidance: **prefer message passing and data parallelism; use shared memory
plus locks only where you must, and then keep the shared region as small as possible.**
Most Python concurrency should be a pipeline of stages connected by bounded queues, which
is CSP, and which gets you backpressure for free (L09).

### 2.7 Amdahl, Gustafson, and Universal Scalability

**Amdahl's Law** (1967): if a fraction `p` of a program is parallelizable, speedup on `N`
processors is bounded by `1 / ((1-p) + p/N)`, and as `N → ∞` by `1/(1-p)`. Five percent
serial means a maximum speedup of 20×, no matter how many cores. The lesson: **the serial
fraction is the ceiling**, so find it and reduce it before adding hardware.

**Gustafson's Law** (1988): the counterpoint. In practice, larger machines are used for
larger problems, and the serial portion often does not grow with problem size. So scaled
speedup can be near-linear even with a non-trivial serial fraction. Both are right about
different questions: Amdahl asks "how much faster for a fixed problem?", Gustafson asks "how
much bigger a problem in fixed time?"

**Gunther's Universal Scalability Law** adds the term both omit: **coherency cost**. Beyond
contention (Amdahl's serial fraction), there is the cost of keeping N workers consistent
with each other, which grows as N². That is why real systems do not merely plateau — they
get *worse* past some N. Every engineer who has added workers and seen throughput drop has
met this term. Measure your scalability curve; if it turns down, you have a coherency
problem and adding capacity is actively harmful.

## 3. Construction: measuring the models

Take one workload — say, fetching 200 URLs and computing a hash of each body — and implement
it five ways. The point is the *measurement*, not the code.

1. **Sequential.**
2. **Threads** (`ThreadPoolExecutor`).
3. **`asyncio`** (`httpx.AsyncClient` + `TaskGroup`).
4. **Processes** (`ProcessPoolExecutor`).
5. **Hybrid**: async fetch, process pool for the hashing.

Measure: wall clock, CPU time, peak memory, and the number of OS threads/processes. Vary the
CPU cost of the per-item work from trivial to heavy, and plot.

What you should find, and should be able to *predict before running*:

- With trivial CPU work, async and threads are close and both crush sequential; processes
  lose to serialization overhead.
- As CPU work grows, threads flatten (the GIL serializes the Python-level work) while
  processes improve.
- There is a crossover; find it and explain it.
- The hybrid wins at the high end and costs complexity.

Then compute the *serial fraction* of each implementation from the measured speedup curve
(invert Amdahl) and compare with what you expected from reading the code. The discrepancy is
the lesson — the serial part is usually not where people think it is (it is often
serialization, or a shared connection pool, or the GIL handoff, not the "obvious" critical
section).

Finally: fit a Universal Scalability Law curve to the process version as you increase worker
count past the core count. Find where throughput turns down.

## 4. Failure modes

- **Threads for CPU-bound Python.** Slower than sequential, often.
- **Processes for I/O-bound work.** Pays serialization and memory for nothing.
- **Assuming "thread-safe" composes.** §2.2.
- **Not saying which correctness condition you mean.** §2.4.
- **Testing concurrency by running the test again.** Passing is weak evidence.
- **Ignoring liveness.** All the effort goes into safety; the outage is a deadlock.
- **Adding workers to fix a throughput problem** without measuring the scalability curve.
- **Mixing models.** Shared mutable state *and* queues, so neither discipline holds.
- **Reasoning about the GIL as if it made your code thread-safe.** L02 and L03 destroy this
  belief in detail.

## 5. Exercises

### Warm-up (25 min)

**W1.** Give three programs that are concurrent but not parallel, and one that is parallel
but (arguably) not concurrent. Defend the last one.

**W2.** Write a check-then-act race that corrupts no memory and still produces a wrong
result. Demonstrate it with two threads.

**W3.** For a system you know, classify five correctness properties as safety or liveness.

### Core (2 h)

**C1 — The five implementations.** Complete §3. Deliverable: the code, the plots (wall
clock and CPU vs per-item CPU cost), the crossover point with an explanation, the inferred
serial fraction versus your prediction, and the USL curve for the process version. Record
machine, OS, Python build.

**C2 — Model translation.** Take a piece of shared-memory-and-locks code and rewrite it in
the CSP style with queues. Compare: lines, number of shared mutable variables, ease of
reasoning, throughput, and latency. Report both directions of the trade honestly.

**C3 — The Coffman conditions.** Write a program that deadlocks. Then produce four
variants, each breaking exactly one Coffman condition. For each, state what it costs (lock
ordering costs flexibility; timeouts cost determinism; and so on) and which you would use in
production.

**C4 — Linearizability by hand.** Take a small concurrent object (a counter, a stack). Write
down two concurrent histories, one linearizable and one not, and prove each case by
exhibiting or ruling out a linearization point ordering. Then write a checker that verifies
linearizability of recorded histories by brute force for small cases, and use it on a
deliberately broken implementation.

### Challenge

**X1.** Read Herlihy & Wing (1990) §§1–4. Explain, in 1,000 words, why linearizability is a
*local* property and sequential consistency is not, why locality matters for building
systems, and what you give up by choosing a weaker condition.

**X2.** Measure the Universal Scalability Law on a real system you have access to: throughput
versus concurrency, from 1 to well past the point where it degrades. Fit the contention and
coherency coefficients. Identify the physical cause of the coherency term (a shared lock, a
cache line, a database row, a connection pool). Report what you would change.

## 6. Self-check

1. State the concurrency/parallelism distinction and give the diagnostic question for
   choosing a tool.
2. Name the three things that make concurrency hard, and say which is deepest.
3. Distinguish safety from liveness and say why testing finds one more easily.
4. Define linearizability. Give the two properties that make it the preferred condition.
5. Distinguish a race condition from a data race, and say why Python programmers are
   unprepared for the latter.
6. State the four Coffman conditions and the one you would normally break.
7. Name the four concurrency models and give the practical preference order.
8. State Amdahl's, Gustafson's, and the USL's claims, and what each explains that the others
   do not.

## 7. Primary sources

- Pike, "Concurrency Is Not Parallelism" (2012 talk).
- Herlihy & Wing, "Linearizability: A Correctness Condition for Concurrent Objects"
  (TOPLAS 1990).
- Hoare, "Communicating Sequential Processes" (CACM 1978).
- Lamport, "Proving the Correctness of Multiprocess Programs" (1977) — safety and liveness.
- Amdahl (1967); Gustafson (1988); Gunther, *Guerrilla Capacity Planning* (USL).
- Herlihy & Shavit, *The Art of Multiprocessor Programming*, chs. 1–3.

---

**Next:** [L02 — Threads, the GIL, and the Free-Threaded Build](L02-threads-and-the-gil.md)
