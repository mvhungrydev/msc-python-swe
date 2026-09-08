# PY-601 · Lesson 06 — `asyncio` in Practice and Structured Concurrency

**Estimated study time:** 4 hours
**Prerequisites:** L05

---

## 1. Orientation

Nathaniel J. Smith's 2018 essay "Notes on structured concurrency, or: Go statement
considered harmful" made an argument that has since reshaped async APIs across several
languages:

> **An unstructured `spawn` is the concurrency equivalent of `goto`.** It creates a
> control-flow edge that escapes the enclosing block, so you can no longer reason about a
> function by reading it: after `f()` returns, work it started may still be running, may
> still fail, and may still hold resources.

`goto` was eliminated by giving control flow *block structure*: a construct entered at the
top and left at the bottom, with the guarantee that what happened inside is finished.
Structured concurrency does the same for tasks: **a scope that does not exit until every
task created inside it has finished.**

Python got this as `asyncio.TaskGroup` in 3.11, following `trio`'s nursery. This lesson is
about using `asyncio` well, and structured concurrency is the organizing idea.

## 2. Theory

### 2.1 The problem with `create_task`

```python
async def handle(request):
    asyncio.create_task(audit(request))     # fire and forget
    return await process(request)
```

Four separate problems:

1. **The exception is lost.** If `audit` raises, nothing observes it; you get a log line from
   a destructor, later, maybe.
2. **The task may be garbage collected mid-execution.** The loop holds only a weak reference.
3. **When `handle` returns, `audit` may still be running.** The caller has no way to know.
4. **At shutdown, it is killed at an arbitrary point**, possibly mid-write.

You cannot fix these with discipline; you fix them with structure.

### 2.2 `TaskGroup`

```python
async def handle(request):
    async with asyncio.TaskGroup() as tg:
        audit_task = tg.create_task(audit(request))
        result_task = tg.create_task(process(request))
    return result_task.result()
```

The guarantees, and each one removes a problem above:

- **The `async with` block does not exit until every task created in it has completed.** So
  when `handle` returns, nothing it started is still running.
- **If any task raises, the others are cancelled**, and the group waits for their
  cancellation to complete before propagating.
- **Errors are aggregated into an `ExceptionGroup`** (PY-501 L10 §2.5), because several may
  fail. Handle with `except*`.
- **Cancellation of the enclosing scope propagates inward** to every child.

The resulting property is the important one: **you can reason about a function locally
again.** Reading `handle`, you know that when it returns, its concurrency is finished. That
is exactly what block structure gave to sequential control flow.

Practical notes:

- Calling `tg.create_task()` after the block has begun exiting raises. This is deliberate.
- A task that ignores cancellation blocks the group's exit indefinitely; §2.5 and L07.
- `tg.create_task` returns a `Task`, and you read results after the block, from the task
  objects.

### 2.3 `gather`, `wait`, `as_completed`, `wait_for`

The pre-TaskGroup toolkit, still present and still needed for specific shapes:

**`gather(*aws, return_exceptions=False)`** — run concurrently, return results in order.

The default is a trap: on the first exception, `gather` propagates it immediately **but does
not cancel or wait for the other awaitables** — they keep running, unobserved. With
`return_exceptions=True` it waits for all and returns exceptions as values, which is safer
and often what you want.

Rule: **use `TaskGroup` unless you specifically need results-in-order-with-exceptions-as-
values**, in which case `gather(..., return_exceptions=True)` inside a `TaskGroup` is a
reasonable combination.

**`wait(tasks, return_when=...)`** — returns `(done, pending)`. It does **not** cancel the
pending ones and does not raise their exceptions. Useful for "first to finish wins"
(`FIRST_COMPLETED`) — and you must then cancel the losers yourself, and await their
cancellation.

**`as_completed(aws)`** — yields awaitables in completion order. Right for streaming results
as they arrive.

**`wait_for(aw, timeout)`** — cancels on timeout and raises `TimeoutError`. Since 3.11,
`asyncio.timeout()` as a context manager is better (L07 §2.3), because it composes and
applies to a *block* rather than a single awaitable.

