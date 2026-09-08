# PY-601 · Lesson 05 — The Event Loop from First Principles

**Estimated study time:** 4 hours
**Prerequisites:** PY-502 L05, L06; PY-601 L01

---

## 1. Orientation

`asyncio` is not magic and it is not concurrency-by-framework. It is:

> **a `while` loop that asks the operating system which file descriptors are ready, and then
> resumes the coroutines waiting on them.**

Everything else — tasks, futures, `await`, `gather`, timeouts — is bookkeeping around that
loop. This lesson builds one, because the difference between using `asyncio` and
understanding it is exactly the difference between debugging by search and debugging by
reasoning.

By the end you will have written a scheduler that runs coroutines, performs real network
I/O, supports sleeping and cancellation, and is perhaps 150 lines. Then `asyncio`'s
behaviour — including its surprising parts — will follow from what you built.

## 2. Theory

### 2.1 The kernel's side: readiness notification

A blocking `recv()` puts the thread to sleep until data arrives. To handle many connections
in one thread you need to ask the kernel: *of these N file descriptors, which are ready?*

| Mechanism | Complexity | Notes |
|---|---|---|
| `select()` | O(n) per call, FD_SETSIZE limit (~1024) | portable, ancient |
| `poll()` | O(n) per call, no limit | POSIX |
| `epoll` (Linux) | O(1) amortized, edge- or level-triggered | the workhorse |
| `kqueue` (BSD/macOS) | O(1) | also handles timers, signals, files |
| IOCP (Windows) | completion-based, not readiness-based | different model entirely |
| `io_uring` (Linux 5.1+) | completion-based, batched submission | the future; not yet in `asyncio` |

Python's `selectors` module abstracts these:

```python
import selectors
sel = selectors.DefaultSelector()          # epoll on Linux, kqueue on macOS
sel.register(sock, selectors.EVENT_READ, data=callback)
for key, events in sel.select(timeout=0.05):
    key.data()                              # invoke the callback
```

**Readiness versus completion** is a real architectural distinction. epoll/kqueue tell you
"you may now read without blocking"; you then read. IOCP and `io_uring` tell you "the read
you asked for has completed, here is the data". `asyncio` presents a readiness-style API and
has a separate proactor event loop on Windows to bridge the difference — which is why some
`asyncio` behaviours differ on Windows.

### 2.2 Non-blocking sockets

```python
sock.setblocking(False)
try:
    data = sock.recv(4096)
except BlockingIOError:              # EWOULDBLOCK — nothing available right now
    ...
```

The whole event-driven model is: set everything non-blocking, ask the selector what is
ready, act only on those. A single blocking call anywhere in the loop stalls *every*
connection — which is the origin of async's cardinal sin (§2.6).

### 2.3 The scheduler

A coroutine suspends at `await` and resumes when driven. The loop's job is to know *why*
each suspended coroutine is waiting and to resume it when that reason is resolved.

Minimal design:

```
ready:    deque of (coroutine, value_to_send)  — resume these now
sleeping: heap of (deadline, coroutine)        — resume when the clock passes deadline
waiting:  {fd: coroutine}                      — resume when the selector says ready
```

The loop:

```python
while ready or sleeping or waiting:
    # 1. how long may we block in select()?
    timeout = 0 if ready else (sleeping[0].deadline - now() if sleeping else None)

    # 2. ask the kernel
    for key, _ in sel.select(timeout):
        ready.append((waiting.pop(key.fd), None))

    # 3. move expired timers to ready
    while sleeping and sleeping[0].deadline <= now():
        ready.append((heappop(sleeping).coro, None))

    # 4. run every currently-ready coroutine until it suspends again
    for _ in range(len(ready)):
        coro, value = ready.popleft()
        try:
            request = coro.send(value)      # ← runs the coroutine until its next await
        except StopIteration as e:
            finish(coro, e.value)
            continue
        dispatch(request, coro)             # register in sleeping or waiting
```

That is the entire idea. Note step 4's `coro.send(value)` — this is PY-502 L06's generator
protocol, unchanged. The loop is a driver; `await` is a yield; the "request" a coroutine
yields is a description of what it is waiting for.

### 2.4 Awaitables: what `await x` actually does

`await x` requires `x` to be an **awaitable**: it has `__await__` returning an iterator.
`await x` is close to `yield from x.__await__()` (PY-502 L06 §2.4), so the values that
iterator yields go *out to the loop*, and what the loop sends back becomes the result.

Three kinds:

- **Coroutines.** `await other()` delegates; the loop never sees it as a separate entity.
- **Futures.** An object with a result that will be set later. `__await__` yields itself and
  the loop resumes the awaiting coroutine when the result is set. This is the primitive.
- **Objects with a custom `__await__`.** How you write your own primitives.

The bottom of every await chain is a bare `yield` reaching the loop. In your toy loop, define
a small protocol:

```python
class Wait(NamedTuple):     # what a coroutine yields to the loop
    kind: str               # "read" | "write" | "sleep"
    arg: object

@types.coroutine            # makes a generator awaitable
def _wait(kind, arg):
    return (yield Wait(kind, arg))

async def sleep(seconds: float) -> None:
    await _wait("sleep", time.monotonic() + seconds)

async def recv(sock, n: int) -> bytes:
    await _wait("read", sock)
    return sock.recv(n)
```

`@types.coroutine` marks a generator as a coroutine so `await` accepts it. This is the seam
where "generator that yields to a driver" and "coroutine" meet, and seeing it removes the
last of the mystery.

### 2.5 Tasks

A **coroutine** is inert. Calling `f()` on an `async def` produces an object and runs
nothing. A **task** is a coroutine that the loop has taken responsibility for driving.

```python
def spawn(coro) -> Task:
    task = Task(coro)
    ready.append((coro, None))
    return task
```

This distinction explains three of the most common `asyncio` mistakes:

- **`f()` without `await` or `create_task`** does nothing, and emits
  "coroutine was never awaited" *if* the object is collected — sometimes long after.
- **`await f()`** runs it to completion before continuing: sequential, not concurrent. Two
  awaited calls take the sum of their times.
- **`create_task(f())`** schedules it: now it is concurrent with the caller, and you must
  eventually await it or its exception is lost (§2.7).

Concurrency comes from *tasks*, not from `await`.

### 2.6 The cardinal sin

**A blocking call inside a coroutine stalls the entire loop**, and therefore every other
connection, timer, and task in the process.

```python
async def handler(request):
    data = requests.get(url)          # blocking HTTP — stalls everything
    rows = cursor.execute(sql)        # blocking DB driver — stalls everything
    time.sleep(1)                     # stalls everything
    hashlib.pbkdf2_hmac(...)          # CPU-bound: stalls everything (correctly, but stalls)
```

The symptom in production is not an error. It is that p99 latency for *unrelated* endpoints
degrades, which is one of the harder things to diagnose from the outside.

The fixes:

- Use an async library (`httpx`, `asyncpg`, `aiofiles`).
- `await loop.run_in_executor(pool, blocking_fn)` or `await asyncio.to_thread(fn)` for
  blocking I/O.
- A process pool for CPU-bound work (L08).
- For unavoidable CPU work in the loop, `await asyncio.sleep(0)` periodically to yield —
  crude, and better than starving everything.

**Detect it, do not rely on discipline.** `loop.set_debug(True)` logs callbacks slower than
`loop.slow_callback_duration` (default 100 ms). Set that threshold to something meaningful
for your service (10 ms is often right) and log it in production. This is the highest-value
single line of configuration in an async service.

### 2.7 Error handling in a task-based world

When a task raises, nobody is on the stack to catch it. `asyncio` stores the exception in
the task, and:

- If you `await` the task, it is raised there.
- If you never await it and the task object is collected, `asyncio` logs
  "Task exception was never retrieved" — from the destructor, at an arbitrary later time,
  possibly during shutdown, possibly not at all.

This is why **fire-and-forget tasks are dangerous**, and why two further rules exist:

- **Keep a reference to every task you create.** `asyncio.create_task` holds only a *weak*
  reference from the loop, so a task whose only reference you dropped can be garbage
  collected mid-execution. The standard fix is a module-level `set` that tasks are added to
  and remove themselves from on completion.
- **Prefer `TaskGroup`** (L06), which awaits everything it created and propagates failures.

## 3. Construction: build the loop

This is the central exercise of the course. Build it incrementally; each stage should run.

**Stage 1 — cooperative scheduling only.** A `ready` deque, `run_until_complete`, and a
`sleep(0)`-style yield point. Run three coroutines that print and yield in a loop; observe
interleaving. ~30 lines.

**Stage 2 — timers.** A heap of deadlines, a real `sleep(seconds)`, and a `select`-free loop
that blocks with `time.sleep` until the next deadline. Verify that ten concurrent
`sleep(1)`s take one second, not ten.

**Stage 3 — I/O.** `selectors`, non-blocking sockets, `recv`/`send`/`accept`/`connect`
primitives, and an echo server. Test it with 100 concurrent clients. This is the stage where
it becomes a real event loop.

**Stage 4 — tasks and futures.** `spawn(coro) -> Task`, a `Future` with `set_result`, and
`await task`. Then `gather(*tasks)`. Now you can write concurrent programs on your loop.

**Stage 5 — cancellation.** `task.cancel()` throws an exception into the coroutine at its
suspension point (PY-502 L06 §2.4 — this is `gen.throw`). Implement it, and then discover
the problems: a task cancelled while it holds a resource; a task that catches the
cancellation and continues; a cancellation delivered while the task is already finishing.
L07 is about those problems; meeting them here first is the point.

