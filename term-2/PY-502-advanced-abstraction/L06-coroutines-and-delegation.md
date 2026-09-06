# PY-502 · Lesson 06 — Coroutines, `yield from`, and Delegation

**Estimated study time:** 3.5 hours
**Prerequisites:** L05

---

## 1. Orientation

`yield from` looks like a convenience for flattening iteration:

```python
def chain(a, b):
    yield from a
    yield from b
```

It is that, and it is also considerably more: a **bidirectional delegation channel** that
forwards `send`, `throw`, and `close` to a sub-generator and propagates its return value.
That machinery is what made `asyncio` possible — `async`/`await` is, at the interpreter
level, a specialization of exactly this mechanism, and understanding it here makes PY-601's
async lessons ordinary rather than magical.

## 2. Theory

### 2.1 What `yield from` actually does

`RESULT = yield from EXPR` is defined (PEP 380) to be roughly equivalent to:

```python
_i = iter(EXPR)
try:
    _y = next(_i)
except StopIteration as _e:
    RESULT = _e.value
else:
    while True:
        try:
            _s = yield _y                      # forward the yielded value out
        except GeneratorExit:
            _i.close(); raise                  # forward close
        except BaseException as _e:
            if hasattr(_i, "throw"):
                _y = _i.throw(_e)              # forward throw
            else:
                raise
        else:
            try:
                _y = _i.send(_s) if _s is not None else next(_i)
            except StopIteration as _e:
                RESULT = _e.value              # capture the return value
                break
```

Four things it does that a `for` loop does not:

1. **Forwards `send()`** into the sub-generator.
2. **Forwards `throw()`** into the sub-generator, giving it a chance to handle.
3. **Forwards `close()`**, so the sub-generator's `finally` blocks run.
4. **Captures the sub-generator's `return` value** as the value of the expression.

`for x in sub: yield x` does *none* of these. It is a strictly weaker construct, and if
anything sends, throws, or returns, the two are not interchangeable.

Efficiency matters too: `yield from` short-circuits the delegation chain in CPython, so a
deep chain does not pay a frame resume per level per element.

### 2.2 Generators as coroutines: the send protocol

```python
def averager() -> Generator[float | None, float, float]:
    total = count = 0.0
    average = None
    try:
        while True:
            value = yield average          # send in a value, get the running average out
            total += value
            count += 1
            average = total / count
    except GeneratorExit:
        return average                     # the final result
```

Usage:

```python
avg = averager()
next(avg)              # prime: advance to the first yield
avg.send(10)           # 10.0
avg.send(20)           # 15.0
```

The typing is `Generator[YieldType, SendType, ReturnType]` — three parameters, in that
order, and getting them the wrong way round is a common error. For a plain iteration
generator, `Iterator[T]` is the right annotation and `Generator[T, None, None]` is the
verbose equivalent.

**Priming** is mandatory: a generator suspended before its first `yield` has no `yield`
expression to receive a value, so `send(x)` with `x is not None` raises `TypeError`. A
priming decorator is the standard fix and is worth writing once:

```python
def primed[**P, T, S, R](fn: Callable[P, Generator[T, S, R]]) -> Callable[P, Generator[T, S, R]]:
    @functools.wraps(fn)
    def start(*a: P.args, **kw: P.kwargs) -> Generator[T, S, R]:
        g = fn(*a, **kw)
        next(g)
        return g
    return start
```

This style — generators as stateful consumers driven by `send` — was the dominant Python
concurrency idiom between PEP 342 (2005) and PEP 492 (2015). You will meet it in older
code, in `contextlib`, and in libraries that implement their own schedulers. It is largely
superseded by `async`/`await` for I/O, but remains a clean way to express **incremental
parsers, state machines, and accumulators** with the state held in local variables rather
than in an object's attributes.

### 2.3 Delegation and the return value