**`shield(aw)`** — protects an awaitable from *outer* cancellation. Necessary for critical
cleanup, dangerous in general (L07 §2.5).

### 2.4 Synchronization primitives

`asyncio.Lock`, `Event`, `Condition`, `Semaphore`, `BoundedSemaphore`, `Queue`,
`Barrier` (3.11+). Same shapes as `threading`, with two crucial differences:

- **They are not thread-safe.** They coordinate tasks on one loop. Using an `asyncio.Lock`
  across threads is a bug. Conversely, using a `threading.Lock` in a coroutine *blocks the
  loop* if it is contended (§L05 §2.6).
- **You need them less often**, because a coroutine only suspends at `await`. Any sequence of
  statements with no `await` in it is atomic with respect to other tasks on the same loop.

That second point is the most useful practical fact about `asyncio`, and its corollary is the
most common async bug:

```python
async def transfer(a, b, amount):
    balance = await get_balance(a)      # ← suspension point
    if balance >= amount:               # the world may have changed here
        await debit(a, amount)          # ← and here
        await credit(b, amount)
```

Every `await` is a place another task can run and invalidate what you just read. **Read your
code and mark the `await`s: those are your interleaving points.** This is far more tractable
than thread interleaving (which can occur between any two bytecodes), and it is the main
reason async code is easier to reason about than threaded code.

`asyncio.Semaphore` is the standard concurrency limiter:

```python
sem = asyncio.Semaphore(10)
async def fetch(url):
    async with sem:
        return await client.get(url)
```

### 2.5 Cancellation, briefly (L07 has the detail)

`task.cancel()` schedules a `CancelledError` to be raised at the task's next suspension
point. Facts to carry into L07:

- **`CancelledError` inherits from `BaseException`**, deliberately, so that
  `except Exception:` does not swallow it. Code that catches `BaseException` must re-raise.
- **Cancellation is cooperative.** A task that never awaits cannot be cancelled.
- **A task can refuse.** Catching `CancelledError` and continuing is legal and almost always
  wrong; if you catch it to clean up, re-raise.
- **`finally` blocks run** during cancellation, and an `await` inside a `finally` can itself
  be cancelled (L07 §2.5).

### 2.6 Running blocking code

```python
result = await asyncio.to_thread(blocking_fn, arg)          # 3.9+, convenient
result = await loop.run_in_executor(process_pool, cpu_fn)   # explicit executor
```

`to_thread` uses the default thread executor. Points that matter:

- The default executor's size is `min(32, cpu_count + 4)`. For an I/O-heavy service that is
  often too small and is a hidden bottleneck; create your own executor with a size derived
  from Little's Law (L09 §2.2).
- `to_thread` propagates `contextvars` correctly; a raw `run_in_executor` with a bare
  function does not (`contextvars.copy_context().run` is the fix).
- **A blocked thread cannot be cancelled.** `to_thread` returns an awaitable you can cancel,
  but the thread keeps running to completion. This surprises people during shutdown.
- CPU-bound work needs a **process** pool, not a thread pool (L02 §2.3, L08).

### 2.7 Async iteration, context managers, and generators

```python
async for chunk in response.aiter_bytes(): ...
async with session.begin(): ...

async def stream(url):                 # an async generator
    async with client.stream("GET", url) as r:
        async for chunk in r.aiter_bytes():
            yield chunk
```

Async generators are useful and have a genuinely awkward corner: **finalization**. When an
async generator is abandoned, its `aclose()` must be *awaited*, but garbage collection is
synchronous and cannot await. `asyncio` handles this with
`loop.shutdown_asyncgens()` (called by `asyncio.run`), which finalizes outstanding async
generators at loop shutdown — but if the loop is already closed, or the generator is
collected at an awkward moment, cleanup can be skipped or produce
"async generator ignored GeneratorExit".

Practical rules:

- Use `contextlib.aclosing()` around an async generator you may abandon:
  `async with aclosing(stream(url)) as s: async for c in s: ...`
