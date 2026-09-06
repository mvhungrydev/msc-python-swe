# PY-502 · Lesson 04 — Decorators: Design, Composition, and Typing

**Estimated study time:** 4 hours
**Prerequisites:** PY-501 L03, L07; SE-511 L07

---

## 1. Orientation

A decorator is just `f = deco(f)`. The mechanism takes ten minutes to learn. What takes
practice is everything around it: preserving the wrapped function's identity, working
correctly on methods and classmethods, composing in a defined order, being introspectable,
carrying state without leaking it, supporting both sync and async, and — the one that gets
skipped — keeping the type checker informed.

The reason this matters: decorators are how cross-cutting concerns enter a Python codebase.
Retries, caching, authorization, tracing, validation, rate limiting, transactions. Every one
of those is a decorator in most systems, and a badly-built decorator is a defect multiplied
by every function it is applied to.

## 2. Theory

### 2.1 The four shapes

```python
# 1. plain
@deco
def f(): ...
# f = deco(f)

# 2. parameterized — a decorator factory
@deco(n=3)
def f(): ...
# f = deco(n=3)(f)

# 3. optionally parameterized — usable as @deco or @deco(n=3)
# 4. class decorator (L03 §2.3)
```

Shape 3 requires a little care and is worth doing, because callers will inevitably write
both forms:

```python
def retry(fn=None, /, *, attempts=3):
    def deco(fn):
        @functools.wraps(fn)
        def wrapper(*a, **kw): ...
        return wrapper
    return deco if fn is None else deco(fn)
```

The positional-only marker matters: it prevents `retry(fn=something)` from being
misinterpreted.

### 2.2 `functools.wraps` and what it actually copies

```python
@functools.wraps(fn)
def wrapper(*a, **kw): ...
```

Copies `__module__`, `__name__`, `__qualname__`, `__doc__`, `__dict__`, and — critically —
sets `__wrapped__ = fn`.

`__wrapped__` is what makes `inspect.signature` report the *original* signature, and what
lets `inspect.unwrap` walk a decorator stack. Without `wraps`:

- Tracebacks say `wrapper`, everywhere.
- `help()` shows nothing useful.
- `inspect.signature` reports `(*a, **kw)`.
- Frameworks that introspect signatures — `pytest` fixtures, FastAPI, `click`, dependency
  injectors — break, sometimes silently.
- Two decorated functions may collide in a registry keyed by `__name__`.

**Always use `wraps`.** The one nuance: `wraps` copies `__dict__` by *update*, so attributes
your decorator sets before `wraps` runs are preserved, and attributes on the wrapped
function shadow nothing. If you attach metadata (`wrapper.is_retryable = True`), do it
after.

### 2.3 Decorators on methods

A decorator applied inside a class body sees a plain **function**, not a method — binding
happens later, via the descriptor protocol (PY-501 L03 §2.4). So `self` arrives as the
first positional argument of `wrapper`:

```python
class Service:
    @retry(attempts=3)
    def fetch(self, url): ...
    # wrapper(*a) receives (service_instance, url)
```

That works because `wrapper` is itself a plain function and therefore itself a non-data
descriptor. Three things go wrong:

**Order with `classmethod`/`staticmethod`.** These return *descriptor objects*, not
functions. A decorator applied on top of them receives a `classmethod` object, which is not
callable in the old days and has no `__name__`.

```python
class C:
    @classmethod          # must be OUTERMOST
    @retry()
    def make(cls): ...
```

Since 3.9 `classmethod` and `staticmethod` are more cooperative (they forward
`__wrapped__` and are callable), but the rule stands: **`classmethod`/`staticmethod`
outermost**, and `property` likewise (`@property` outermost, `@x.setter` on the setter).

**Per-instance state.** A decorator that caches at decoration time caches *per function*,
which for a method means *shared across all instances*. `functools.lru_cache` on a method
keys on `self`, retaining every instance forever (PY-501 L09 §4). If you want per-instance
behaviour on a method, you need a descriptor (L02), not a decorator.

**Identity.** `obj.decorated_method` produces a new bound method each access, so it cannot
be compared by identity or reliably removed from a callback list.

### 2.4 Composition and order

```python
@a
@b
def f(): ...
# f = a(b(f))
```

Bottom-up application; top-down execution at call time. `a`'s wrapper runs first.

Order matters whenever the decorators are not independent:

- `@cache` above `@retry` caches the *retried* result — right.
  `@retry` above `@cache` retries the *cache lookup* — wrong, and retries a deterministic
  operation.
- `@authorize` must be above `@audit` if you want denied requests audited; below if you
  do not.
- `@transaction` above `@retry` retries inside one transaction (probably wrong);
  below retries the whole transaction (probably right).
- `@timing` position determines what it measures.

**Design rule:** document the required order at the definition of each decorator, and where
order is critical, make it enforceable — a decorator can inspect `__wrapped__` chains and
raise at import time if it finds a forbidden neighbour. That check costs twenty lines and
prevents a whole class of subtle bug.

### 2.5 State in decorators

Three places state can live, with different lifetimes:

```python
def counted(fn):
    calls = 0                       # (a) per-decoration, in a closure cell
    @functools.wraps(fn)
    def wrapper(*a, **kw):
        nonlocal calls
        calls += 1
        return fn(*a, **kw)
    wrapper.calls = lambda: calls   # expose it
    return wrapper
```

- **(a) Closure cell** — one copy per decorated function. Not thread-safe (`calls += 1` is
  read-modify-write across bytecodes — PY-601 L04); use `itertools.count` or a lock if it
  matters.
- **(b) On the wrapper function object** — same lifetime, but *introspectable and
  mutable from outside*, which is how `lru_cache` exposes `cache_info()` and
  `cache_clear()`. Prefer this for anything callers may need.
- **(c) Module-level** — shared across all decorated functions. Registries live here; be
  explicit that it is global state.

For anything with per-instance semantics, none of these work. Use a descriptor.

### 2.6 Typing decorators

Without help, a decorator erases the signature:

```python
def deco(fn: Callable[..., R]) -> Callable[..., R]: ...   # arguments now unchecked
```

`ParamSpec` (PEP 612) fixes it:

```python
def logged[**P, R](fn: Callable[P, R]) -> Callable[P, R]:
    @functools.wraps(fn)
    def wrapper(*args: P.args, **kwargs: P.kwargs) -> R:
        return fn(*args, **kwargs)
    return wrapper
```

`Concatenate` for decorators that consume or inject a leading parameter:

```python
def with_conn[**P, R](
    fn: Callable[Concatenate[Connection, P], R]
) -> Callable[P, R]:
    @functools.wraps(fn)
    def wrapper(*a: P.args, **kw: P.kwargs) -> R:
        with get_connection() as conn:
            return fn(conn, *a, **kw)
    return wrapper
```

For a parameterized decorator, the factory returns the `Callable[[Callable[P, R]],
Callable[P, R]]`, and you need the type variables scoped to the *inner* function.

A decorator that changes the return type (e.g. wrapping in a `Result`) types as
`Callable[P, R] -> Callable[P, Result[R]]` — and this is where you discover whether your
decorator's semantics are actually expressible. If you cannot write the type, the decorator
is probably doing two unrelated things.

**Overloaded decorators** (shape 3 of §2.1) need overloads to type correctly:

```python
@overload
def retry[**P, R](fn: Callable[P, R], /) -> Callable[P, R]: ...
@overload
def retry[**P, R](*, attempts: int = 3) -> Callable[[Callable[P, R]], Callable[P, R]]: ...
def retry(fn=None, /, *, attempts=3): ...
```

### 2.7 Async, and the sync/async split

A decorator that wraps an `async def` with a *sync* wrapper returns a coroutine object
without awaiting it — which silently does nothing and produces a "coroutine was never
awaited" warning if you are lucky.

```python
def timed[**P, R](fn: Callable[P, R]) -> Callable[P, R]:
    if inspect.iscoroutinefunction(fn):
        @functools.wraps(fn)
        async def awrapper(*a: P.args, **kw: P.kwargs) -> R:
            t = time.perf_counter()
            try:
                return await fn(*a, **kw)
            finally:
                record(fn.__name__, time.perf_counter() - t)
        return awrapper                    # type: ignore[return-value]
    @functools.wraps(fn)
    def wrapper(*a: P.args, **kw: P.kwargs) -> R:
        ...
    return wrapper
```

The `type: ignore` is unavoidable with current typing: expressing "returns an async function
iff given one" requires overloads on the *awaitability* of `R`, which is expressible but
verbose. Write the overloads if the decorator is public API; use the ignore with a comment
if it is internal.

Note also: `iscoroutinefunction` must see through `functools.wraps` — since 3.8 it follows
`__wrapped__`, but a decorator that does not set it will fool the check. Another reason to
always use `wraps`.