```python
def read_header(f) -> Generator[None, str, Header]:
    fields = {}
    while True:
        line = yield
        if not line.strip():
            return Header(fields)          # StopIteration.value
        k, _, v = line.partition(":")
        fields[k.strip()] = v.strip()

def read_message(f) -> Generator[None, str, Message]:
    header = yield from read_header(f)     # captures the returned Header
    body = yield from read_body(header)
    return Message(header, body)
```

This is the incremental-parser pattern, and it is genuinely elegant: each sub-parser is an
independent, testable generator; composition is `yield from`; state is in local variables and
the call stack rather than in an explicit state machine.

Compare with the alternatives — an explicit state enum with a big `match`, or a
callback-based push parser — on: readability, testability of each state, and what happens
when you need to add a state. The generator version usually wins on the first two and is
harder to introspect (you cannot easily ask "what state am I in?").

### 2.4 From `yield from` to `await`

The history matters because it explains the semantics.

- **PEP 342 (2005)** — generators gain `send`/`throw`/`close`. Coroutines become possible.
- **PEP 380 (2012)** — `yield from`. Delegation becomes composable.
- **`asyncio` (2014, PEP 3156)** — a scheduler built on generator coroutines:
  `@asyncio.coroutine` + `yield from future`. The event loop `send`s results back in.
- **PEP 492 (2015)** — `async def` / `await`: dedicated syntax and a separate
  *coroutine object* type, so that a coroutine cannot be accidentally iterated and an
  ordinary generator cannot be accidentally awaited.

The mechanism is the same. `await x` is, at the interpreter level, close to
`yield from x.__await__()`. An `async def` function compiles to a code object with
`CO_COROUTINE` instead of `CO_GENERATOR`; the object it returns has `send`, `throw`, and
`close`, and the event loop drives it with exactly those calls.

Two things follow, and they are the practical payoff of this lesson:

1. **`await` suspends the coroutine and returns control to whoever is driving it.** There
   is no thread, no preemption, no magic. The event loop is a `while` loop calling `send`.
2. **`asyncio.CancelledError` is a `throw` into a suspended coroutine** at its current
   `await` point — precisely the `GeneratorExit` mechanism of L05 §2.4, generalized. Which
   is why cancellation semantics are exactly the semantics of an exception raised at an
   arbitrary suspension point, and why `finally` blocks are how you clean up
   (PY-601 L07).

If you can build a toy event loop from generators — exercise X1 — `asyncio` stops being a
framework you use and becomes a library you understand.

### 2.5 `contextlib.contextmanager`, explained

```python
@contextlib.contextmanager
def transaction(conn):
    tx = conn.begin()
    try:
        yield tx
    except BaseException:
        tx.rollback()
        raise
    else:
        tx.commit()
```

The decorator wraps the generator in a `_GeneratorContextManager` whose:

- `__enter__` calls `next(gen)` and returns the yielded value.
- `__exit__(typ, val, tb)` calls `gen.throw(val)` if there was an exception, else
  `next(gen)` expecting `StopIteration`.

So: an exception in the body is *thrown into the generator at the `yield`*. That is why a
`try/finally` around the `yield` is mandatory for guaranteed cleanup, and why a bare
`yield` with no `try` leaks on exception.

It also explains a subtle rule: **`__exit__` must return truthy to suppress an exception**
(PY-501 L06 §2.4), and `_GeneratorContextManager` returns truthy exactly when the generator
*swallows* the thrown exception and yields or returns. So a `@contextmanager` that catches
an exception and does not re-raise **silently suppresses it** — the same footgun, one level
removed.

L07 goes further into resource composition.

### 2.6 Where this pattern is still the right answer

Given `async`/`await` exists, when do you still use `send`-driven generators?

- **Incremental parsers and protocol decoders.** `h11` (the HTTP/1.1 state machine) is the
  canonical example, and the "sans-I/O" design philosophy it embodies is worth studying:
  the protocol logic is a pure state machine driven by `send`, with *no* I/O at all, so it
  can be used from sync code, async code, or a test with no network.