- Always use `asyncio.run()` (or an equivalent that calls `shutdown_asyncgens`).
- Prefer an async *iterator class* over an async generator when it owns a resource and the
  consumer may stop early — the lifetime is then explicit.

### 2.8 Ecosystem and alternatives

- **`uvloop`** — a `libuv`-based loop, typically 2–4× faster than the default for
  network-heavy workloads. A one-line change. Measure; the gain depends on how much of your
  time is in the loop rather than in your handlers.
- **`trio`** — structured concurrency from the ground up: nurseries are mandatory, there is
  no `create_task`, cancellation is scope-based and checkpoint-based, and the design is more
  coherent than `asyncio`'s. Smaller ecosystem.
- **`anyio`** — an abstraction over both, providing trio-style structure on top of
  `asyncio`. The pragmatic choice for a *library*, because it works under either runtime.
  Its `CancelScope` and `create_task_group` are worth learning even if you stay on
  `asyncio`.

If you are writing a library that does async I/O, seriously consider `anyio`, or better,
consider a sans-I/O core (PY-502 L06 §2.6) with thin async and sync wrappers — that removes
the choice entirely and is the design most likely to still be right in five years.

## 3. Construction: an async service done properly

Build a small service: an HTTP API that, per request, fans out to three upstreams, does one
database call, and publishes an event.

**Stage 1 — the naive version.** `create_task` for the event publish, `gather` for the
fan-out, `wait_for` for the timeout, a `requests` call for one upstream (deliberately).

**Stage 2 — find the problems.** Under load, with one upstream made slow:

- The `requests` call stalls the entire loop. Measure the effect on unrelated endpoints
  (`loop.set_debug(True)` and a low `slow_callback_duration`).
- `gather` without `return_exceptions` means one upstream failing leaves the other two
  running unobserved. Demonstrate it: make one fail fast and the others slow, and show the
  slow ones still executing after the handler returned.
- The fire-and-forget publish loses exceptions. Demonstrate.
- On shutdown, in-flight publishes are killed. Demonstrate.

**Stage 3 — structure it.**

```python
async def handle(req: Request) -> Response:
    async with asyncio.timeout(2.0):
        async with asyncio.TaskGroup() as tg:
            a = tg.create_task(upstream_a(req))
            b = tg.create_task(upstream_b(req))
            c = tg.create_task(upstream_c(req))
        results = (a.result(), b.result(), c.result())
    row = await db.fetch(req.id)
    await publisher.publish(Event(req.id, results, row))    # awaited, not fire-and-forget
    return Response(...)
```

Then ask the questions the structure forces you to answer:

- **Should one failed upstream fail the request?** `TaskGroup` says yes by default. If not,
  wrap each in a coroutine that catches and returns a sentinel — and now the *policy* is
  explicit and testable rather than emergent.
- **Should the publish block the response?** If it must be durable, yes — or use an outbox
  (SE-521 L07 §2.5) and stop pretending fire-and-forget is durable.
- **What is the timeout budget?** 2 s total, or 2 s per upstream? `asyncio.timeout` around
  the group gives a total budget, which is almost always what you want; per-call timeouts
  additionally protect against one call consuming the whole budget.

**Stage 4 — concurrency limits.** Add a `Semaphore` per upstream so that one slow dependency
cannot consume all your capacity — this is the bulkhead pattern (SE-521 L09 §2.6). Measure
what happens under a slow upstream with and without it.

**Stage 5 — shutdown.** On `SIGTERM`: stop accepting, wait for in-flight requests with a
deadline, cancel the rest, close the pools, flush the publisher. Test it by sending SIGTERM
mid-request. L07 §3 is this in detail.

**Stage 6 — measure.** Throughput and p50/p99/p999 latency for stages 1 and 3, with a
healthy upstream and with one upstream at p99 = 5 s. The difference under the degraded case
is the entire argument for the structure.

## 4. Failure modes

