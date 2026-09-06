# PY-502 · Lesson 07 — Context Managers and Resource Composition

**Estimated study time:** 3.5 hours
**Prerequisites:** PY-501 L06, L09, L10; PY-502 L06

---

## 1. Orientation

```python
def open_all(paths):
    files = [open(p) for p in paths]
    try:
        yield files
    finally:
        for f in files:
            f.close()
```

Two bugs. If `open` fails on the fourth path, the first three are never closed — the
exception escapes before the `try` is entered. And if the second `f.close()` raises, the
rest are never closed.

Resource management looks solved in Python — `with` handles it — right up to the moment the
number of resources is dynamic, or acquisition can fail partway, or cleanup can fail. Then
it needs actual design. This lesson is that design.

## 2. Theory

### 2.1 The protocol, and the two rules

```python
class Resource:
    def __enter__(self) -> Handle: ...
    def __exit__(self, exc_type, exc, tb) -> bool | None: ...
```

**Rule 1: `__enter__`'s return value is what `as` binds.** It need not be `self`. Returning
a *different* object is often better design: `with pool.connection() as conn` should bind a
connection, not the pool.

**Rule 2: a truthy return from `__exit__` suppresses the exception.** `None` (the default)
propagates it. This is the single most dangerous line in the protocol, because a
`__exit__` whose last statement is an expression can accidentally return truthy:

```python
def __exit__(self, *exc):
    return self.log.info("closed")     # returns None — fine, by luck
def __exit__(self, *exc):
    return self.cleanup()              # if cleanup() returns True... exceptions vanish
```

Enable `ruff`'s bugbear rules and write an explicit `return None` or nothing at all.

**Reentrancy and reusability** are separate properties, both non-default:

- **Reusable** — can be entered again after exiting. `threading.Lock` is; a
  `@contextmanager` generator is *not* (its generator is exhausted).
- **Reentrant** — can be entered while already entered, by the same code.
  `threading.RLock` is; `threading.Lock` is not (it deadlocks).

`contextlib` provides `nullcontext`, and `contextlib.ContextDecorator` lets a context
manager also be used as a decorator — but note that a `@contextmanager`-based one used as a
decorator is single-use per call, which the implementation handles by re-invoking the
generator function. Know which of your context managers are reusable and document it.

### 2.2 `ExitStack`: the general solution

`contextlib.ExitStack` maintains a stack of cleanup callbacks and unwinds them in reverse
order on exit, **including if an exception occurs during acquisition**. It solves the
opening problem exactly:

```python
from contextlib import ExitStack

def open_all(paths: Sequence[Path]) -> Iterator[list[TextIO]]:
    with ExitStack() as stack:
        files = [stack.enter_context(open(p)) for p in paths]
        yield files
```

If the fourth `open` raises, the stack unwinds the three already registered. If a `close`
raises during unwinding, `ExitStack` still runs the remaining ones and chains the
exceptions.

The full API is worth knowing:

| Method | Use |
|---|---|
| `enter_context(cm)` | enter a context manager, register its exit |
| `callback(fn, *a, **kw)` | register an arbitrary cleanup call |
| `push(exit_fn)` | register something with an `__exit__`-shaped signature |
| `pop_all()` | **transfer ownership** — see §2.3 |
| `close()` | unwind now |

`AsyncExitStack` is the `async with` equivalent, with `enter_async_context` and
`push_async_callback`.

**`ExitStack` is the right answer whenever the set of resources is not statically known.**
That includes: a list of files, a variable number of database connections, a plugin's
declared resources, and the common case of "acquire a resource conditionally".

```python
with ExitStack() as stack:
    conn = stack.enter_context(db.connect())
    if needs_cache:
        cache = stack.enter_context(redis.client())
    if dry_run:
        stack.enter_context(transaction_rollback_always(conn))
```

That is much cleaner than nested `if`s with duplicated `with` blocks, and it is the idiom
worth internalizing from this lesson.

### 2.3 The transfer-of-ownership problem