- **Cooperative state machines** where the state is naturally the program counter.
- **Building your own scheduler** — a trampoline, a simulation, a deterministic test
  harness for concurrent logic.
- **`contextlib.contextmanager`** — used constantly, and now you know why it works.
- **Streaming accumulators** where the accumulator's state is complex.

And where it is not: anything doing I/O concurrency in new code. Use `async`/`await`.

### 2.7 Typing

```python
from collections.abc import Generator, Iterator, Coroutine, AsyncIterator

def count_up(n: int) -> Iterator[int]: ...                       # yield only
def averager() -> Generator[float | None, float, float]: ...     # yield, send, return
async def fetch(u: str) -> bytes: ...                            # Coroutine[Any, Any, bytes]
async def stream(u: str) -> AsyncIterator[bytes]: ...            # async generator
```

- Annotate a plain iteration generator's return as `Iterator[T]`, not `Generator[T, None,
  None]` — shorter and states the intent.
- An `async def` function's declared return type is the type it *returns*; the coroutine
  wrapper is implicit. An `async def` containing `yield` is an **async generator** and its
  return type is `AsyncIterator[T]` — a common confusion, because the two look similar and
  behave very differently (an async generator cannot `return` a value).
- `Generator`'s three parameters are the classic ordering error; PEP 696's defaults make
  `Generator[int]` legal on newer versions, which helps.

## 3. Construction: a sans-I/O protocol parser

Build a small line-oriented protocol (think Redis's RESP or a simplified SMTP) as a pure
state machine driven by `send`, with no I/O.

**Step 1 — the contract.** The parser accepts `bytes` chunks and produces events. It never
reads a socket. Its interface:

```python
class Parser:
    def feed(self, data: bytes) -> None: ...
    def events(self) -> Iterator[Event]: ...
```

**Step 2 — the generator core.**

```python
def _parse() -> Generator[Event | None, bytes, None]:
    buf = bytearray()
    while True:
        while b"\r\n" not in buf:
            buf += yield None                    # ask for more data
        line, _, rest = buf.partition(b"\r\n")
        buf = bytearray(rest)
        yield parse_line(line)
```

Immediately notice a design problem: this yields one event per `send`, but one chunk may
contain many complete messages, and one message may span many chunks. The generator's
yield/send rhythm is a *protocol between the parser and its driver* and it must be designed,
not stumbled into. Two workable rhythms:

- **Yield `None` for "need more data", an `Event` otherwise; the driver loops.**
- **Yield a list of events per send.** Simpler driver, less incremental.

Choose, and write down why. This is the exercise.

**Step 3 — composition with `yield from`.** Split into `_parse_header` and `_parse_body`
generators, composed with `yield from`, with the header parser *returning* the parsed header
so the body parser can use the content length. Note that this is only possible because
`yield from` propagates the return value.

**Step 4 — drive it three ways.** The payoff of sans-I/O:

```python
# synchronous
with socket_conn as s:
    while chunk := s.recv(4096):
        for ev in parser.feed_and_events(chunk): handle(ev)

# asyncio
async for chunk in stream:
    for ev in parser.feed_and_events(chunk): await handle(ev)

# test — no I/O at all
parser.feed(b"HEAD"); parser.feed(b"ER\r\nbo"); parser.feed(b"dy\r\n")
assert list(parser.events()) == [...]
```

The test is the point. A protocol implementation with I/O inside it can only be tested with
I/O. This one can be tested with a list of byte strings — including the pathological
splits (one byte at a time, split in the middle of a delimiter) that find every real parser
bug.

**Step 5 — property test it** (SE-511 L04): for any message and any random chunking of its
serialization, the parser produces the same events. This single property finds essentially
every buffering bug.

## 4. Failure modes

- **`for x in sub: yield x` where delegation is needed.** Drops `send`, `throw`, `close`,
  and the return value.
- **Forgetting to prime.** `TypeError: can't send non-None value to a just-started
  generator`.
- **`Generator` type parameters in the wrong order.**
- **`@contextmanager` without `try/finally`.** Cleanup skipped on exception.
- **`@contextmanager` that catches and does not re-raise.** Silently suppresses.
- **`@contextmanager` yielding more than once.** `RuntimeError: generator didn't stop`.
- **Confusing an async generator with a coroutine.** `AsyncIterator[T]` vs
  `Coroutine[..., T]`; the former cannot return a value.
