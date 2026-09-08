# PY-601 · Lesson 04 — Thread-Safe Design: Confinement, Immutability, and Queues

**Estimated study time:** 3.5 hours
**Prerequisites:** L03

---

## 1. Orientation

L03 taught locks. This lesson is about not needing them.

The observation underneath: **every concurrency bug requires state that is both shared and
mutable.** Remove either property and the bug class disappears — not "becomes less likely",
disappears. So the design question is not "where do I put the locks?" but "how do I arrange
for less state to be both shared and mutable?"

That reframing is worth more than any amount of skill with synchronization primitives, and
it is what separates concurrent code you can reason about from concurrent code you can only
test.

## 2. Theory

### 2.1 Thread safety is a specification, not a property

"Is this class thread-safe?" is under-specified. The real question has levels
(Goetz's taxonomy, from *Java Concurrency in Practice*, and it transfers exactly):

| Level | Meaning |
|---|---|
| **Immutable** | State never changes after construction. Safe, always, with no synchronization. |
| **Thread-safe** | Behaves correctly under concurrent access with no external synchronization. |
| **Conditionally thread-safe** | Individual operations are safe; *sequences* need external synchronization. |
| **Thread-compatible** | Not safe itself, but safe if the caller synchronizes. |
| **Thread-hostile** | Unsafe even with external synchronization (it mutates global state). |

`queue.Queue` is thread-safe. A `dict` is conditionally thread-safe: `d[k] = v` is fine,
`if k not in d: d[k] = v` is not. A class that mutates a module-level global without
synchronization is thread-hostile, and no caller can fix it.

**Document the level in the class docstring.** "Thread-safe" without qualification is the
statement that causes the composition bug of L01 §2.2, because a caller reads it as
"anything I do with this is safe."

### 2.2 Immutability

The strongest tool. An object whose state cannot change after construction is safe to share
with any number of threads, forever, with no synchronization.

In Python:

```python
@dataclass(frozen=True, slots=True)
class Config:
    timeout: float
    retries: int
    endpoints: tuple[str, ...]      # tuple, not list
```

Points that matter:

- **Frozen is shallow.** `frozen=True` prevents rebinding attributes; it does not prevent
  mutating a `list` inside. Use tuples, `frozenset`, and nested frozen dataclasses
  throughout. A frozen dataclass containing a list is a mutable object with a misleading
  label.
- **Publish by rebinding a name.** To "update" immutable shared state, build a new object
  and rebind a single reference. Readers see the old or the new object, never a mix —
  because the rebinding is one bytecode:

```python
_config: Config = load()            # module-level

def reload() -> None:
    global _config
    _config = load()                # atomic rebinding; readers see old or new

def get_config() -> Config:
    return _config                  # a consistent snapshot, always
```

This is **copy-on-write publication**, and it is the right pattern for configuration,
routing tables, feature flags, and any read-mostly shared structure. Cost: a full copy per
update, which is irrelevant when updates are rare.

- **Persistent (immutable) data structures** make the copy cheap by sharing structure.
  `pyrsistent` and `immutables` (the latter is what `contextvars` uses internally) give you
  O(log n) updates producing a new version while sharing most nodes. Worth knowing about
  when the structure is large and updates are frequent.

### 2.3 Confinement

State owned by exactly one thread. Nothing shared, so nothing to synchronize.

**Thread confinement.** A GUI's widget tree belongs to the UI thread; a database connection
belongs to the thread that opened it. Other threads request work by message.

**Stack confinement.** Local variables are per-frame and therefore per-thread by
construction. A local `list` accumulated in a loop is safe by definition — which is why
"build a local result, then hand it over" is a good pattern.

**Ad-hoc confinement.** By convention only, enforced by nothing. This is the fragile kind;
if you rely on it, at least document it and consider an assertion:

```python
def _assert_owner(self) -> None:
    assert threading.get_ident() == self._owner, "called from the wrong thread"
```

That assertion costs nothing and turns a heisenbug into an immediate, localized failure.
(Remember it disappears under `-O`, so use a real check if it matters.)

**Ownership transfer.** The most useful pattern: an object is confined to thread A, then
*handed off* through a queue to thread B, and A never touches it again. Safe if the handoff
is disciplined; the queue provides the synchronization. Rust encodes this in its type system;
in Python it is a convention, and setting the local reference to `None` after `put()` is a
cheap way to make violations noisy.

### 2.4 The pipeline: stages and bounded queues

The default architecture for concurrent Python:

```
producer ──▶ Queue(maxsize=100) ──▶ worker×N ──▶ Queue(maxsize=100) ──▶ writer
```

Properties, all of which you get for free:

- **No shared mutable state.** Each stage owns its own; items are transferred.
- **Backpressure.** Bounded queues block producers when consumers fall behind, so memory is
  bounded and the slowest stage sets the rate (L09).
- **Independent tuning.** Each stage's parallelism is a separate number.
- **Testable in isolation.** A stage is a function from an input iterable to an output
  iterable — exactly PY-502 L05's pipeline stages, now with concurrency between them.
- **The failure modes are explicit.** What happens when a queue is full, when a worker
  raises, when the consumer dies — each is a decision you make rather than a behaviour you
  discover.

The template worth internalizing:

```python
def run_pipeline(items: Iterable[Item], n_workers: int = 8) -> Iterator[Result]:
    inq: queue.Queue[Item | None] = queue.Queue(maxsize=100)
    outq: queue.Queue[Result | Failure | None] = queue.Queue(maxsize=100)

    def worker() -> None:
        while (item := inq.get()) is not None:
            try:
                outq.put(process(item))
            except Exception as e:
                outq.put(Failure(item, e))
        outq.put(None)                        # one sentinel per worker

    threads = [threading.Thread(target=worker, daemon=False) for _ in range(n_workers)]
    for t in threads: t.start()

    def feed() -> None:
        for item in items:
            inq.put(item)
        for _ in threads:
            inq.put(None)                     # one sentinel per worker
    feeder = threading.Thread(target=feed); feeder.start()

    finished = 0
    while finished < n_workers:
        out = outq.get()
        if out is None:
            finished += 1
        else:
            yield out
    feeder.join()
    for t in threads: t.join()
```

Things this template gets right that hand-written versions usually get wrong:

- **One sentinel per worker**, on both queues. A single `None` is consumed by one worker and
  the others hang.
- **Bounded queues on both sides.** An unbounded output queue means a slow consumer causes
  unbounded memory growth (L09 §2.1).
- **Failures travel as values**, not as exceptions that kill a worker silently.
- **Non-daemon threads with explicit joins**, so shutdown is orderly.
- **The generator yields**, so the caller sets the consumption rate.

What it still does not handle, and which you must add: early termination by the consumer
(the generator is abandoned — L05's cleanup problem, now with threads blocked on `put`);
exceptions in the feeder; and a shutdown timeout. Those are C1.

### 2.5 Idempotence and commutativity

Two properties that make concurrency easier and are worth designing for deliberately:

**Idempotence** — applying an operation twice has the same effect as once. Then at-least-once
delivery, retries, and duplicate processing are all safe. `set_status(PAID)` is idempotent;
`increment_balance(10)` is not. Making an operation idempotent — usually by keying it with a
request id and recording what has been applied — converts a hard distributed problem into an
easy one (L07 §2.5 of SE-521; DS-701 L08).

**Commutativity** — order does not matter. Then you need no ordering guarantee. Adding to a
set commutes; appending to a list does not. CRDTs (DS-701 L07) are built entirely on
finding commutative formulations of update operations.

When you can design an operation to be idempotent and commutative, most of the concurrency
problem evaporates. This is a *modelling* choice made early, not something you retrofit, and
noticing the opportunity is a senior-level skill.

### 2.6 Lock-free and wait-free, briefly

Formal progress guarantees, worth knowing as vocabulary:

- **Blocking** — a thread can be delayed indefinitely by another (any lock).
- **Obstruction-free** — a thread makes progress if it runs alone for long enough.
- **Lock-free** — *some* thread always makes progress. The system as a whole cannot stall,
  though an individual thread may starve.
- **Wait-free** — *every* thread makes progress in a bounded number of steps. The strongest
  and the rarest.

Built on atomic primitives, above all **compare-and-swap (CAS)**: atomically, "if this
location holds X, set it to Y". The canonical pattern is a retry loop: read, compute a new
value, CAS, retry if it failed.

The **ABA problem** is the standard trap: a value changes from A to B and back to A between
your read and your CAS, so the CAS succeeds while the world has changed underneath. Solved
with version tags or hazard pointers.

In Python you cannot write lock-free algorithms directly — there is no CAS on Python objects.
You meet these concepts when: reading C/Rust extension source, using `multiprocessing.Value`
with hardware atomics, using a database's optimistic concurrency (which is CAS on a row —
SE-521 L07 §2.4), or reasoning about the free-threaded interpreter's internals. Know the
vocabulary; do not try to implement it in Python.

## 3. Construction: a metrics collector

A component every service has, and which is a good exercise because the obvious
implementation is wrong.

**Requirements.** Many threads increment counters and record histogram observations. One
thread periodically reads and exports a consistent snapshot. High write throughput, low read
frequency.

**Version 1 — a lock per operation.**

```python
class Metrics:
    def __init__(self): self._lock = threading.Lock(); self._counters = defaultdict(int)
    def incr(self, name, n=1):
        with self._lock: self._counters[name] += n
    def snapshot(self): 
        with self._lock: return dict(self._counters)
```

Correct. And every increment on every thread contends on one lock — the classic coherency
bottleneck (L01 §2.7). Measure it at 1, 2, 4, 8, 16 threads and watch throughput turn down.

**Version 2 — per-thread accumulation, aggregate on read.** Confinement:

```python
class Metrics:
    def __init__(self) -> None:
        self._local = threading.local()
        self._all: list[dict[str, int]] = []      # registry of per-thread dicts
        self._lock = threading.Lock()             # only for registration and snapshot

    def _mine(self) -> dict[str, int]:
        d = getattr(self._local, "d", None)
        if d is None:
            d = self._local.d = defaultdict(int)
            with self._lock:
                self._all.append(d)
        return d

    def incr(self, name: str, n: int = 1) -> None:
        self._mine()[name] += n                   # NO LOCK — thread-confined

    def snapshot(self) -> dict[str, int]:
        with self._lock:
            per_thread = list(self._all)
        out: dict[str, int] = defaultdict(int)
        for d in per_thread:
            for k, v in list(d.items()):          # read while another thread writes
                out[k] += v
        return dict(out)
```

Writes are uncontended. Now be precise about what you gave up:

- The snapshot is **not a consistent point-in-time cut**. It sums per-thread dicts read at
  slightly different moments, so a counter may reflect an increment that happened after
  another counter was read. For monotonic counters exported to a monitoring system this is
  fine and is exactly what real metrics libraries do — but *say so*, because someone will
  eventually compute a ratio from two counters and be confused.
- Reading `d.items()` while another thread writes is safe on CPython (the dict will not
  corrupt) but can raise `RuntimeError: dictionary changed size during iteration`. The
  `list(d.items())` is doing real work here — and on a free-threaded build you should
  reconsider it entirely.
- Dead threads' dicts stay in `_all` forever: a leak proportional to threads created. Fix
  with weak references, or with a pool of long-lived threads.

**Version 3 — the owning-thread variant.** Increments go into a lock-free-ish per-thread
buffer flushed periodically to a single aggregator thread via a queue. Trades a small
latency for a clean consistency story and no leak. Implement and measure.

**Version 4 — measure all three.** Throughput of `incr` at 1–32 threads; snapshot cost;
memory; and the *staleness* of the snapshot. Plot. Then decide, and write down which
property you are prioritizing.

This progression — lock, then confinement, then confinement with a defined consistency story
— is the general shape of thread-safe design, and doing it once on a real component teaches
more than any amount of reading.

## 4. Failure modes

- **"Thread-safe" without a level.** §2.1.
- **`frozen=True` with a mutable field inside.** A misleading label.
- **`threading.local` in a thread pool.** Per-worker, not per-task; leaks across tasks. Use
  `contextvars`.
- **Unbounded queues.** Memory grows until failure.
- **One sentinel for N workers.** N−1 workers hang.
- **Losing exceptions in workers.** They die silently and the pipeline stalls.
- **Daemon threads for pipeline workers.** Killed mid-item at exit.
- **A "snapshot" that is not consistent, presented as if it were.**
- **Per-thread state that leaks with thread lifetime.**
- **Retrofitting idempotence.** It is a modelling decision, cheap early and expensive late.
- **Trying to write lock-free code in Python.**

## 5. Exercises

### Warm-up (25 min)

**W1.** Write a `frozen=True` dataclass with a `list` field and mutate it. Then make it
genuinely immutable and show the difference.

**W2.** Build a pipeline with a single `None` sentinel and four workers. Demonstrate the
hang. Fix it.

**W3.** Classify five classes from the standard library or a library you use by §2.1's five
levels, with justification.

### Core (2.5 h)

**C1 — The complete pipeline.** Take §2.4's template and make it production-grade: consumer
early termination (with all producer threads unblocked), feeder exceptions, a bounded
shutdown timeout, and per-stage metrics. Write the tests for each failure path — including
the one where the consumer abandons the generator while all queues are full, which is the
hard one.

**C2 — The metrics collector.** Complete §3, all four versions, with the measurements and
plots. Deliverable includes a 500-word note stating precisely what consistency your chosen
version provides and one way a caller could misuse it.

**C3 — Copy-on-write publication.** Implement a routing table updated rarely and read
constantly, using immutable publication. Measure read throughput against a `RLock`-guarded
mutable version at 1–32 reader threads, with an update every second. Report the curves.
Then state what the immutable version cannot do (hint: multi-key consistent update *and*
read-your-writes for the updater).

**C4 — Make an operation idempotent.** Take a non-idempotent operation in a real system.
Redesign it to be idempotent (request key + applied-record, or a naturally idempotent
formulation). Report: schema changes, the window where duplicates are still possible, and
what you now no longer have to guarantee elsewhere.

### Challenge

**X1.** Implement a Michael–Scott lock-free queue in Rust or C, expose it to Python via
PyO3/Cython, and benchmark it against `queue.Queue` and `collections.deque` under
contention. Report the crossover. Then explain the ABA problem in your implementation and
how you avoided it. (After PY-602 L07.)

**X2.** Take a real service and audit every piece of shared mutable state: what it is, who
writes it, what protects it, and what invariant it has. Produce the table. For each entry,
propose a move up the §2.7 hierarchy and estimate the cost. Report how many could be
eliminated entirely.

## 6. Self-check

1. Give the five levels of thread-safety specification with an example of each.
2. Why is `frozen=True` insufficient for immutability, and what do you do instead?
3. Explain copy-on-write publication and when it is the right pattern.
4. Name four kinds of confinement and say which is fragile.
5. Give five properties a bounded-queue pipeline provides for free.
6. Why one sentinel per worker?
7. Define idempotence and commutativity, and say what each buys.
8. Distinguish lock-free from wait-free, and say why you cannot write either in pure Python.

## 7. Primary sources

- Goetz et al., *Java Concurrency in Practice*, chs. 3–5. The thread-safety taxonomy and the
  confinement/publication discussion transfer directly.
- Herlihy & Shavit, chs. 3 (progress conditions), 10 (concurrent queues).
- Hoare, "Communicating Sequential Processes" (1978).
- Michael & Scott, "Simple, Fast, and Practical Non-Blocking and Blocking Concurrent Queue
  Algorithms" (PODC 1996).
- CPython `Lib/queue.py`.

---

**Previous:** [L03](L03-shared-state-and-locks.md) · **Next:**
[L05 — The Event Loop from First Principles](L05-event-loop-from-first-principles.md)