A constructor that acquires several resources must either succeed completely or clean up
completely. `pop_all()` is the mechanism:

```python
class Session:
    def __init__(self, cfg: Config) -> None:
        with ExitStack() as stack:
            self._conn = stack.enter_context(db.connect(cfg.dsn))
            self._cache = stack.enter_context(redis.client(cfg.redis))
            self._tmp = stack.enter_context(TemporaryDirectory())
            # everything acquired successfully — take ownership of the cleanups
            self._stack = stack.pop_all()

    def close(self) -> None:
        self._stack.close()

    def __enter__(self) -> "Session": return self
    def __exit__(self, *exc) -> None: self.close()
```

If any acquisition fails, the `with ExitStack()` block unwinds and everything acquired so
far is released; the constructor raises and no half-built `Session` exists. If all succeed,
`pop_all()` moves the cleanup callbacks to a new stack that the object owns.

**This is the correct pattern for any object owning multiple resources**, and almost nobody
writes it. The alternative — acquiring in `__init__` with a bare `try/except` that closes
what it can — is longer, easier to get wrong, and does not handle exceptions during
cleanup.

### 2.4 Cleanup that can fail

Three questions with no universal answer, so decide them explicitly:

**Should a cleanup failure mask the original exception?** Usually no. The original exception
is why you are unwinding, and it is the more informative one. `ExitStack` chains rather than
replaces, so the original appears as `__context__` — read the whole chain when debugging.

**Should one failing cleanup stop the others?** Usually no. Continue and aggregate:

```python
def close_all(resources: Sequence[Closeable]) -> None:
    errors: list[BaseException] = []
    for r in reversed(resources):
        try:
            r.close()
        except BaseException as e:
            errors.append(e)
    if errors:
        raise ExceptionGroup("failures during cleanup", errors)
```

`ExceptionGroup` (PY-501 L10 §2.5) is exactly right here: multiple independent failures.

**Is cleanup idempotent?** It must be. `close()` will be called twice — by an explicit
call and by `__exit__`, or by `__exit__` and a finalizer. Make it safe:

```python
def close(self) -> None:
    if self._closed:
        return
    self._closed = True
    ...
```

### 2.5 The three-layer resource discipline

For anything holding an OS resource, all three layers, in this order:

1. **A context manager** — the primary interface. Callers should use `with`.
2. **An explicit, idempotent `close()`** — for callers whose lifetime does not match a
   block (a connection pool, an object stored in a registry).
3. **A `weakref.finalize` safety net** — that emits a `ResourceWarning` and releases the
   resource if the object is collected without being closed.

```python
class Connection:
    def __init__(self, sock: socket.socket) -> None:
        self._sock = sock
        self._closed = False
        self._finalizer = weakref.finalize(self, self._cleanup, sock, repr(self))

    @staticmethod
    def _cleanup(sock: socket.socket, who: str) -> None:
        warnings.warn(f"unclosed {who}", ResourceWarning, stacklevel=2)
        sock.close()

    def close(self) -> None:
        if self._finalizer.detach():        # returns None if already detached
            self._closed = True
            self._sock.close()

    def __enter__(self) -> "Connection": return self
    def __exit__(self, *exc) -> None: self.close()
```

Note carefully: `_cleanup` is a **staticmethod** taking the socket, not `self`. A
`finalize` callback that references the object keeps it alive forever and never fires
(PY-501 L09 §2.5). This is the single most common mistake with `weakref.finalize` and it
fails silently.

`finalizer.detach()` is the idiomatic idempotence check — it returns the registered
callback the first time and `None` thereafter.

The standard library follows exactly this pattern for files, sockets, and subprocesses.
Run your tests with `-W error::ResourceWarning` and the safety net becomes a *test failure*
for every leak, which is the real payoff.

### 2.6 Async resources

`async with` uses `__aenter__`/`__aexit__`. Two differences that matter:

- **Cleanup can `await`**, which means cleanup can be *cancelled*. An `__aexit__` that
  awaits inside a cancelled task may itself be interrupted, leaving the resource
  half-released. `asyncio.shield` or `asyncio.timeout` around critical cleanup is the
  mitigation, and PY-601 L07 treats it properly.
- **Ordering with the event loop.** An async resource must not be closed from a different
  loop or after the loop has closed — a common source of "Event loop is closed" errors at
  shutdown.

`contextlib.asynccontextmanager` and `AsyncExitStack` mirror the sync versions. Everything
in §2.2–2.5 applies with `async` prefixes; §2.4's aggregation is more important, because
async cleanup failures are more likely.

### 2.7 Context managers as scoped state

Beyond resources, `with` is Python's mechanism for *dynamically scoped* behaviour:

```python
with decimal.localcontext() as ctx:  ctx.prec = 50; ...
with warnings.catch_warnings():      warnings.simplefilter("error"); ...
with tracer.span("checkout"):        ...
with mock.patch.object(...):         ...
```

The pattern is: save state, set new state, restore on exit. `contextvars.ContextVar`
(PEP 567) is the correct storage for this in concurrent code, because it is per-task rather
than per-thread and it propagates correctly into `asyncio` tasks:

```python
request_id: ContextVar[str] = ContextVar("request_id")

@contextlib.contextmanager
def request_scope(rid: str) -> Iterator[None]:
    token = request_id.set(rid)
    try:
        yield
    finally:
        request_id.reset(token)        # the token restores the *previous* value
```

Note `reset(token)` rather than `set(old_value)` — the token handles nesting correctly.

This is how distributed tracing, per-request logging context, and database session scoping
work in async frameworks. It is also a form of global mutable state, so the SE-521 L04
caution applies: it is convenient, it is invisible at the call site, and it makes functions
non-pure. Use it for genuinely ambient concerns (a trace id, a locale) and not as a way to
avoid passing arguments.

## 3. Construction: a resource-owning service object

Build `IngestService`, which owns: a database connection, an object-store client, a
temporary working directory, a metrics flusher, and a variable number of shard connections
determined by config.

**Step 1 — the wrong way, for the record.**

```python
class IngestService:
    def __init__(self, cfg):
        self.db = db.connect(cfg.dsn)
        self.store = s3.client(cfg.bucket)
        self.tmp = tempfile.mkdtemp()
        self.shards = [db.connect(d) for d in cfg.shards]
```

If the third shard fails to connect, the db connection, the store client, the temp
directory, and two shard connections all leak, and the exception the caller sees says
nothing about it. Under load — the situation in which connections fail — this leaks
resources fastest exactly when you can least afford it.

**Step 2 — `ExitStack` + `pop_all`.** Rewrite per §2.3. Verify with a fault-injecting fake
that fails at each acquisition point in turn (5 tests) that nothing is left open.

**Step 3 — the three layers.** Add `close()` (idempotent), `__enter__`/`__exit__`, and the
`weakref.finalize` safety net with `ResourceWarning`. Add
`filterwarnings = ["error::ResourceWarning"]` to your pytest config and prove that a test
which forgets to close now fails.

**Step 4 — cleanup that fails.** Make two of the fakes raise on close. Verify that: all
other resources still close, the failures are aggregated into an `ExceptionGroup`, and if
there was an original exception it is not masked. Write the test that would have caught a
regression in each.

**Step 5 — the async version.** Convert to `AsyncExitStack`. Then add the harder case: the
service is closed *while a cancellation is propagating*. Show what breaks, and fix it with
`asyncio.shield` around the critical part. Document precisely what is still not guaranteed
(the answer involves process death, and the correct response is idempotence on the server
side, not more cleanup code).

**Step 6 — scoped state.** Add a `ContextVar` carrying an ingest-run id, set by a context
manager, and verify it propagates into tasks created inside the scope and does not leak out
of it.

## 4. Failure modes

