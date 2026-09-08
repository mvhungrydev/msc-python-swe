# PY-601 · Lesson 03 — Shared State, Locks, and the Memory Model

**Estimated study time:** 4 hours
**Prerequisites:** L01, L02; PY-501 L08

---

## 1. Orientation

```python
counter = 0
def worker():
    global counter
    for _ in range(1_000_000):
        counter += 1
```

Eight threads, eight million increments, and the result is not eight million. On a
GIL-enabled CPython it will be somewhere between one and eight million; on a free-threaded
build it will be much lower.

Disassemble the loop body (PY-501 L08):

```
LOAD_GLOBAL   counter
LOAD_CONST    1
BINARY_OP     +
STORE_GLOBAL  counter
```

Four instructions. A thread switch can occur between any two of them. Two threads both load
1000, both compute 1001, both store 1001 — one increment lost. **The GIL guarantees that one
bytecode instruction runs at a time; it guarantees nothing about sequences.**

This lesson is about what is atomic, what is visible, and what a lock actually gives you.

## 2. Theory

### 2.1 The three problems a memory model addresses

**Atomicity** — does an operation happen all at once, or can another thread observe an
intermediate state?

**Visibility** — when thread A writes, when (if ever) does thread B see it? On real hardware,
a write may sit in a store buffer or a private cache for an unbounded time unless
synchronization forces it out.

**Ordering** — do operations appear to other threads in the order the program wrote them?
Compilers reorder for optimization; CPUs reorder for pipelining and store buffering. Both are
permitted as long as *single-threaded* semantics are preserved — and both can break
multithreaded assumptions.

Languages designed for concurrency (Java, C++11, Go, Rust) publish a formal **memory model**
answering these. **CPython does not have one.** What it has instead is: the GIL, plus a set
of behaviours that are implementation detail, plus `_Py_atomic_*` primitives internally.

The practical consequence: **on CPython, do not reason from a memory model — reason from
"which sequences are single bytecodes", and prefer to not need to.** And when the
free-threaded build makes real parallelism possible, the safe assumption is that the
guarantees you had were accidental.

### 2.2 What is atomic in CPython (and why you should not rely on it)

Under the GIL, an operation implemented as a single bytecode that does not call back into
Python is effectively atomic:

- `x = y` (simple names), `list.append(x)`, `list.pop()`, `d[k] = v` for a plain `dict`
  with a hashable key whose `__hash__`/`__eq__` are C-implemented, `d.get(k)`,
  `set.add`, `deque.append`/`popleft`.
- `sorted(list)`, `list(d)` — these iterate, so they can be interrupted between elements;
  they will not crash, but they may see a mid-update state.

**Not atomic:** `x += 1`, `d[k] += 1`, `x = x + 1`, `if k not in d: d[k] = v`,
`list[i] = list[i] + 1`, anything with a Python-level `__hash__` or `__eq__` (which can
release the GIL by executing Python code), and every read-modify-write.

The "effectively atomic" list is:

- **Version-dependent.** The set of operations compiling to one bytecode changes.
- **Implementation-dependent.** PyPy makes no such promise; free-threaded CPython uses
  per-object locks that keep individual operations safe but do nothing for sequences.
- **Fragile under refactoring.** `d[k] = v` is atomic; extracting `k` into a computed value
  with a Python `__hash__` is not.

So: know it, to read existing code and to understand why bugs are rare rather than absent.
Do not write new code that depends on it. `queue.Queue` and explicit locks cost almost
nothing and are correct by construction.

### 2.3 Locks

```python
import threading
lock = threading.Lock()

with lock:                    # always use the context manager
    counter += 1
```

`threading.Lock` is a binary semaphore; `acquire()` blocks, `acquire(timeout=)` is available,
and `release()` from a thread that does not hold it raises. It is **not reentrant**: the same
thread acquiring twice deadlocks.

`threading.RLock` is reentrant — the owning thread may acquire repeatedly, and must release
the same number of times. Reach for it when a locked method calls another locked method on
the same object. But note: needing an `RLock` is often a design signal that your public
methods are calling each other, and the cleaner fix is a private unlocked `_do_x()` called
by both public methods under one acquisition.

**What a lock actually gives you.** Three things, and only the first is what people think
about:

1. **Mutual exclusion** — one thread in the critical section.
2. **A visibility boundary** — everything written before a release is visible to a thread
   that subsequently acquires. On CPython this is currently subsumed by the GIL, but it is
   the property you should reason with, because it is what remains true on free-threaded
   builds and other implementations.
3. **An ordering boundary** — operations are not reordered across acquire/release.

**What a lock does not give you:** protection of anything you forgot to lock. A lock protects
an *invariant*, and the invariant is a fact about your data that you must state. Write it
down, next to the lock:

```python
class Account:
    """`_lock` guards `_balance` and `_history`; their sum invariant
    (balance == sum of history amounts) holds outside the lock."""
```

Without that comment nobody — including you in six months — knows what the lock covers.

### 2.4 Deadlock and lock ordering

Two locks acquired in different orders by two threads is the classic deadlock. The practical
disciplines, in order of preference:

**1. Do not hold two locks.** Restructure so that only one is needed. Usually possible and
usually better.

**2. A global lock order.** Every thread acquires in the same, documented order. For dynamic
sets of locks (transferring between two accounts), order by a stable key — `id()` is
tempting but not stable across time; use an object's own identifier:

```python
def transfer(a: Account, b: Account, amount: Money) -> None:
    first, second = sorted((a, b), key=lambda x: x.id)
    with first._lock, second._lock:
        ...
```

**3. Timeouts.** `acquire(timeout=)` and back off on failure. Turns a deadlock into a
livelock risk and a detectable error; acceptable as a safety net, not as the primary
mechanism.

**4. Never call out while holding a lock.** No callbacks, no I/O, no user code. A lock held
across an unbounded wait is a latent outage, and a callback that acquires another lock is a
deadlock you cannot see locally. This is the rule most often broken.

`threading.Lock` in CPython has no deadlock detection. `faulthandler.dump_traceback_later()`
and `py-spy dump` are how you diagnose one in production (L10).

### 2.5 Condition variables

For "wait until a condition holds":

```python
class BoundedBuffer:
    def __init__(self, capacity: int) -> None:
        self._items: deque = deque()
        self._capacity = capacity
        self._cv = threading.Condition()

    def put(self, item) -> None:
        with self._cv:
            while len(self._items) >= self._capacity:    # WHILE, not if
                self._cv.wait()
            self._items.append(item)
            self._cv.notify()

    def get(self):
        with self._cv:
            while not self._items:
                self._cv.wait()
            item = self._items.popleft()
            self._cv.notify()
            return item
```

Three rules, each protecting against a specific bug:

- **Always hold the lock while checking the predicate and while waiting.** `wait()` atomically
  releases the lock and blocks, and reacquires before returning. Without atomicity you get
  the *lost wakeup*: check, someone signals, then you wait forever.
- **Always wait in a `while` loop, never an `if`.** Reasons: spurious wakeups are permitted;
  `notify_all` wakes several waiters and only one can proceed; and another thread may have
  consumed the condition between the notify and your reacquisition. This last is the real one
  and it is not exotic.
- **`notify_all` when waiters are waiting on *different* predicates**, `notify` when they are
  all equivalent. Using `notify` with mixed predicates wakes the wrong thread, which
  re-checks, fails, and waits again — while the thread that could have proceeded is never
  woken. This is a genuine deadlock and it is subtle.

In practice, use `queue.Queue` rather than hand-writing this. Write it once as an exercise so
you can read the standard library's version and recognize the same structure in other
languages.

### 2.6 The rest of the toolkit

| Primitive | Use |
|---|---|
| `Lock` | mutual exclusion |
| `RLock` | reentrant; a signal to reconsider the design |
| `Semaphore(n)` | limit concurrent access to n; the standard rate/pool limiter |
| `BoundedSemaphore` | as above, and raises on over-release — catches a real bug class |
| `Event` | one-shot broadcast signal (shutdown, ready) |
| `Condition` | wait for an arbitrary predicate |
| `Barrier(n)` | all n threads wait, then all proceed |
| `queue.Queue(maxsize)` | the workhorse; thread-safe, blocking, bounded |
| `contextvars` | per-task (not per-thread) context |

**`queue.Queue` deserves emphasis.** A bounded queue between stages gives you mutual
exclusion, condition-variable signalling, and *backpressure* (L09) in one object, with no
lock in your code. Most thread-safe design in Python should be "stages plus queues", and
explicit locks should be rare.

