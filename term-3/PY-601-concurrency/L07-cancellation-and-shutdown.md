# PY-601 · Lesson 07 — Cancellation, Timeouts, and Shutdown

**Estimated study time:** 4 hours
**Prerequisites:** L05, L06; PY-501 L10; PY-502 L07

---

## 1. Orientation

Cancellation is the hardest correctness problem in async programming, and it is hard for a
precise reason:

> **Cancellation is an exception raised at a point you did not choose.**

Every `await` in your code is a place where `CancelledError` may appear. Your function must
be correct if it is interrupted at *any* of them — with a half-written database row, a
half-sent HTTP request, a lock held, a file open, or a compensating action pending.

This is the same shape as exception safety in C++ or Rust's `Drop` guarantees, and Python
gives you fewer tools. So the discipline has to be yours.

## 2. Theory

### 2.1 The mechanism

`task.cancel()` does not stop the task. It:

1. Marks the task as cancel-requested.
2. Arranges for `CancelledError` to be **thrown into** the coroutine at its current
   suspension point — which is `gen.throw()` (PY-502 L06 §2.4), unchanged.
3. Returns immediately. The task is *not* cancelled yet; it is *asked*.

Consequences that must be internalized:

- **Cancellation is cooperative.** A coroutine that never `await`s cannot be cancelled. A
  CPU-bound loop in a coroutine is uninterruptible.
- **`await task` after `cancel()` is required** to know that cancellation completed.
  `task.cancel()` followed immediately by process exit tells you nothing.
- **A task may refuse.** Catching `CancelledError` and continuing is legal.
- **The exception can arrive in a `finally` block**, if that block awaits.

### 2.2 `CancelledError` is a `BaseException`

Since 3.8, `asyncio.CancelledError` inherits from `BaseException`, not `Exception`. This is
deliberate: `except Exception:` — which appears in every retry loop, every generic error
handler, every "log and continue" — must **not** swallow cancellation.

The rules that follow:

```python
try:
    await work()
except asyncio.CancelledError:
    await cleanup()          # allowed
    raise                    # MANDATORY
except Exception:
    log.exception("failed")  # safe: cancellation does not land here
```

```python
try:
    await work()
except BaseException:        # only if you re-raise
    await cleanup()
    raise
```

**Never write `except BaseException: pass`.** And be suspicious of any library that does; it
will make your service unshutdownable.

Note the asymmetry with threads: a thread cannot be cancelled at all in Python. There is no
`Thread.cancel()`. This is one of the strongest arguments for async over threads for
long-running I/O work — cancellation is *possible*.

### 2.3 Timeouts

```python
async with asyncio.timeout(2.0):        # 3.11+, preferred
    await step_one()
    await step_two()
```

`asyncio.timeout` cancels *the enclosing block* when the deadline passes, then converts the
`CancelledError` into `TimeoutError` as it leaves the block. Its advantages over `wait_for`:

- It applies to a **block**, so a multi-step operation gets one budget.
- It **composes**: nested timeouts work, and the inner one firing does not look like the
  outer one firing (this is the "cancel scope identity" problem, and `asyncio.timeout`
  handles it with `Task.uncancel()`).
- `asyncio.timeout_at(deadline)` takes an absolute deadline, which is what you want for
  propagating a budget through a call chain.

**Total budgets versus per-call timeouts.** A three-service chain with a 2 s timeout on each
call has a 6 s worst case; add two retries each and it is 18 s. The caller's client gave up
at 3 s and retried, so you are now doing the work three times over. This is how a slow
dependency becomes an outage.

The fix is a **deadline propagated as a value**:

```python
_deadline: ContextVar[float | None] = ContextVar("deadline", default=None)

def remaining() -> float:
    d = _deadline.get()
    if d is None:
        return DEFAULT_BUDGET
    left = d - time.monotonic()
    if left <= 0:
        raise TimeoutError("budget exhausted")
    return left

async def call_upstream(url: str) -> Response:
    async with asyncio.timeout(remaining() * 0.9):   # leave room for our own work
        return await client.get(url)
```

Every hop consumes from one budget. gRPC and modern HTTP frameworks do this natively; if
yours does not, do it yourself. It is perhaps forty lines and it eliminates a whole class of
cascading failure.

### 2.4 Writing cancellation-safe code

The core question for every `await`: **if `CancelledError` is raised here, what is left in a
bad state?**