- **`create_task` without holding a reference or awaiting.** §2.1.
- **`gather` default behaviour on error.** Siblings keep running unobserved.
- **`wait` without cancelling the pending set.**
- **Blocking calls in coroutines.** L05 §2.6.
- **`threading.Lock` in a coroutine.** Blocks the loop when contended.
- **`asyncio.Lock` across threads.** Not thread-safe.
- **Assuming a read-check-write sequence is atomic across `await`s.** §2.4.
- **Catching `BaseException` and not re-raising `CancelledError`.**
- **Async generators abandoned without `aclosing`.**
- **The default thread executor as a hidden bottleneck.**
- **No timeout on anything.** Every `await` on a network resource needs a deadline.
- **Per-call timeouts with no total budget**, so three retries of a 2 s timeout produce a
  6 s request.

## 5. Exercises

### Warm-up (30 min)

**W1.** Demonstrate `gather`'s default behaviour: one fails immediately, two are slow; show
the slow ones still running after `gather` raised. Then fix it three ways.

**W2.** Show that a `threading.Lock` held by another thread blocks the whole event loop.

**W3.** Show that a sequence of statements without `await` is atomic with respect to other
tasks, and that inserting an `await` breaks it.

### Core (2.5 h)

**C1 — The service.** Complete §3, all six stages. Deliverable: both versions, the four
demonstrated problems from stage 2 with measurements, the structured version, the bulkhead
experiment, the shutdown test, and the latency comparison under a degraded upstream.

**C2 — TaskGroup semantics.** Write tests establishing, for `asyncio.TaskGroup`: what
happens when one child fails; when two fail; when the body raises; when the group is
cancelled from outside; when a child ignores cancellation; and when `create_task` is called
during exit. Document each observed behaviour. Then do the same for `anyio`'s task group and
report the differences.

**C3 — Timeout budgets.** Implement a deadline that propagates through a call chain (a
`contextvar` holding an absolute deadline, with each call computing its remaining budget).
Compare with per-call timeouts on a three-level chain with retries. Report the worst-case
total latency of each.

**C4 — `uvloop` and the loop's share.** Benchmark a realistic async workload on the default
loop and `uvloop`, at three levels of per-request handler work (trivial, 1 ms, 10 ms).
Report the speedup at each. Explain why the gain shrinks as handler work grows — and derive
the general rule about when a faster runtime is worth adopting.

### Challenge

**X1.** Read Smith's "Notes on structured concurrency" and the `trio` design documents.
Write 1,500 words: state the `goto` analogy precisely, identify what `asyncio.TaskGroup`
does and does not adopt from nurseries (cancel scopes, checkpoints, the absence of a bare
`spawn`), and argue whether `asyncio` should deprecate `create_task` outside a group.

**X2.** Implement a `CancelScope` for `asyncio` in the trio style: a scope object with a
deadline, `cancel()`, and correct nesting semantics (an inner scope's cancellation must not
be mistaken for an outer one's — the "cancel scope" identity problem). Test the nesting
cases. Then explain why `asyncio`'s design makes this harder than trio's, referring to
uncancel counts (PEP 654-era `Task.uncancel`).

## 6. Self-check

1. State the `goto` analogy for unstructured concurrency and what block structure restores.
2. Give four guarantees `TaskGroup` provides.
3. What does `gather` do on the first exception by default, and why is that dangerous?
4. Why are `asyncio` primitives not thread-safe, and why do you need them less often?
5. What are the interleaving points in async code, and why is that easier than threads?
6. Give three facts about `to_thread` that surprise people.
7. Why is async generator finalization awkward, and what are the two mitigations?
8. When is `uvloop` worth adopting, and what determines the size of the gain?

## 7. Primary sources

- Smith, "Notes on structured concurrency, or: Go statement considered harmful" (2018).
  Required reading; it is one of the best software-design essays of the last decade.
- `trio` documentation, "Nurseries and spawning" and "Cancellation and timeouts".
- PEP 654 (`ExceptionGroup` / `except*`) — the *Motivation* section is about exactly this.
- `asyncio` documentation, "Developing with asyncio" (the debug-mode section) and the
  `TaskGroup` reference.
- `anyio` documentation.

---

**Previous:** [L05](L05-event-loop-from-first-principles.md) · **Next:**
[L07 — Cancellation, Timeouts, and Shutdown](L07-cancellation-and-shutdown.md)