Notes: `Queue.join()`/`task_done()` implement "wait until all work is processed" and are
easy to misuse (a missing `task_done` hangs forever). `Queue.get(timeout=)` is how you make
a worker shutdown-responsive. A `None` sentinel per worker is the standard shutdown signal,
and you need one per worker because each consumes one.

### 2.7 The design hierarchy

Prefer, in order:

1. **No shared state.** Pass values, return values.
2. **Immutable shared state.** Frozen dataclasses, tuples, `frozenset`. Publish by rebinding
   a name — safe under the GIL and, for a single reference, safe in practice on
   free-threaded builds too (though say so explicitly).
3. **Confinement.** State owned by exactly one thread; others communicate by message. This
   is the actor model, and `queue.Queue` is how you build it.
4. **A concurrent collection.** `queue.Queue`, or a `dict` used only with atomic operations
   whose limits you have documented.
5. **Explicit locks with a written invariant.** Last resort, and keep the critical section
   as small as it can be.

Most concurrency bugs come from starting at 5.

## 3. Construction: a thread-safe cache, done three ways

Requirements: `get_or_compute(key, fn)` — return the cached value, or compute and store it.
Bounded size, LRU eviction, metrics, and *do not compute the same key twice concurrently*
(the "cache stampede" or "thundering herd" problem: on a cache miss for a hot key, a hundred
requests all compute the same expensive thing).

**Version 1 — one lock around everything.**

```python
def get_or_compute(self, key, fn):
    with self._lock:
        if key in self._cache:
            return self._cache[key]
        value = fn()                 # ← computing while holding the lock
        self._cache[key] = value
        return value
```

Correct, and it violates §2.4's rule 4: `fn()` is arbitrary code, possibly slow, possibly
I/O, possibly calling back into the cache (deadlock, since `Lock` is not reentrant).
Throughput is that of a single thread.

**Version 2 — release the lock during computation.**

```python
def get_or_compute(self, key, fn):
    with self._lock:
        if key in self._cache:
            return self._cache[key]
    value = fn()                     # unlocked
    with self._lock:
        return self._cache.setdefault(key, value)
```

Now concurrent, and it computes duplicates: N concurrent misses do N computations. The
`setdefault` at least ensures one value wins consistently. For a cheap `fn` this is the right
answer and you should say so.

**Version 3 — per-key futures.** One computation per key, everyone else waits:

```python
def get_or_compute(self, key, fn):
    with self._lock:
        if key in self._cache:
            self._hits += 1
            return self._cache[key]
        fut = self._inflight.get(key)
        if fut is None:
            fut = self._inflight[key] = Future()
            leader = True
        else:
            leader = False
    if not leader:
        return fut.result()          # wait for the leader, outside the lock
    try:
        value = fn()
    except BaseException as e:
        with self._lock:
            del self._inflight[key]
        fut.set_exception(e)
        raise
    with self._lock:
        self._cache[key] = value
        del self._inflight[key]
    fut.set_result(value)
    return value
```

Now write down what is *still* wrong, because there is a lot and finding it is the exercise:

- A follower that arrives, gets the future, and waits — while the leader fails — receives the
  leader's exception. Is that right? Probably, but it should be a documented decision, and it
  means one caller's transient failure is propagated to N callers.
- No timeout on `fut.result()`. If the leader hangs, all followers hang. §2.4 rule 4 again,
  one level removed.
- Eviction is not shown, and evicting a key with an in-flight future needs care.
- The leader might be cancelled or its thread killed, leaving the future unresolved forever.
- `self._hits += 1` inside the lock is fine; outside it would be the opening bug.

**Version 4 — reconsider.** Does this need to be lock-based at all? A single owning thread
with a request queue (design hierarchy level 3) has no locks, no stampede, and a natural
place for eviction — at the cost of serializing cache lookups through one thread. Measure
both. For a cache with cheap lookups and expensive computations, the queue version is often
faster *and* simpler, which is the lesson.

## 4. Failure modes

- **Relying on bytecode-level atomicity.** §2.2.
- **`+=` on shared state.** The canonical bug.
- **Check-then-act.** `if k not in d: d[k] = v`.
- **No written invariant for a lock.** Nobody knows what it protects.
- **Calling out while holding a lock.** Callbacks, I/O, user code.
- **Different lock orders in different code paths.**
- **`if` instead of `while` around `cv.wait()`.**
- **`notify` where `notify_all` is needed** (mixed predicates), producing a silent deadlock.
- **Unbounded queues.** No backpressure; memory grows until failure (L09).
- **`Queue.join()` without matching `task_done()`.**
- **Locks in `__del__` or finalizers.** They run at arbitrary points, including while the
  lock is held.
