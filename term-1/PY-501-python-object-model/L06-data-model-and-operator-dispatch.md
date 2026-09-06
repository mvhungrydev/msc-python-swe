# PY-501 · Lesson 06 — The Data Model: Protocols and Operator Dispatch

**Estimated study time:** 4 hours
**Prerequisites:** L02, L03, L05

---

## 1. Orientation

Python's object model is *protocol-oriented*. There is no `Iterable` interface you must
declare; there is a question the interpreter asks — "does the type have `__iter__`?" — and
a fallback if not. A type that answers the questions correctly is indistinguishable from a
built-in.

This is the source of Python's famous coherence: `len`, `for`, `in`, `[]`, `with`, `+`,
`f"{x}"`, `pickle`, unpacking, and pattern matching are all *the same mechanism* — a
special method looked up on the type. Learn the protocols and you can build types that feel
native, and you can predict how any unfamiliar type will behave.

This lesson is a survey with depth in the places that are subtle: binary operator dispatch,
the iterator/iterable distinction, and the sequence protocol's fallbacks.

## 2. Theory

### 2.1 The rule that governs all of it

From L03 §2.6, restated because everything here depends on it:

> **Implicit special-method invocations are looked up on the type, bypassing the instance
> `__dict__` and bypassing `__getattribute__`.**

`len(x)` → `type(x).__len__(x)`. `x + y` → the type's slots. This is why special methods
must be class attributes, why `MagicMock` exists, and why proxies must forward explicitly.

### 2.2 Binary operator dispatch

For `a + b`, the interpreter:

1. Let `A = type(a)`, `B = type(b)`.
2. **If `B` is a proper subclass of `A` and `B` overrides `__radd__`**, try
   `B.__radd__(b, a)` first. (The subclass-priority rule again, as in L05 §2.2.)
3. Otherwise try `A.__add__(a, b)`.
4. If it returns `NotImplemented`, try `B.__radd__(b, a)`.
5. If that also returns `NotImplemented`, raise
   `TypeError: unsupported operand type(s) for +: 'A' and 'B'`.

Step 4 is why you write `__radd__`: it lets *your* type interoperate with a type that
existed first and knows nothing about you. `3 * Vector(...)` works because `int.__mul__`
returns `NotImplemented` and `Vector.__rmul__` picks it up.

The in-place variants (`__iadd__` etc.) are consulted first for augmented assignment, per
L01 §2.5, with fallback to `__add__`/`__radd__` and an unconditional store.

**`sum()` and the `start` trap.** `sum(vectors)` begins with `0 + vectors[0]`, so your type
needs `__radd__` that tolerates `0`, or callers must pass `start=Vector.zero()`. Handling
`0` specially in `__radd__` is a common, slightly ugly, and entirely standard concession.

### 2.3 The container protocols

**Sized** — `__len__`. Must return a non-negative `int`. `len()` on an object whose
`__len__` returns a non-int raises. Note: `__len__` is also consulted for truthiness
(§2.6).

**Container** — `__contains__(self, item)`. If absent, `in` falls back to iterating and
comparing with `==` (identity first, then equality — the NaN behaviour of L05 §2.5). This
fallback means `in` on a large sequence without `__contains__` is O(n) and on an infinite
iterator never returns.

**Iterable / Iterator.** These are *different*:

- An **iterable** has `__iter__` returning a fresh iterator.
- An **iterator** has `__next__` *and* `__iter__` returning `self`.

A `list` is iterable but not an iterator: `iter(xs)` gives a new cursor each time, so two
`for` loops over the same list both work. A generator is an iterator: it is consumed once.
Conflating the two produces the classic bug where a function iterates its argument twice
and silently sees nothing the second time.

```python
def bad(rows):
    n = sum(1 for _ in rows)      # consumes rows if it is an iterator
    return [r for r in rows], n   # [] for a generator, correct for a list
```

Defend with `rows = list(rows)` at the boundary, or by typing the parameter
`Sequence[Row]` rather than `Iterable[Row]` and letting the type checker enforce it. The
type system encodes exactly this distinction — use it.

The legacy **`__getitem__` iteration fallback**: a class with `__getitem__` accepting
integers from 0 and raising `IndexError` at the end is iterable without `__iter__`. This
predates `__iter__` and is still live. It is why some old classes are iterable
unexpectedly, and it is a good trick for a minimal sequence.

**Sequence** — `__getitem__`, `__len__`, and by convention `__contains__`, `__iter__`,
`__reversed__`, `index`, `count`. `collections.abc.Sequence` supplies the last five as
mixin methods once you provide the first two — one of the genuinely good uses of an ABC.