- **Acquiring N resources in a loop without `ExitStack`.** §1.
- **`__exit__` returning truthy accidentally.** Silent exception suppression.
- **Non-idempotent `close()`.** Double-close errors on the unwinding path.
- **`weakref.finalize` capturing `self`.** Never fires; object never collected.
- **Cleanup failure masking the original exception.**
- **One failing cleanup aborting the rest.**
- **`@contextmanager` reused.** `RuntimeError: generator didn't yield` on second use.
- **Async cleanup that awaits inside a cancelled scope.** Half-released resources.
- **`ContextVar` set without `reset`.** Leaks into sibling tasks or subsequent requests in
  a worker.
- **Using `contextvars` as a way to avoid passing arguments.** Invisible coupling.
- **Not testing the failure paths.** Every bug in this lesson lives on a path that a happy-
  path test never executes.

## 5. Exercises

### Warm-up (25 min)

**W1.** Reproduce both bugs in the opening example, then fix them with `ExitStack`.

**W2.** Write a `__exit__` that accidentally suppresses exceptions. Then write the test that
catches it.

**W3.** Write a `weakref.finalize` that captures `self` and show that it never fires.

### Core (2.5 h)

**C1 — The service object.** Complete §3, all six steps. Deliverable: the code, the five
fault-injection tests, the `ResourceWarning` demonstration, the cleanup-aggregation tests,
the async version with the cancellation analysis, and a 500-word note on what remains
unguaranteed and why the answer is idempotence rather than more cleanup.

**C2 — Audit.** Find every `__exit__` and `@contextmanager` in a real codebase. For each,
determine: can it suppress unintentionally? Is cleanup guaranteed on exception? Is it
reusable, and is that documented? Is it reentrant? Report the findings and fix the worst
three.

**C3 — Conditional acquisition.** Find code with nested or duplicated `with` blocks caused
by conditional resources. Rewrite with `ExitStack`. Report the diff and whether readability
improved (sometimes it does not — report honestly).

**C4 — Scoped state.** Implement a request-scoped context (request id, user, locale) with
`contextvars`. Verify: propagation into `asyncio.create_task`, propagation into a
`ThreadPoolExecutor` (it does *not* propagate automatically — find out why and fix it), no
leakage between requests, and correct nesting. Write 300 words on when this is better than
passing a parameter and when it is worse.

### Challenge

**X1.** Build a `ResourceScope` that supports: nested scopes, named resources retrievable
by key, lazy acquisition on first use, explicit release of one resource before scope exit,
and cleanup ordering constraints ("close the cache before the connection it uses"). Then
explain why the ordering constraint makes this a topological sort and what happens with a
cycle.

**X2.** Study `unittest.mock.patch`, `decimal.localcontext`, and `warnings.catch_warnings`.
For each, determine: what state it saves, whether it is thread-safe, whether it is
async-safe, and what happens if two are nested in the wrong order. Report a case where one
of them is *not* safe under concurrency, with a demonstration.

## 6. Self-check

1. State the two rules of the context manager protocol.
2. Distinguish reusable from reentrant, with an example of each.
3. What problem does `ExitStack` solve that nested `with` cannot?
4. Explain `pop_all()` and the transfer-of-ownership pattern.
5. Give the three-layer resource discipline and the role of each layer.
6. Why must a `weakref.finalize` callback not reference the object?
7. Should cleanup failure mask the original exception? Should one failure stop the others?
8. Why does `ContextVar.reset(token)` exist rather than setting the old value back?

## 7. Primary sources

- `contextlib` documentation and source — read `ExitStack`'s implementation, it is short
  and instructive.
- PEP 343 (`with`), PEP 492 (`async with`), PEP 567 (`contextvars`), PEP 654
  (`ExceptionGroup`).
- CPython `socket.py` and `subprocess.py` — the three-layer pattern in the wild.

---

**Previous:** [L06](L06-coroutines-and-delegation.md) · **Next:**
[L08 — Declarative Models: dataclasses, attrs, pydantic](L08-declarative-models.md)