**Pattern 1 — cleanup in `finally`.**

```python
conn = await pool.acquire()
try:
    await conn.execute(sql)
finally:
    await pool.release(conn)      # runs on cancellation too
```

Prefer `async with` — it is the same thing with the cleanup impossible to forget.

**Pattern 2 — do not leave partial state.** If cancellation between two writes leaves an
inconsistent record, the two writes belong in one transaction, or the operation needs to be
idempotent and retriable so that a partial application is recoverable.

**Pattern 3 — cancellation-safe primitives.** Some operations cannot be interrupted safely:

```python
async def transfer_funds(a, b, amount):
    async with asyncio.timeout(5):
        await debit(a, amount)
        await credit(b, amount)      # cancelled here → money vanishes
```

There is no amount of `finally` that fixes this. It needs either a transaction spanning both
or a saga with compensation (SE-521 L07 §2.6). **Cancellation makes the atomicity question
unavoidable**, which is one of its underappreciated virtues.

**Pattern 4 — shielding.** `asyncio.shield(aw)` protects an awaitable from cancellation
arriving from *outside*:

```python
try:
    await asyncio.shield(commit_transaction())
except asyncio.CancelledError:
    # the shield re-raises for us, but commit_transaction() keeps running
    raise
```

Two warnings. First, `shield` does not stop the *caller* from being cancelled; it stops the
cancellation from propagating *into* the shielded awaitable. The caller resumes with a
`CancelledError` while the shielded work continues in the background — so you must still
track it. Second, **an unbounded shield is a hang**: at shutdown, a shielded operation with
no timeout blocks forever. Always shield *with* a timeout:

```python
async with asyncio.timeout(GRACE):
    await asyncio.shield(critical_cleanup())
```

**Pattern 5 — the two-phase cleanup.** In `__aexit__`, cleanup that awaits may itself be
cancelled. The standard shape:

```python
async def __aexit__(self, *exc) -> None:
    try:
        async with asyncio.timeout(self.close_timeout):
            await asyncio.shield(self._graceful_close())
    except (TimeoutError, asyncio.CancelledError):
        self._force_close()          # synchronous, cannot be interrupted
        raise
```

Graceful first, with a bounded budget; forceful and synchronous as the fallback.

### 2.5 The uncancel problem

A subtle one worth knowing before it bites.

When `asyncio.timeout` fires it calls `task.cancel()`, and when the `CancelledError` reaches
the block it calls `task.uncancel()` and raises `TimeoutError` instead. `Task.uncancel()`
decrements a counter. If your code catches `CancelledError` in between and does something
that awaits, the bookkeeping can be confused about *whose* cancellation this was.

The practical rules:

- **Do not catch `CancelledError` without re-raising**, and do not catch it merely to log.
- If you must handle it, keep the handler short and synchronous where possible.
- If you are writing a library with cancel scopes, read `Task.cancelling()` and
  `Task.uncancel()` and test the nesting cases explicitly.

`trio`'s design avoids this class of problem by making cancel scopes first-class and
cancellation *level-triggered* rather than edge-triggered: in trio, a cancelled scope keeps
delivering cancellation at every checkpoint until you leave the scope, so "catching it and
continuing" is not possible by accident. That is a better design and it is worth
understanding *why* when arguing about async runtimes.

### 2.6 Graceful shutdown

The full sequence for a service, in order, with the reason for each step:

1. **Receive the signal.** `SIGTERM` from an orchestrator, `SIGINT` from a terminal.
   Register with `loop.add_signal_handler` (POSIX) — a plain `signal.signal` handler runs on
   the main thread between bytecodes and cannot safely touch loop state.
2. **Stop accepting new work.** Close the listening socket; fail readiness checks so the
   load balancer stops routing to you *before* you start refusing. In Kubernetes this
   ordering matters: fail readiness, wait for the endpoint propagation delay (a few
   seconds), then stop accepting.
3. **Drain in-flight work** with a deadline. Wait for outstanding requests; the deadline
   must be less than the orchestrator's grace period (`terminationGracePeriodSeconds`),
   or you will be `SIGKILL`ed mid-drain.
4. **Cancel what remains** and await the cancellations.
5. **Flush.** Metrics, logs, traces, the outbox. This is the step people forget, and it is
   why the last minute before a deploy is invisible in your dashboards.