`__getitem__` must handle `slice` objects if you want slicing:

```python
def __getitem__(self, key: int | slice):
    if isinstance(key, slice):
        return type(self)(self._items[key])     # return same type, not a list
    return self._items[key]
```

Returning `self._items[key]` (a plain list) for a slice is a common defect: `v[1:3]` on
your `Vector` should be a `Vector`.

Negative indices, `IndexError` for out of range, `TypeError` for a non-index key — all
your responsibility.

**Mapping** — `__getitem__`, `__setitem__`, `__delitem__`, `__iter__`, `__len__`, and
`__missing__` (consulted by `dict.__getitem__` only, for `dict` subclasses — this is how
`defaultdict` works, and it does *not* fire for `.get()`).

### 2.4 Callables, context managers, and descriptors

**`__call__`** makes an instance callable. Its uses: stateful function-likes (a rate
limiter, a memoizer with introspectable state), and classes that need a function's
interface without a closure's opacity. A class with `__call__` is a closure you can debug.

**Context managers** — `__enter__` / `__exit__(exc_type, exc, tb)`. Two rules people get
wrong:

- `__enter__`'s return value is what `as` binds. It is *not* required to be `self`, and for
  `open()` it happens to be. Returning something else is legitimate and occasionally right.
- `__exit__` returning a **truthy** value **suppresses the exception**. Returning `None`
  (the default) propagates it. Accidentally returning a truthy value — e.g. ending
  `__exit__` with `return self.log(...)` — swallows exceptions silently. This is a real and
  nasty bug class.

`contextlib.contextmanager` turns a generator into one; the code before `yield` is
`__enter__`, after is `__exit__`, and an exception is thrown *into* the generator at the
`yield`, so you need `try/finally` to guarantee cleanup:

```python
@contextlib.contextmanager
def acquired(lock):
    lock.acquire()
    try:
        yield lock
    finally:
        lock.release()        # without try/finally, an exception skips this
```

`contextlib.ExitStack` composes a dynamic number of context managers and is
under-used; it is the right answer whenever the set of resources is not known statically.

### 2.5 String conversion: `__repr__`, `__str__`, `__format__`

- `__repr__` — unambiguous, for developers, ideally `eval`-able. Used by the REPL, in
  containers, in debuggers, and by `%r` / `!r`. **Always define it.** The default
  (`<Foo object at 0x7f...>`) makes every log line and every failing test harder to read.
- `__str__` — readable, for users. Defaults to `__repr__`. Define it only when the two
  should genuinely differ.
- `__format__(self, spec)` — backs `format(x, spec)` and f-string format specs. Default
  implementation raises for a non-empty spec, which is why `f"{my_obj:>10}"` fails on most
  classes. If your type has a natural textual form, implement it and delegate the spec:
  `return format(str(self), spec)`.

An f-string `f"{x}"` calls `__format__` with an empty spec (which defaults to `str(x)`);
`f"{x!r}"` calls `__repr__`. Both bypass the instance, per §2.1.

### 2.6 Truthiness

`bool(x)`:

1. `__bool__` if defined — must return an actual `bool`.
2. else `__len__` if defined — zero is false.
3. else `True`.

Consequence: **an empty container is falsy**, so `if my_collection:` means "is non-empty",
not "is not None". A function returning `MyCollection() | None` cannot be tested with
`if result:` — you need `if result is not None:`. Every codebase has this bug somewhere.

Types with an ambiguous truth value should raise: `numpy` arrays do
(`ValueError: truth value of an array is ambiguous`), and that is good design worth
imitating when your type has no sensible single answer.

### 2.7 Copy, pickle, and the reconstruction protocol

- `__copy__`, `__deepcopy__(memo)` — explicit hooks for `copy`.
- `__reduce__` / `__reduce_ex__` — the pickle protocol; returns a callable and arguments to
  reconstruct.
- `__getstate__` / `__setstate__` — the common, simpler hooks. Since 3.11 `object` provides
  a default `__getstate__`.
- `__getnewargs__` / `__getnewargs_ex__` — needed when `__new__` requires arguments
  (immutable types, L02 §2.4).

