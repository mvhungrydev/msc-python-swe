# PY-501 · Lesson 10 — Exceptions, Control Flow, and Cleanup Semantics

**Estimated study time:** 3.5 hours
**Prerequisites:** L06, L08, L09

---

## 1. Orientation

```python
def f():
    try:
        return "try"
    finally:
        return "finally"

f()      # "finally"
```

The `try` block's `return` computed its value, the `finally` ran, and its `return`
*replaced* the pending one. If the `try` had raised instead, the `finally`'s `return` would
have **swallowed the exception silently**.

Exceptions in Python are not an error-reporting afterthought bolted onto a language; they
are a general control-flow mechanism used by iteration (`StopIteration`), generators
(`GeneratorExit`), the import system, and `asyncio` cancellation. Understanding their exact
semantics — especially around cleanup — is what separates code that fails safely from code
that fails silently.

## 2. Theory

### 2.1 The hierarchy

```
BaseException
├── BaseExceptionGroup
├── GeneratorExit
├── KeyboardInterrupt
├── SystemExit
└── Exception
    ├── ArithmeticError, LookupError, OSError, ValueError, TypeError, ...
    └── ExceptionGroup  (also inherits BaseExceptionGroup)
```

The split is deliberate: `except Exception:` deliberately does **not** catch
`KeyboardInterrupt`, `SystemExit`, or `GeneratorExit`, because catching those breaks
Ctrl-C, `sys.exit()`, and generator cleanup respectively.

**Rules that follow:**

- Catch `Exception`, never `BaseException`, unless you are writing a top-level supervisor
  and you re-raise.
- A bare `except:` is `except BaseException:`. It is essentially always a bug.
- Custom exceptions inherit from `Exception`, and from a *domain base class* of your own:
  `class PaymentError(Exception)` then `class CardDeclined(PaymentError)`. Callers can then
  catch at the granularity they need, and you can add new subclasses without breaking them.
- Prefer existing built-ins where they fit exactly (`ValueError` for a bad value,
  `TypeError` for a wrong type, `LookupError`/`KeyError` for a missing key). Inventing
  `MyValueError` for a plain bad value costs callers an import and buys nothing.

### 2.2 The `try` statement in full

```python
try:
    A
except E1 as e:
    B
except E2:
    C
else:
    D
finally:
    F
```

- `A` runs. If it completes normally, `D` (`else`) runs.
- If `A` raises, handlers are tried **in source order**, matching by `isinstance`. First
  match wins. This is why `except Exception` before `except ValueError` makes the second
  handler dead code.
- `F` (`finally`) runs on *every* exit path: normal completion, handled exception,
  unhandled exception, `return`, `break`, `continue`.

The `else` clause is under-used and worth the habit. Its purpose is to keep the `try` block
minimal:

```python
try:
    value = d[key]
except KeyError:
    handle_missing()
else:
    process(value)         # NOT inside try — so a KeyError from process() isn't caught here
```

Without `else`, `process(value)` inside the `try` would have its own `KeyError`s captured by
your handler — a genuine and hard-to-find bug class.

### 2.3 `finally` and the pending-action rule

The precise semantics: when control leaves the `try` (or `except`) block by `return`,
`break`, `continue`, or an exception, that action is **saved as pending**, `finally` runs,
and then the pending action resumes — *unless* `finally` itself performs a `return`,
`break`, `continue`, or raises, in which case the pending action is **discarded**.

Consequences:

- `return` in `finally` replaces the return value **and swallows any in-flight exception**.
  Never write it. `ruff`'s `B012` and pylint's `lost-exception` flag it.
- `break`/`continue` in a `finally` inside a loop does the same.
- An exception raised in `finally` replaces the original — the original becomes the new
  one's `__context__` (§2.4), so it is not lost, but it is now buried.
- The return *value* is computed before `finally` runs:

```python
def g():
    x = [1]
    try:
        return x
    finally:
        x.append(2)     # mutation is visible: returns [1, 2]
```

because the *reference* was captured, and the list was mutated, not rebound. Change
`x.append(2)` to `x = [3]` and the return is still `[1]`. This is L01's label model again.

### 2.4 Chaining: `__context__`, `__cause__`, `__suppress_context__`

Raising inside a handler chains automatically:

```python
try:
    1 / 0
except ZeroDivisionError:
    raise ValueError("bad input")
```

The traceback shows both, joined by *"During handling of the above exception, another
exception occurred"*. The `ZeroDivisionError` is on `e.__context__` — implicit chaining.

`raise X from Y` sets `e.__cause__ = Y` and prints *"The above exception was the direct
cause of the following exception"* — explicit chaining, a stronger claim.