Async generators, context managers, and `functools.partial` objects each need their own
detection if you intend to support them. Decide the supported set and *document it*; the
alternative is a decorator that fails mysteriously on one of them.

### 2.8 When not to use a decorator

Decorators are attractive because they are invisible at the call site. That is also their
main cost:

- **They hide control flow.** A reader of the call site sees no indication that a retry, a
  transaction, or a cache is involved. In a debugger you step into a wrapper.
- **They are hard to configure per call site.** `@retry(attempts=3)` is fixed at import.
  Making it dynamic means reading global config from inside the wrapper, which is worse.
- **They are hard to test in isolation** — you must decorate something to exercise them.
- **They stack invisibly.** Four decorators on a function is four hidden behaviours in an
  order the reader must reconstruct.

The alternatives worth considering each time: an explicit call
(`with transaction(): ...`), a context manager, a wrapper object, or a middleware/pipeline
where the composition is written out in one place. That last one — an explicit list of
middlewares applied at the composition root — is usually better than decorators for
application-wide concerns precisely because the order is visible and configurable.

**Rule of thumb:** a decorator is right when the behaviour is (a) genuinely uniform across
all uses, (b) not something the reader of the call site needs to know about, and (c) not
something you will want to vary per environment. Fail any of the three and consider an
explicit form.

## 3. Construction: an observability decorator

Build one decorator, properly, hitting every concern above.

**Requirements:** record duration and outcome; work on sync and async functions and on
methods; preserve signature and typing; allow per-function naming; be introspectable
(expose stats); be thread-safe; add negligible overhead when disabled.

**Pass 1 — the naive version**, and count its defects:

```python
def observed(fn):
    def wrapper(*a, **kw):
        t = time.time()
        r = fn(*a, **kw)
        print(fn.__name__, time.time() - t)
        return r
    return wrapper
```

Defects: no `wraps`; `time.time` is wall clock and can go backwards (use
`time.perf_counter`); no exception path, so failures are unmeasured; `print` instead of a
metrics sink; no async support; no way to turn it off; not thread-safe if it accumulated
state.

**Pass 2 — correct core:**

```python
def observed[**P, R](
    fn: Callable[P, R] | None = None, /, *, name: str | None = None
) -> Any:
    def deco(fn: Callable[P, R]) -> Callable[P, R]:
        metric = name or f"{fn.__module__}.{fn.__qualname__}"

        def record(t0: float, outcome: str) -> None:
            SINK.observe(metric, time.perf_counter() - t0, outcome)

        if inspect.iscoroutinefunction(fn):
            @functools.wraps(fn)
            async def awrapper(*a: P.args, **kw: P.kwargs) -> R:
                t0 = time.perf_counter()
                try:
                    result = await fn(*a, **kw)
                except BaseException as e:
                    record(t0, type(e).__name__)
                    raise
                record(t0, "ok")
                return result
            wrapper: Any = awrapper
        else:
            @functools.wraps(fn)
            def swrapper(*a: P.args, **kw: P.kwargs) -> R:
                t0 = time.perf_counter()
                try:
                    result = fn(*a, **kw)
                except BaseException as e:
                    record(t0, type(e).__name__)
                    raise
                record(t0, "ok")
                return result
            wrapper = swrapper

        wrapper.metric_name = metric        # introspectable
        return wrapper
    return deco if fn is None else deco(fn)
```

Points worth noticing:

- `except BaseException` here is correct — we re-raise immediately and we want cancellation
  and `KeyboardInterrupt` recorded (PY-501 L10 §2.1). This is one of the few legitimate uses.
- `record` is called *outside* the `try` on the success path so that an exception from the
  sink does not get recorded as a failure of `fn`.
- `qualname` not `name`, so methods are distinguishable.
- The duplication between the two wrappers is real and ugly. Factoring it out is exercise
  C1 and is harder than it looks, because the `await` is inside the `try`.

**Pass 3 — the questions the code cannot answer.** Write these down:

- **What happens on a generator function?** `fn(*a)` returns a generator immediately;
  you measure construction, not consumption. Detect it (`isasyncgenfunction`,
  `isgeneratorfunction`) and either wrap the iteration or refuse to decorate.
- **What is the overhead when the sink is a no-op?** Measure it (PY-602 L01). If a
  decorator costs 300 ns and is applied to a function called 10⁷ times a second, it is not
  free.