**Stage 6 — the diagnostics.** A `slow_callback` warning: time each `coro.send()` and log if
it exceeds a threshold. Then deliberately put a `time.sleep(0.5)` in one handler and watch
every other connection stall. Measure the effect on the others' latency.

**Stage 7 — compare.** Run the same echo-server benchmark on your loop and on `asyncio`.
You will be slower — by how much, and where does the difference come from? Profile both
(PY-602 L02). The answer is usually: `asyncio`'s C-accelerated `Future`, its transport
buffering, and `uvloop` if you install it.

## 4. Failure modes

- **A blocking call in a coroutine.** §2.6. The single most common async bug in production.
- **`await` where `create_task` was meant.** Sequential execution that looks concurrent.
- **A coroutine never awaited.** Silently does nothing.
- **Fire-and-forget tasks.** Exceptions lost; the task may be collected mid-flight.
- **No reference held to a created task.**
- **CPU-bound work in the loop.** Correct results, catastrophic latency for everything else.
- **Mixing sync and async libraries.** A "small" blocking DB call in an async handler.
- **`asyncio.sleep(0)` as a general concurrency fix.** It yields once; it does not make CPU
  work concurrent.
- **Assuming `await` yields.** `await` on an already-complete future may not yield to the
  loop at all, so a loop of awaits can starve other tasks.
- **Debug mode off in production**, so slow callbacks are invisible.

## 5. Exercises

### Warm-up (30 min)

**W1.** Write two `async def` functions that each `asyncio.sleep(1)`. Time: awaiting them
sequentially, `gather`ing them, and creating tasks then awaiting. Explain all three numbers.

**W2.** Call a coroutine function without awaiting. Show the warning, and show a case where
the warning does not appear.

**W3.** Put `time.sleep(1)` in one handler of a small `asyncio` server and measure the
latency effect on concurrent requests to a different endpoint.

### Core (3 h)

**C1 — Build the loop.** Complete §3, stages 1–6. Deliverable: the loop (~150–250 lines), an
echo server, a test with 100 concurrent clients, the cancellation implementation with the
problems it revealed documented, and the slow-callback demonstration with measured latency
impact.

**C2 — Benchmark against `asyncio`.** Stage 7. Report throughput and p99 latency for your
loop, `asyncio`, and `asyncio` with `uvloop`, at 10/100/1000 concurrent connections. Profile
the gap and attribute it. Report machine and versions.

**C3 — Find the blocking calls.** Write a detector: monkey-patch or use `sys.settrace`/
`sys.monitoring` (PEP 669) to time every coroutine step in a real async application and
report the slowest. Run it on a real service. Report what you found — in most async
codebases, something.

**C4 — Readiness vs completion.** Read about `io_uring` and IOCP. Write 600 words explaining
how a completion-based loop differs structurally from your readiness-based one, what changes
in the scheduler, and why `asyncio` needs a separate proactor loop on Windows.

### Challenge

**X1.** Extend your loop with: a thread-pool executor bridge (`run_in_executor`), signal
handling that integrates with the loop, and a `TaskGroup` with structured-concurrency
semantics (all children awaited; a failing child cancels its siblings; errors aggregated
into an `ExceptionGroup`). Then write the test that a `TaskGroup` leaves no task running
after any of its exit paths.

**X2.** Read `asyncio`'s `base_events.py` and `selector_events.py`. Produce an annotated
account of `BaseEventLoop._run_once`, mapping every part to your implementation. Identify
three things `asyncio` does that yours does not and explain why each is necessary.

## 6. Self-check

1. Describe an event loop in one sentence.
2. Distinguish readiness-based from completion-based I/O notification and name an example of
   each.
3. Give the three data structures a scheduler needs and what each is for.
4. What does `await x` require of `x`, and what is at the bottom of every await chain?
5. Distinguish a coroutine from a task, and say where concurrency comes from.
6. Give four categories of blocking call and the fix for each.
7. Why is a fire-and-forget task dangerous, in two distinct ways?
8. Why must you keep a reference to a task you created?

## 7. Primary sources

- PEP 3156 (`asyncio`), PEP 492 (`async`/`await`), PEP 525 (async generators),
  PEP 530 (async comprehensions).
- Beazley, "Build Your Own Async" (2019) and "Concurrency From the Ground Up" (2015). The
  first is essentially this lesson, done live.
- `asyncio` source: `base_events.py`, `selector_events.py`, `tasks.py`, `futures.py`.
- Kerrisk, *The Linux Programming Interface*, chs. 63 (alternative I/O models) and 63.4
  (epoll).

---

**Previous:** [L04](L04-thread-safe-design.md) · **Next:**
[L06 — `asyncio` in Practice and Structured Concurrency](L06-asyncio-and-structured-concurrency.md)