`raise X from None` sets `__suppress_context__ = True`, hiding the chain.

**Guidance:**

- When translating an exception across an abstraction boundary (a `KeyError` from your
  storage layer becoming a `NotFound` in your domain layer), use `from e`. The cause is
  genuinely the cause, and the operator debugging at 3 a.m. needs the original.
- Use `from None` only when the inner exception is a pure implementation detail that would
  mislead — e.g. a `ValueError` from `int()` inside a parser, where you raise a
  domain-appropriate parse error with position information.
- Never lose the original silently. `except E: raise F` at least keeps `__context__`;
  `except E: pass` followed by a later `raise F` does not.

Re-raising the *same* exception preserves everything:

```python
except SomeError:
    log.exception("failed")
    raise                # bare raise — preserves traceback, does not add a frame
```

`raise e` (naming it) appends a frame at the re-raise point, which is usually noise.

### 2.5 Exception groups (3.11+)

PEP 654 introduced `ExceptionGroup` and `except*` for the case where multiple independent
things fail concurrently — the natural situation in structured concurrency (PY-601 L06).

```python
try:
    raise ExceptionGroup("fanout", [ValueError("a"), TypeError("b"), ValueError("c")])
except* ValueError as eg:
    print("values:", eg.exceptions)      # a group containing a and c
except* TypeError as eg:
    print("types:", eg.exceptions)       # a group containing b
```

Semantics differ from `except` in ways you must internalize:

- **Every matching `except*` clause runs**, not just the first. The group is split by type.
- The bound name is always an `ExceptionGroup`, never a bare exception.
- Unmatched sub-exceptions propagate as a residual group.
- `except*` and `except` cannot be mixed in one `try`.
- You cannot `except* ExceptionGroup` — it is a `TypeError`, because it would be ambiguous.

`BaseExceptionGroup.split(condition)` and `.subgroup(condition)` let you filter
programmatically, which is how libraries handle groups without `except*`.

Use them when concurrency genuinely produces independent failures. Do not wrap single
exceptions in groups "for consistency" — you make every caller handle a group for no
benefit.

### 2.6 Exceptions as control flow

Idiomatic Python uses exceptions for expected, non-error conditions:

- `StopIteration` terminates iteration. `next(it)` raises it; `for` catches it. Since PEP
  479 (3.7), a `StopIteration` that escapes a generator body is converted to
  `RuntimeError` — because the old behaviour silently truncated the generator, an
  outstandingly confusing bug.
- `GeneratorExit` is thrown into a generator when it is closed, so `finally` blocks run.
  Catching it and *not* re-raising (or yielding again) raises `RuntimeError`.
- `KeyError`/`IndexError` in `EAFP` style: `try: d[k] except KeyError:` is idiomatic and,
  since 3.11's zero-cost model, fast on the success path (L08 §2.5).
- `asyncio.CancelledError` (which inherits `BaseException` since 3.8, deliberately, so that
  `except Exception` does not eat cancellation) — PY-601 L07.

**The design line:** exceptions are appropriate for conditions the *immediate* caller
usually cannot handle inline, and for terminating a nested computation. They are
inappropriate as a routine return channel for a value the caller always inspects — that is
what a return value, an `Optional`, or a result type is for. Raising and catching within
three lines is a smell.

### 2.7 Writing exceptions worth catching

Four properties of a good exception type:

1. **A useful hierarchy.** A package base class; specific subclasses under it.
2. **Structured data, not just a message.** `raise QuotaExceeded(limit=1000, used=1043,
   resource="api_calls")` lets callers *act*. A formatted string forces them to parse it.
3. **A message that says what was expected, what was found, and where.** Compare
   `"invalid config"` with `"config key 'timeout': expected int, got str 'thirty' (line 14)"`.
4. **`__str__`/`__repr__` that survive logging.** And be careful: exceptions are pickled
   when crossing process boundaries (`multiprocessing`, `concurrent.futures`), and
   **pickle reconstructs by calling the class with `args`** — so a custom `__init__` whose
   signature does not match `self.args` will fail to unpickle. Either accept a single
   message argument and stash structured data as attributes set after `super().__init__`,
   or implement `__reduce__`.

```python
class QuotaExceeded(Exception):
    def __init__(self, resource: str, limit: int, used: int) -> None:
        super().__init__(f"{resource}: used {used} of {limit}")
        self.resource, self.limit, self.used = resource, limit, used

    def __reduce__(self):
        return (type(self), (self.resource, self.limit, self.used))
```

Test that your exceptions round-trip through `pickle`. Almost nobody does, and it fails in
production the first time a worker process raises.