- **How do you disable it in production without redeploying?** A module-level flag checked
  in the wrapper costs a `LOAD_GLOBAL` per call; a rebinding of the wrapper at configuration
  time costs nothing per call but cannot change later.
- **Does it belong here at all?** For HTTP handlers, the framework's middleware already does
  this, once, in one place, with correct ordering. §2.8's argument.

## 4. Failure modes

- **No `functools.wraps`.** §2.2. Breaks introspection and frameworks.
- **Wrong `classmethod`/`property` order.** §2.3.
- **`lru_cache` on a method.** Retains every instance forever.
- **Sync wrapper on an async function.** Returns an unawaited coroutine.
- **Untyped decorator.** Erases the signature of everything it touches.
- **Mutable shared state in a closure without a lock.** §2.5.
- **Undocumented ordering requirements.** §2.4.
- **Decorating a generator function and measuring nothing.** §3 pass 3.
- **A decorator that swallows exceptions.** Anything with a bare `except` inside a wrapper
  is a hidden `except Exception: pass` applied to many functions at once.
- **Configuration frozen at import time** when it needed to vary.
- **Four decorators on one function.** Consider middleware.

## 5. Exercises

### Warm-up (25 min)

**W1.** Write a decorator without `wraps` and show four distinct things that break.

**W2.** Show that `@retry` above `@classmethod` fails and explain the error precisely.

**W3.** Demonstrate the unawaited-coroutine failure and the warning it produces.

### Core (2.5 h)

**C1 — The observability decorator.** Complete §3, all three passes, including factoring out
the sync/async duplication (there are at least three approaches; try two and compare).
Deliverable: the code, the overhead measurement, generator handling, the disable mechanism,
and a 400-word note on whether it should exist given §2.8.

**C2 — Order enforcement.** Write two decorators with a mandatory relative order. Then
implement an import-time check, using `__wrapped__` traversal, that raises if they are
stacked wrongly. Test both orders. Report what your check cannot detect.

**C3 — Fully typed decorator suite.** Build four decorators — plain, parameterized,
signature-changing (injects a leading argument via `Concatenate`), and
optionally-parameterized with overloads. All must pass `mypy --strict` and `pyright`, and
you must demonstrate at a call site that the checker knows the decorated function's
signature. Report any place the two checkers disagree.

**C4 — Decorator or middleware?** Take a cross-cutting concern in a real system currently
implemented as a decorator. Reimplement it as an explicit middleware pipeline assembled at
the composition root. Compare on: visibility of ordering, per-environment configurability,
testability, and reading a call site. Recommend one, with reasons.

### Challenge

**X1.** Build a decorator that works correctly on: functions, methods, classmethods,
staticmethods, properties, coroutine functions, generator functions, async generator
functions, and `functools.partial` objects. For each unsupported case, produce a clear error
at decoration time rather than a confusing one at call time. Document what you learned about
which of these are actually distinguishable.

**X2.** Measure decorator overhead precisely (after PY-602 L01): empty decorator, `wraps`-ed
decorator, decorator with a `try/except`, decorator with a global flag check, and a stack of
four. Report ns/call with medians and spread, on two Python versions. Explain the results
using PY-501 L08's cost model.

## 6. Self-check

1. What exactly does `functools.wraps` copy, and what is `__wrapped__` for?
2. Why does a decorator on a method receive `self` as an ordinary argument?
3. Why must `classmethod` be outermost?
4. Give three decorator pairs where order changes the semantics.
5. Name the three places decorator state can live, and the lifetime of each.
6. What does `ParamSpec` fix, and what does `Concatenate` add?
7. Why does a sync wrapper break an async function, and how do you detect one?
8. Give the three-part rule for when a decorator is the right tool.

## 7. Primary sources

- PEP 318 (decorators), PEP 612 (`ParamSpec`), PEP 3129 (class decorators).
- `functools` source — `wraps`, `lru_cache`, `singledispatch`, `cached_property`.
- Ramalho, *Fluent Python* 2e, ch. 9.
- Graham Dumpleton, "How you implemented your Python decorator is wrong" (the `wrapt`
  series). Long, opinionated, and the most thorough treatment of the edge cases in
  existence.

---

**Previous:** [L03](L03-class-construction-and-metaclasses.md) · **Next:**
[L05 — Generators and Lazy Evaluation](L05-generators-and-lazy-evaluation.md)