- **Building a scheduler when `asyncio` exists.** Fine as an exercise, rarely in production.
- **I/O inside a protocol parser.** §3. Makes it untestable and unusable from the other
  concurrency model.

## 5. Exercises

### Warm-up (25 min)

**W1.** Write a sub-generator and a delegating generator. Demonstrate all four behaviours
`yield from` provides that a `for` loop does not.

**W2.** Write a `@contextmanager` without `try/finally` and show the leak. Then one that
swallows an exception and show the silent suppression.

**W3.** Write the priming decorator of §2.2 and show what breaks without it.

### Core (2.5 h)

**C1 — The sans-I/O parser.** Complete §3, all five steps. Deliverable: the parser, the
written rationale for the yield/send rhythm, all three drivers, and the property test with
random chunking (including one-byte chunks). Report every bug the property test found.

**C2 — A state machine, two ways.** Implement a non-trivial protocol state machine
(a retry/backoff state machine, a TCP-like handshake, a multi-step form flow) as (a) a
generator with `yield from` composition, and (b) an explicit state enum with a `match`
dispatcher. Compare on: lines, testability of an individual state, ability to answer "what
state am I in?", ability to serialize the state, and ease of adding a state. Recommend one
per use case.

**C3 — `contextlib` from scratch.** Implement `contextmanager` yourself, including
correct `__exit__` semantics for: normal exit, exception in the body, exception suppressed
by the generator, generator that yields twice, generator that raises a *different*
exception. Test against the standard library's behaviour on all five.

**C4 — Read `h11`.** Read the `h11` source and its "sans-I/O" documentation. Write 800 words
on: what its state machine looks like, how it avoids I/O, what it gains, and what it costs
the user of the library. Then find one thing in it you would design differently and argue
for it.

### Challenge

**X1.** Build a toy event loop from generators: a scheduler that runs multiple generator
"tasks", where a task yields a `Sleep(seconds)` or `Read(fd)` request and the loop resumes
it when ready. Implement `sleep`, `gather`, and cancellation. Then map every piece of your
implementation onto its `asyncio` counterpart. This exercise, done properly, removes all the
mystery from PY-601 L06–L07.

**X2.** Read PEP 380's specification section and implement `yield from` semantics *by hand*
(the expansion in §2.1) as an explicit loop. Then construct three programs where your
hand-expansion differs observably from real `yield from` and explain each difference.

## 6. Self-check

1. Give the four things `yield from` does that a `for` loop does not.
2. What is priming and why is it required?
3. Give the three type parameters of `Generator` in order.
4. Explain how `@contextmanager`'s `__exit__` works, and why `try/finally` is mandatory.
5. Relate `await` to `yield from` at the interpreter level.
6. Explain `CancelledError` in terms of the generator protocol.
7. What is "sans-I/O" design and what does it buy?
8. What is the difference between an async generator and a coroutine, in return type and in
   capability?

## 7. Primary sources

- PEP 380 (`yield from`) — read the *Formal Semantics* section and work through it.
- PEP 342 (coroutines via enhanced generators), PEP 492 (`async`/`await`), PEP 525
  (async generators).
- Cory Benfield, "Building Protocol Libraries The Right Way" (PyCon 2016) — the sans-I/O
  argument.
- `h11` source and documentation.
- Ramalho, *Fluent Python* 2e, ch. 17's classic-coroutine sections.

---

**Previous:** [L05](L05-generators-and-lazy-evaluation.md) · **Next:**
[L07 — Context Managers and Resource Composition](L07-context-managers-and-resources.md)