### 2.8 Cleanup guarantees, ranked

What actually runs, from strongest to weakest:

1. **`finally` / context manager `__exit__`** — on any normal control-flow exit, including
   exceptions. Not on `os._exit()`, `SIGKILL`, or a segfault.
2. **`contextlib.ExitStack`** — same guarantee, dynamically composed. Correct unwinding
   order (reverse of entry) is guaranteed.
3. **`atexit` handlers** — on normal interpreter shutdown. Not on signals, not on
   `os._exit`.
4. **`weakref.finalize`** — when the object dies, and at interpreter exit. Ordering among
   finalizers is not specified.
5. **`__del__`** — best effort. See L09 §2.5.

For a process that must clean up on termination, none of these is sufficient on their own:
you need a signal handler that converts `SIGTERM` into an ordinary exception or a shutdown
flag, so that `finally` blocks get to run. That pattern — signal → graceful shutdown →
`finally` — is the operational core of every well-behaved service, and CA-731 L05 builds it
properly.

## 3. Construction: a retry decorator that does not lie

A retry decorator is a small program that touches every idea in this lesson.

**Version 1 — the one everyone writes first.**

```python
def retry(n):
    def deco(fn):
        def wrapper(*a, **kw):
            for _ in range(n):
                try:
                    return fn(*a, **kw)
                except Exception:
                    pass
        return wrapper
    return deco
```

Six defects. Name them before reading on.

1. On total failure it returns `None` — a silent wrong answer, the worst failure mode
   available.
2. `except Exception` retries everything, including `TypeError` from a programming error
   and `ValueError` from bad input, neither of which will ever succeed.
3. No backoff — retries hammer a struggling dependency, which is how a partial outage
   becomes a total one.
4. Loses the exception chain: nothing records that four attempts failed differently.
5. Not `functools.wraps`-ed: `__name__`, `__doc__`, `__wrapped__`, and the signature are
   destroyed, breaking introspection, `inspect.signature`, and every framework that reads
   them.
6. Retries non-idempotent operations blindly. This is a *semantic* defect the decorator
   cannot fix alone — it must be documented as a precondition.

**Version 2.**

```python
import functools, random, time
from collections.abc import Callable
from typing import ParamSpec, TypeVar

P = ParamSpec("P")
R = TypeVar("R")

def retry(
    attempts: int = 3,
    *,
    on: tuple[type[BaseException], ...] = (OSError,),
    base_delay: float = 0.1,
    max_delay: float = 5.0,
    jitter: bool = True,
) -> Callable[[Callable[P, R]], Callable[P, R]]:
    if attempts < 1:
        raise ValueError("attempts must be >= 1")

    def deco(fn: Callable[P, R]) -> Callable[P, R]:
        @functools.wraps(fn)
        def wrapper(*a: P.args, **kw: P.kwargs) -> R:
            failures: list[BaseException] = []
            for i in range(attempts):
                try:
                    return fn(*a, **kw)
                except on as e:
                    failures.append(e)
                    if i == attempts - 1:
                        break
                    delay = min(max_delay, base_delay * 2**i)
                    if jitter:
                        delay *= random.uniform(0.5, 1.5)
                    time.sleep(delay)
            raise ExceptionGroup(
                f"{fn.__name__} failed after {attempts} attempts", failures
            ) from failures[-1]
        return wrapper
    return deco
```

What changed and why:

- **Narrow exception set,** supplied by the caller. Default `OSError` covers network and
  file transience.
- **Exponential backoff with jitter.** Jitter is not decoration: without it, N clients that
  failed together retry together, producing a thundering herd. Full jitter
  (`uniform(0, delay)`) is the AWS-recommended variant; the symmetric version above is a
  reasonable compromise. Read the AWS Architecture Blog piece on exponential backoff and
  jitter and decide for yourself — that decision is exercise C1.
- **All failures preserved** in an `ExceptionGroup`, with the last as `__cause__`. An
  operator can see whether the four attempts failed the same way (a real outage) or
  differently (something stranger).
- **`functools.wraps`** so introspection survives.
- **`ParamSpec`** so the type checker knows the decorated function's signature is unchanged
  (SE-511 L08).

Still missing, deliberately: a deadline (attempts is the wrong budget — wall-clock is),
`async` support, circuit breaking, and observability. Those are C1 and C2.

## 4. Failure modes

- **`return` in `finally`.** Swallows exceptions.
- **`except Exception: pass`.** The single most damaging idiom in Python. If you truly mean
  to ignore, use `contextlib.suppress(SpecificError)` — it names what is being ignored and
  cannot accidentally cover more code than intended.