6. **Close resources.** Connection pools, files, subprocesses — in reverse acquisition order
   (`AsyncExitStack`).
7. **Exit.**

```python
async def main() -> None:
    stop = asyncio.Event()
    loop = asyncio.get_running_loop()
    for sig in (signal.SIGTERM, signal.SIGINT):
        loop.add_signal_handler(sig, stop.set)

    async with AsyncExitStack() as stack:
        app = await stack.enter_async_context(build_app())
        server = await stack.enter_async_context(serve(app))
        await stop.wait()                       # run until signalled
        log.info("shutdown: draining")
        await server.stop_accepting()
        try:
            async with asyncio.timeout(DRAIN_SECONDS):
                await server.wait_for_inflight()
        except TimeoutError:
            log.warning("drain timed out; cancelling %d", server.inflight_count)
        # AsyncExitStack unwinds: cancel tasks, flush, close pools
```

**`SIGKILL` cannot be handled.** Whatever is in flight is lost. The only defence is
**idempotence on the server side and at-least-once semantics** — which is the same
conclusion as SE-521 L07 and DS-701 L08, reached from a third direction. When you find
yourself adding more shutdown code to close a correctness gap, you are solving the wrong
problem.

### 2.7 What can go wrong at shutdown

Real failures, each worth testing for:

- **Tasks cancelled mid-write**, leaving partial records. Idempotence, or transactions.
- **Cleanup that awaits, cancelled.** §2.4 pattern 5.
- **Async generators not finalized**, so their `finally` never runs. `asyncio.run` calls
  `shutdown_asyncgens`; a hand-rolled loop must too (PY-502 L05 §2.4, L06 §2.7).
- **A thread-pool task that cannot be cancelled**, blocking `loop.shutdown_default_executor`
  past the grace period.
- **"Event loop is closed"** — a callback scheduled after the loop stopped. Usually a
  finalizer or a background task nobody tracked.
- **Logs and traces lost**, because handlers were not flushed (step 5).
- **Deadlock at shutdown** — a task awaiting something only another cancelled task would
  have provided.
- **A shielded operation with no timeout**, hanging forever.

## 3. Construction: a shutdown-correct worker

Build a queue worker that pulls jobs, processes them, and writes results, and make it
correct under cancellation at every point.

**Stage 1 — the naive version.** A `while True` loop, `await queue.get()`, process, write.
No signal handling.

**Stage 2 — enumerate the interruption points.** Write out every `await` in the worker and,
for each, what state exists if `CancelledError` arrives there:

| `await` | If cancelled here… |
|---|---|
| `queue.get()` | nothing in flight — safe |
| `process(job)` | job dequeued but not done → **lost**, unless the queue redelivers |
| `db.write(result)` | maybe written, maybe not → **unknown** |
| `queue.ack(job)` | written but not acked → **will be redelivered** → duplicate |

That table is the exercise. It shows that the worker's correctness is not about shutdown
code at all: it is about **whether the queue redelivers and whether the write is
idempotent**. Two properties, decided at design time, and no amount of `finally` substitutes
for them.

**Stage 3 — make it correct.** At-least-once delivery from the queue plus an idempotent
write keyed by job id. Now cancellation at any point is safe: the job is either done or
redelivered, and doing it twice is harmless. Prove it with a test that cancels at each of
the four points and asserts the invariant.

**Stage 4 — graceful shutdown.** Signal handling, stop accepting, drain with a deadline,
cancel the rest, flush, close. Test by sending `SIGTERM` while jobs are in flight and
asserting: no job lost, no job duplicated *in effect*, all metrics flushed, exit code 0, and
total shutdown time under the deadline.

**Stage 5 — the hostile tests.** Each of these should be a passing test:

- `SIGTERM` during `db.write`.
- `SIGTERM` twice (the second should force immediate exit, not confuse the first).
- A job whose `process()` ignores cancellation for 30 s (the drain deadline must still be
  honoured, and the process must still exit).
- The database is down during shutdown flush.
- `SIGKILL` — assert recovery on restart, which is a test of the *next* run.

**Stage 6 — deadline propagation.** Add a budget carried in a `ContextVar` from job
enqueue-time, so a job that has been queued for 90 seconds does not get a fresh 30-second
processing budget. Report what this changed about your tail latency under backlog.

## 4. Failure modes