- **A lock acquired in a signal handler.** Signal handlers run on the main thread between
  bytecodes; acquiring a lock the main thread already holds deadlocks the process.
- **Copying a mutable object out of a lock and using it after.** The copy is a snapshot; if
  you needed consistency with later reads you needed the lock.

## 5. Exercises

### Warm-up (25 min)

**W1.** Demonstrate the lost-update bug and measure how many updates are lost at 2, 4, and 8
threads. Then fix it with a lock and measure the cost.

**W2.** Write a check-then-act race in a dict and demonstrate it. Fix it two ways
(`setdefault`, and a lock) and say when each is right.

**W3.** Build a deadlock with two locks. Then fix it with a global ordering, and show that a
timeout turns it into a detectable error instead.

### Core (2.5 h)

**C1 — The cache.** Complete §3, all four versions. Deliverable: the code; a stampede test
(50 threads, one cold key, an `fn` that counts invocations — assert exactly one); the
remaining-defects list from version 3 with a fix for each; throughput and p99 latency for all
four versions under a realistic hit ratio; and a recommendation.

**C2 — Write a `BoundedBuffer`.** From §2.5, by hand with a `Condition`. Then deliberately
break each of the three rules and demonstrate the specific failure each causes (lost wakeup,
spurious-wakeup corruption, notify-the-wrong-waiter deadlock). This exercise is what makes
condition variables stop being magic.

**C3 — Atomicity map.** Empirically determine which operations are atomic on your
interpreter, by running each in a tight loop across eight threads and checking for lost
updates. Cover: `list.append`, `dict[k] = v` with a C-hashed key and with a Python-hashed
key, `set.add`, `deque.append`, `x += 1`, `d[k] += 1`, `list[i] = list[i] + 1`. Report the
table, and if you have a free-threaded build, both columns.

**C4 — Eliminate the locks.** Take a lock-heavy component and rewrite it at level 2 or 3 of
§2.7 (immutability or confinement). Report: lines, number of shared mutables, throughput,
p99 latency, and the difficulty of arguing correctness in each version. The last is the
assessed part.

### Challenge

**X1.** Implement a reader-writer lock with a stated fairness policy (reader-preferring,
writer-preferring, or fair). Prove — by exhibiting traces — that your policy is what you
claim, and demonstrate the starvation your policy permits. Then measure it against a plain
`Lock` at 90/10, 50/50, and 10/90 read/write ratios and report at what ratio it starts
winning. (It is later than most people expect.)

**X2.** Read Herlihy & Shavit chs. 2–3 and implement Peterson's algorithm and a filter lock
in Python. Then explain precisely why they are *not* correct in general on modern hardware,
what memory barriers would be needed, and why the exercise is still worth doing.

## 6. Self-check

1. Give the four bytecodes of `counter += 1` and say where a switch can occur.
2. Name the three problems a memory model addresses, and state what CPython provides.
3. Give five operations that are atomic on CPython and five that are not, and say why you
   should not depend on the first list.
4. Name the three things a lock provides, and the one thing it does not.
5. Give four disciplines for avoiding deadlock, in preference order.
6. Give the three condition-variable rules and the bug each prevents.
7. State the design hierarchy for shared state and say where most bugs originate.
8. What is a cache stampede and what are two ways to prevent it?

## 7. Primary sources

- Herlihy & Shavit, *The Art of Multiprocessor Programming*, chs. 2–3, 7, 8.
- Arpaci-Dusseau, *OSTEP*, chs. 28–31 (locks, condition variables, semaphores). Free, and
  the clearest introduction that exists.
- Lamport, "How to Make a Multiprocessor Computer That Correctly Executes Multiprocess
  Programs" (IEEE ToC, 1979).
- CPython `Lib/threading.py` and `Lib/queue.py` — both readable.
- The Java Memory Model (JSR-133) FAQ — for what a formal memory model looks like, since
  Python has none.

---

**Previous:** [L02](L02-threads-and-the-gil.md) · **Next:**
[L04 — Thread-Safe Design: Confinement, Immutability, and Queues](L04-thread-safe-design.md)