- **Bare `except:`.** Catches `KeyboardInterrupt`, so Ctrl-C stops working.
- **Broad handler ordering.** `except Exception` before a specific handler makes the
  specific one dead.
- **Too much inside `try`.** Use `else`.
- **Catching and re-raising a new exception without `from`.** Not fatal (`__context__` is
  kept) but a lost opportunity to state the causal claim.
- **`raise e` instead of bare `raise`.** Adds noise to the traceback.
- **Retaining exceptions.** L09 §2.3.
- **Custom exceptions that do not pickle.** §2.7.
- **Retrying non-idempotent operations.** A duplicate payment is worse than a failed one.
- **Using exceptions for expected, immediately-handled control flow.** §2.6.
- **`except*` for a single exception.** Complexity with no payoff.

## 5. Exercises

### Warm-up (25 min)

**W1.** Write functions demonstrating each of: `finally` overriding a return, `finally`
swallowing an exception, `finally` mutating a returned object, and `finally` running on
`break`.

**W2.** Show the difference in traceback output between `raise X`, `raise X from e`, and
`raise X from None` inside a handler.

**W3.** Write a generator whose body raises `StopIteration` and show the `RuntimeError`
PEP 479 produces. Explain what the pre-3.7 behaviour would have been and why it was worse.

### Core (2.5 h)

**C1 — The complete retry.** Extend §3 version 2 with: a wall-clock deadline, a
`should_retry(exc) -> bool` predicate, structured logging of each attempt, a `sync` and
`async` variant sharing one implementation, and optional integration with a circuit
breaker. Write tests for: success first try, success on retry, exhaustion, a
non-retryable exception passing straight through, deadline exceeded mid-backoff, and
cancellation. Then write 400 words comparing your jitter strategy to full jitter and
decorrelated jitter, citing the AWS piece.

**C2 — An exception design review.** Take the exception hierarchy of a library you use
(`requests`, `httpx`, `boto3`, `sqlalchemy`). Assess against §2.7's four properties. Write
up: what a caller can and cannot do programmatically, and one concrete improvement. Then
test whether its exceptions round-trip through `pickle`.

**C3 — Cleanup under signals.** Build a worker that holds a resource, and make it clean up
correctly on: normal completion, an exception, `SIGINT`, `SIGTERM`, and
`KeyboardInterrupt` during the cleanup itself. Verify each with a test. Report which of the
five levels of §2.8 you needed and why. Then explain what happens on `SIGKILL` and what
your design does about it (the correct answer involves idempotence, not signal handling).

**C4 — Exception groups.** Build a fan-out function that calls ten services concurrently
(threads are fine here) and aggregates failures into an `ExceptionGroup`. Write a caller
that handles `TimeoutError` and `PermissionError` separately with `except*` and lets
everything else propagate. Then write the same caller *without* `except*`, using
`.split()`. Compare readability and say which you would put in a library API and why.

### Challenge

**X1.** Read PEP 654 in full. Implement `ExceptionGroup.split` yourself, matching CPython's
semantics for: nested groups, traceback preservation on both halves, and the `__cause__`
of the residual. Test against the real implementation on 30 constructed cases.

**X2.** Instrument a real codebase to find every `except` clause and classify each as:
handles-and-recovers, translates, logs-and-reraises, or swallows. Report the distribution.
For the swallows, determine how many are defensible. Write 500 words on what the
distribution tells you about the codebase's error-handling maturity — this is a technique
you can reuse in any code review.

## 6. Self-check

1. Why does `except Exception` not catch `KeyboardInterrupt`?
2. State the pending-action rule for `finally`.
3. What is `__context__` vs `__cause__`, and what does `from None` do?
4. What is the `else` clause of `try` for, and what bug does it prevent?
5. Give three ways `except*` differs from `except`.
6. Why was PEP 479 (StopIteration in generators) necessary?
7. Give the five cleanup mechanisms in order of strength, and what defeats each.
8. Name four defects in the naive retry decorator.

## 7. Primary sources

- Language Reference §8.4 (`try`) and §8.4.2 (`except*`). Tutorial §8 is also unusually
  good here.
- PEP 3134 (chaining), PEP 479 (StopIteration), PEP 654 (exception groups — read the
  Rationale in full), PEP 678 (`add_note`).
- Marc Brooker / AWS Architecture Blog, "Exponential Backoff And Jitter" (2015).
- Nygard, *Release It!*, chapters on circuit breakers and timeouts.

---

**Previous:** [L09](L09-memory-refcounting-gc-finalization.md) ·
**Course complete.** Next: [problem sets](problem-sets.md) and [exam](exam.md), then
[SE-511](../SE-511-software-construction/syllabus.md).