- **`except Exception` around cancellable code, assuming it catches everything.** It does not
  catch `CancelledError` — which is correct, and people are surprised by both directions.
- **`except BaseException: pass`.** Makes the service unshutdownable.
- **Catching `CancelledError` and not re-raising.**
- **`task.cancel()` without `await task`.**
- **CPU-bound loops in coroutines.** Uncancellable.
- **Per-call timeouts with no total budget.** Cascading multiplication.
- **`shield` without a timeout.** Hangs at shutdown.
- **Cleanup that awaits, unprotected.** Cancelled midway.
- **Drain deadline longer than the orchestrator's grace period.** `SIGKILL` mid-drain.
- **Not failing readiness before draining.** New requests arrive during shutdown.
- **Not flushing telemetry.** The most interesting minute of your service's life is
  invisible.
- **Solving a cancellation-safety problem with more shutdown code** instead of idempotence.

## 5. Exercises

### Warm-up (30 min)

**W1.** Show that `except Exception:` does not catch `CancelledError`, and that
`except BaseException: pass` makes a task uncancellable.

**W2.** Write a coroutine that ignores cancellation for five seconds. Show what it does to a
`TaskGroup`'s exit and to a `timeout` block.

**W3.** Demonstrate the cascading-timeout multiplication: three hops, 2 s each, two retries,
and measure the worst case.

### Core (2.5 h)

**C1 — The worker.** Complete §3, all six stages. Deliverable: the interruption-point table,
the correctness argument (redelivery + idempotence), the five hostile tests, the shutdown
timings, and the deadline-propagation measurement.

**C2 — Deadline propagation.** Implement §2.3's budget for a three-level call chain. Compare
worst-case total latency against per-call timeouts, with and without retries. Report the
numbers and write the 300-word argument you would use to get it adopted.

**C3 — Cancellation-safety audit.** Take a real async codebase. For every `async def` that
performs more than one mutating operation, determine what is left inconsistent if cancelled
between them. Produce the table. For the three worst, propose a fix (transaction, saga, or
idempotence) and implement one.

**C4 — The shutdown harness.** Write a reusable test fixture that starts your service in a
subprocess, drives load, sends a signal at a configurable point, and asserts: exit code,
shutdown duration, no lost work, no duplicated *effects*, and telemetry flushed. Run it 100
times with the signal at random points and report any failure.

### Challenge

**X1.** Implement trio-style cancel scopes on `asyncio`: a scope object with a deadline and
`cancel()`, correct nesting, and level-triggered semantics (a cancelled scope keeps
delivering cancellation at every checkpoint until exited). Test the identity problem: an
inner scope's cancellation must not be mistaken for an outer one's. Then write 800 words on
why `asyncio`'s edge-triggered design makes this hard, referring to `Task.cancelling()` and
`uncancel()`.

**X2.** Build a chaos test for cancellation: instrument an async application so that every
`await` has a configurable probability of injecting a cancellation, then run the test suite
with injection enabled. Report every invariant violation found. This is essentially fault
injection for cancellation, it is rarely done, and it finds real bugs.

## 6. Self-check

1. What does `task.cancel()` actually do, and what does it not do?
2. Why is `CancelledError` a `BaseException`, and what two rules follow?
3. What does `asyncio.timeout` provide that `wait_for` does not?
4. Explain the cascading-timeout problem and the deadline-propagation fix.
5. What does `shield` protect, what does it not protect, and why must it have a timeout?
6. Give the two-phase cleanup pattern and why it is needed.
7. Give the seven steps of graceful shutdown in order, with the reason for the ordering of
   steps 2 and 3.
8. Why is idempotence a better answer than more shutdown code?

## 7. Primary sources

- `asyncio` documentation: "Task Cancellation", `asyncio.timeout`, `asyncio.shield`.
- PEP 654 and the `Task.uncancel`/`cancelling` API notes.
- `trio` documentation, "Cancellation and timeouts" — read this even if you use `asyncio`;
  it is the clearest exposition of the semantics anywhere.
- Smith, "Timeouts and cancellation for humans" (2018).
- Kubernetes docs on pod termination and `terminationGracePeriodSeconds` — for §2.6 step 3.

---

**Previous:** [L06](L06-asyncio-and-structured-concurrency.md) · **Next:**
[L08 — Processes, Subinterpreters, and Shared Memory](L08-processes-and-subinterpreters.md)