The thing to know: **pickle does not call `__init__`.** It allocates via `__new__` (or
`__reduce__`'s callable) and restores state directly. Any invariant your `__init__`
establishes — a validated field, a registered instance, an opened resource — is not
re-established on unpickling unless you implement `__setstate__`.

Related caution: never unpickle untrusted data. `__reduce__` can name any callable, so a
pickle is arbitrary code execution by design.

### 2.8 Pattern matching (`__match_args__`)

```python
@dataclass
class Point:
    x: float
    y: float

match p:
    case Point(0, 0):        print("origin")
    case Point(x=0, y=y):    print(f"on y axis at {y}")
    case Point(x, y):        print(f"at {x},{y}")
```

Positional patterns use `__match_args__`, a class-level tuple of attribute names.
`dataclass` sets it from the field order; other classes must set it themselves. Keyword
patterns use plain `getattr`. `case` with a class pattern first does an `isinstance` check
— so pattern matching is nominal, and a `Protocol` will not match structurally (unless
`runtime_checkable`, which only checks method presence).

Guards (`case Point(x, y) if x == y:`) run after binding. Capture patterns bind (L01 §2.3)
and `_` is a wildcard that binds nothing.

## 3. Construction: a `Vector` that feels native

The classic exercise, done properly. Build up.

**Stage 1 — construction and repr.**

```python
from collections.abc import Iterable, Iterator, Sequence
from typing import Any, Self
import math, reprlib, functools, operator

class Vector:
    __slots__ = ("_c",)

    def __init__(self, components: Iterable[float] = ()) -> None:
        self._c: tuple[float, ...] = tuple(float(x) for x in components)

    def __repr__(self) -> str:
        return f"Vector({reprlib.repr(list(self._c))})"
```

`reprlib.repr` truncates long output — important because a `repr` that prints ten thousand
floats makes debuggers unusable. Small detail, real quality signal.

**Stage 2 — sequence protocol.** Add `__len__`, `__getitem__` with slice support returning
a `Vector`, `__iter__` (free via `__getitem__`, but define it: it is faster and clearer),
`__contains__` (free via iteration, but O(n) — define it only if you can do better).

Then a nice touch — named component access via `__getattr__`:

```python
_NAMES = "xyzt"

def __getattr__(self, name: str) -> float:
    if len(name) == 1:
        i = _NAMES.find(name)
        if 0 <= i < len(self._c):
            return self._c[i]
    raise AttributeError(f"{type(self).__name__!r} has no attribute {name!r}")
```

**Now the trap.** Without `__slots__` this class would need a matching `__setattr__`:
`v.x = 5` would create an instance attribute that shadows nothing (since `__getattr__` is a
fallback), giving `v.x == 5` but `v[0]` unchanged — two sources of truth. With
`__slots__ = ("_c",)` the assignment raises, which is the correct outcome. This is the L03
priority rule doing real work: understand *why* `__slots__` fixed it, not just that it did.

**Stage 3 — equality and hashing.** Per L05:

```python
def __eq__(self, other: object) -> bool:
    if not isinstance(other, Vector): return NotImplemented
    return len(self) == len(other) and all(a == b for a, b in zip(self, other))

def __hash__(self) -> int:
    return functools.reduce(operator.xor, map(hash, self._c), 0)
```

Then immediately criticize the hash: XOR is a weak mixing function (`Vector([1,2])` and
`Vector([2,1])` collide). `hash(self._c)` is better in every way. Use it, and keep the XOR
version in a comment as a worked example of what not to do.

**Stage 4 — arithmetic with proper dispatch.**

```python
def __add__(self, other: Any) -> Self:
    try:
        pairs = itertools.zip_longest(self, other, fillvalue=0.0)
    except TypeError:
        return NotImplemented
    return type(self)(a + b for a, b in pairs)

def __radd__(self, other: Any) -> Self:
    return self + other

def __mul__(self, scalar: Any) -> Self:
    try:
        factor = float(scalar)
    except (TypeError, ValueError):
        return NotImplemented
    return type(self)(x * factor for x in self)

__rmul__ = __mul__
```

Note `type(self)(...)` rather than `Vector(...)`: subclasses get their own type back. Note
the `NotImplemented` returns: they are what make `3 * v` and `np.float64(3) * v` work.
Note `zip_longest`: an explicit decision that vectors of different length add by
zero-padding, which you should *document* — the alternative (raise) is equally defensible
and the point is that you chose.

**Stage 5 — `__format__`.** Support `f"{v}"`, `f"{v:.3f}"` (apply the spec per component),
and a custom `f"{v:h}"` for hyperspherical coordinates. This is where you learn that
format-spec parsing is your problem, and that inventing custom spec characters is a
liberty the data model deliberately grants.

**Stage 6 — `__match_args__`, `__copy__`, `__reduce__`.** And then the final question: how
much of this is `Sequence`'s job? Try inheriting `collections.abc.Sequence` and deleting
everything it provides. Compare line counts and measure the performance difference. Report
both.

## 4. Failure modes

- **`__exit__` returning truthy.** Silently swallows exceptions.
- **Iterating an iterator twice.** §2.3.
- **Slicing returning the wrong type.** §2.3.
- **`if collection:` when you meant `is not None`.** §2.6.
- **No `__repr__`.** Every debugging session is worse.
- **`__format__` unimplemented**, then `f"{obj:>20}"` fails in production log formatting.
- **Expecting `__init__` on unpickle.** §2.7.
- **Defining `__eq__` on a mutable type used as a dict key.** L05 again, but it shows up
  here when you add `__eq__` to make tests read nicer.
- **Assuming `__getattr__` catches special methods.** It does not (§2.1).
- **`sum()` without `__radd__` tolerating `0`.**

## 5. Exercises

### Warm-up (25 min)

**W1.** Write a class that is iterable but has no `__iter__`. Explain the mechanism.

**W2.** Write a context manager that swallows `ValueError` and propagates everything else.
Then write the test that would have caught an accidental blanket-swallow.

**W3.** Demonstrate the subclass-priority rule of §2.2 with a two-class example where
`a + b` calls `b.__radd__` even though `a.__add__` exists and would have worked.

### Core (2.5 h)

**C1 — `Vector`, complete** *(Problem Set 2's centrepiece.)* Complete all six stages of
§3. Deliverable: the class, a test suite, and a 600-word design note answering: which
protocols did you implement and which did you deliberately omit; where did you diverge from
`list`'s behaviour and why; what does your type do that `numpy.ndarray` does not, and is
that a reason for it to exist?

**C2 — A frozen, sliceable, pattern-matchable record type.** Build `Row`, a fixed-schema
record supporting `row["name"]`, `row.name`, `row[0]`, slicing, iteration, equality,
hashing, pattern matching, `copy`, and `pickle` (with `__init__`'s validation re-run on
unpickle). Write a test for each protocol. Then compare against `typing.NamedTuple` and
list three things it does better than yours.

**C3 — The `__exit__` audit.** Search a codebase (yours or an open-source project) for
`__exit__` implementations and `@contextmanager` generators. For each, determine whether it
can swallow an exception unintentionally and whether cleanup is guaranteed on exception.
Report findings. Write a `ruff`/AST check that flags `__exit__` methods whose last
statement is a non-`None`-returning expression.

**C4 — Protocol coverage matrix.** Produce a table of every special method in Language
Reference §3.3, with: what invokes it, whether there is a fallback, and whether
`__getattr__` can supply it. Roughly 80 rows. Tedious, and the single most useful reference
document you will produce in this course — you will consult it for years.

### Challenge

**X1.** Implement a lazy, infinite `Stream` type supporting `__iter__`, slicing (returning a
`Stream`), `__getitem__` for a single index, `map`/`filter`/`take` as methods, and memoized
replay so that iterating twice gives the same elements. Then explain the memory model: what
does it retain, and under what usage does it become a leak?

**X2.** Write `conforms(cls, protocol) -> Report` that checks statically whether a class
implements a given `collections.abc` protocol *correctly* — not just that the methods exist
but that their signatures are compatible (use `inspect.signature`). Report what you cannot
check and why. Compare with what `mypy` catches.

## 6. Self-check

1. Why do implicit special-method calls bypass the instance dict?
2. Give the five-step dispatch for `a + b`, including the subclass rule.
3. Distinguish iterable from iterator, and give a bug caused by conflating them.
4. What are the two rules about `__enter__`'s return value and `__exit__`'s return value?
5. State the truthiness resolution order.
6. Why does pickle not call `__init__`, and what must you do about it?
7. What is `__match_args__` for, and what sets it automatically?
8. Why should `__getitem__` with a slice return your own type?

## 7. Primary sources

- Language Reference §3.3 in full. This is the specification of everything above; read it
  end to end at least once in your life, and this is the week.
- PEP 343 (the `with` statement) — the rationale section explains the `__exit__` return
  convention.
- PEP 634/635/636 (structural pattern matching) — 635 is the *motivation and rationale*
  document and is the interesting one.
- Ramalho, *Fluent Python* 2e, chs. 1, 11–13, 16.

---

**Previous:** [L05](L05-identity-equality-hashing-ordering.md) · **Next:**
[L07 — Scopes, Frames, and Closures](L07-scopes-frames-and-closures.md)
