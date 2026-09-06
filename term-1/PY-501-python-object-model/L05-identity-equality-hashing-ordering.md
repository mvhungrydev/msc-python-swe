# PY-501 · Lesson 05 — Identity, Equality, Hashing, and Ordering

**Estimated study time:** 3.5 hours
**Prerequisites:** L01–L04

---

## 1. Orientation

```python
class Point:
    def __init__(self, x, y): self.x, self.y = x, y
    def __eq__(self, other): return (self.x, self.y) == (other.x, other.y)

p = Point(1, 2)
{p}          # TypeError: unhashable type: 'Point'
```

Defining `__eq__` silently sets `__hash__ = None`. That is not a bug — it is the language
protecting an invariant, and the invariant is the subject of this lesson.

Equality looks trivial and is not. It is the place where a language's value semantics live,
and getting it wrong produces bugs that are invisible until a dict lookup misses, a `set`
grows duplicates, or a cache returns the wrong entry six months later.

## 2. Theory

### 2.1 The hash invariant

The whole of hashing rests on one requirement:

> **If `a == b` then `hash(a) == hash(b)`.**

The converse need not hold: unequal objects may collide, and hash tables handle that. But
an equal pair with different hashes breaks every hash-based container, because a lookup
computes the hash, goes to that bucket, and never examines the bucket where the equal
object actually lives.

Corollary: **hash must be derived from the same data as equality, and that data must not
change while the object is in a hash-based container.** This is why mutable built-ins
(`list`, `dict`, `set`) are unhashable — not because hashing them is hard, but because a
mutation would silently corrupt any container holding them.

Corollary 2: a hashable object should be **immutable in the fields that determine
equality**. It may have other mutable fields (a cache, a timestamp) provided they play no
part in `__eq__`.

CPython enforces the first half of this automatically: defining `__eq__` in a class sets
`__hash__` to `None` unless you also define `__hash__`. The reasoning is that the inherited
`object.__hash__` (identity-based) would almost certainly violate the invariant against
your new `__eq__`.

To restore hashability, define it explicitly:

```python
class Point:
    __slots__ = ("x", "y")
    def __init__(self, x: float, y: float) -> None:
        object.__setattr__(self, "x", x)   # if you also freeze it
        object.__setattr__(self, "y", y)
    def __eq__(self, other: object) -> bool:
        if not isinstance(other, Point):
            return NotImplemented
        return (self.x, self.y) == (other.x, other.y)
    def __hash__(self) -> int:
        return hash((self.x, self.y))
```

Hashing a tuple of the same fields you compare is the idiomatic and near-always-correct
implementation.

### 2.2 `NotImplemented` and the reflected-operand protocol

Note `return NotImplemented`, not `False`. This is a protocol, and the distinction matters.

When Python evaluates `a == b`:

1. If `type(b)` is a *proper subclass* of `type(a)` and overrides `__eq__`, try
   `b.__eq__(a)` **first** (the subclass gets priority — it may know about the superclass).
2. Otherwise try `a.__eq__(b)`.
3. If that returns `NotImplemented`, try the reflected operation `b.__eq__(a)`.
4. If that also returns `NotImplemented`, fall back to identity comparison (`a is b`) for
   `==`, or raise `TypeError` for ordering operators.

Returning `False` short-circuits this: you have declared "these are unequal", denying the
other operand its chance to answer. Returning `NotImplemented` says "I don't know how to
compare with that", which is the truth and lets the protocol work.

The same applies to every binary operator (L06). `NotImplemented` is a singleton, distinct
from `NotImplementedError` (an exception), and — a small trap — it is truthy, so
`if a.__eq__(b):` on a `NotImplemented` result silently succeeds. Never call dunder methods
directly.

### 2.3 What `hash` actually needs to be

Requirements, in order of importance:

1. **Consistent with `__eq__`** (§2.1).
2. **Stable for the object's lifetime.**
3. **Cheap.** It is on the critical path of every dict operation.
4. **Well-distributed.** Poor distribution turns O(1) lookups into O(n).

`hash(tuple_of_fields)` satisfies all four for most types, because tuple hashing is
implemented in C with a decent mixing function.

Two subtleties:

**Hash randomization.** Since Python 3.3, `str` and `bytes` hashing is salted per process
(PEP 456, SipHash) to defend against algorithmic-complexity denial-of-service attacks. This
means **`hash("abc")` differs between runs**. Never persist a Python `hash()` value, never
use it as a database key, never assume iteration order derived from it is reproducible
across processes. Set `PYTHONHASHSEED` only for debugging.

**`hash(-1)`.** CPython uses `-1` as an error sentinel internally, so no object's hash may
be `-1`; `hash(-1)` returns `-2`. Harmless, but it appears in interview questions and in
your own hash implementations if you do bit tricks.

### 2.4 Total ordering

Comparison operators map to six methods: `__lt__`, `__le__`, `__gt__`, `__ge__`, `__eq__`,
`__ne__`. The reflected pairs are `<`/`>` and `<=`/`>=`; `__ne__` defaults to the negation
of `__eq__` unless overridden (Python 3 gives you this for free — do not write `__ne__`).

`functools.total_ordering` fills in the rest from `__eq__` plus one of the four ordering
methods:

```python
import functools

@functools.total_ordering
class Version:
    def __init__(self, *parts: int) -> None: self.parts = tuple(parts)
    def __eq__(self, other: object) -> bool:
        if not isinstance(other, Version): return NotImplemented
        return self.parts == other.parts
    def __lt__(self, other: "Version") -> bool:
        if not isinstance(other, Version): return NotImplemented
        return self.parts < other.parts
    def __hash__(self) -> int: return hash(self.parts)
```

It costs a little speed (derived operators go through an extra call) and is worth it for
correctness. Write it by hand only if profiling says so.

**A total order must be:** irreflexive for `<`, transitive, and trichotomous (for any `a,b`
exactly one of `a<b`, `a==b`, `a>b`). Python does not check this. `sort()`, `min()`,
`heapq`, and `bisect` all *assume* it, and if your comparison is inconsistent they do not
raise — they produce wrong answers quietly. Property-based testing (SE-511 L04) is the
practical defence.

**Partial orders** are legitimate and require care. Sets are partially ordered by `⊆`, and
`{1} < {2}` is `False` while `{1} >= {2}` is also `False`. That is correct, and it means
you must not `sort()` a list of sets and expect anything meaningful. If your type has a
partial order, do not implement `__lt__` — implement a named method, or you will be
silently misused by the standard library.

### 2.5 NaN, and why equality is not an equivalence relation

`float('nan') != float('nan')`. Worse:

```python
x = float('nan')
x == x        # False
x is x        # True
[x] == [x]    # True  (!)
x in [x]      # True  (!)
```

Containers use an *identity-first* comparison (`is` or `==`) as an optimization. So
membership and list equality disagree with scalar equality on NaN. IEEE-754 mandates the
scalar behaviour; Python's containers prioritize the reflexivity that their algorithms need.

The lesson generalizes: **if your `__eq__` is not reflexive, containers will behave in ways
your scalar tests do not predict.** Never write a non-reflexive `__eq__` unless you are
implementing IEEE floats.

### 2.6 Equality across types, and the `bool`/`int` trap

```python
True == 1          # True
hash(True) == hash(1)   # True
{1, True}          # {1}
{1: 'a', True: 'b'}     # {1: 'b'}  — same key!
```

`bool` is a subclass of `int` with `True == 1`, so they collide in dicts and sets. This
occasionally matters: a dict keyed by mixed booleans and integers loses entries.

More generally, deciding whether your type compares equal to *other* types is a design
decision. `Fraction(1,2) == 0.5` is `True` by deliberate design across the numeric tower;
`Point(1,2) == (1,2)` is a choice you should probably decline, because equality between a
type and its serialization is the beginning of a long unhappy road.

**Guidance:** be conservative. Return `NotImplemented` for types you were not designed to
compare with. Widening equality later is easy; narrowing it breaks callers.

### 2.7 Dataclasses do this for you (mostly)

```python
from dataclasses import dataclass

@dataclass(frozen=True, slots=True)
class Point:
    x: float
    y: float
```

Generates `__init__`, `__repr__`, `__eq__` (field-tuple comparison, and — importantly —
*type-checked*: `other.__class__ is self.__class__`, else `NotImplemented`), and, because
`frozen=True` and `eq=True`, a `__hash__`.

The rules for `__hash__` generation are worth knowing exactly, because they surprise people:

| `eq` | `frozen` | Result |
|---|---|---|
| True (default) | True | `__hash__` generated from fields |
| True | False | `__hash__ = None` — **unhashable** |
| False | either | `__hash__` inherited from `object` (identity) |

The middle row is the common surprise: a plain `@dataclass` is *not* hashable. That is the
correct default (§2.1: mutable equality fields must not be hashed) and you should not
override it with `unsafe_hash=True` — the name is a warning, not a joke.

Exclude a field from equality with `field(compare=False)`; exclude it from hashing
separately with `field(hash=False)`.

## 3. Construction: a value type that behaves

Build `Money`, incrementally, exercising every rule above.

**Version 1 — naive.**

```python
class Money:
    def __init__(self, amount: float, currency: str) -> None:
        self.amount, self.currency = amount, currency
    def __eq__(self, other): return self.amount == other.amount
```

Four defects, in order of severity:

1. `float` for money. Binary floating point cannot represent 0.10; sums drift.
   `Decimal` or an integer count of minor units. This is not pedantry — it is the most
   consequential single decision in the class.
2. `__eq__` ignores currency. `Money(1,'USD') == Money(1,'EUR')` is `True`.
3. `__eq__` raises `AttributeError` on non-`Money` operands instead of returning
   `NotImplemented`.
4. No `__hash__`, so it is now unhashable (correctly, but unintentionally).

**Version 2.**

```python
from decimal import Decimal
from dataclasses import dataclass

@dataclass(frozen=True, slots=True, order=False)
class Money:
    amount: Decimal
    currency: str

    def __post_init__(self) -> None:
        if len(self.currency) != 3 or not self.currency.isupper():
            raise ValueError(f"bad currency code: {self.currency!r}")
```

Now: immutable, hashable, correct equality including currency, sensible `repr`.

**Version 3 — ordering, and the partial-order problem.** Should `Money` be orderable?
Within a currency, yes; across currencies, the question is meaningless. This is a
*partial* order, and §2.4 says do not implement `__lt__` for a partial order.

Two defensible designs:

```python
# (a) Raise on cross-currency comparison — total order within a currency, error otherwise.
def __lt__(self, other: "Money") -> bool:
    if not isinstance(other, Money): return NotImplemented
    if other.currency != self.currency:
        raise TypeError(f"cannot compare {self.currency} with {other.currency}")
    return self.amount < other.amount
```

```python
# (b) Don't implement ordering. Provide Money.compare_same_currency(a, b) explicitly.
```

(a) is more usable and violates the "comparison operators do not raise" expectation that
`sorted()` relies on — a list of mixed currencies will blow up mid-sort, leaving the list
partially reordered. (b) is more honest and more annoying. Pick one, and *write down why*:
this is exactly the kind of decision the assessment rubric's "Justification" criterion is
looking for.

**Version 4 — arithmetic.** `Money + Money` (same currency), `Money * int`, and crucially
*not* `Money * Money`. Return `NotImplemented` for unsupported combinations so that a
future `Rate` type can define `__rmul__` and interoperate. L06 covers the dispatch rules
this depends on.

## 4. Failure modes

- **`__eq__` without `__hash__`.** Silent unhashability. The error appears far from the
  definition.
- **`__hash__` over mutable fields.** Object mutates while in a `set`; lookups miss; the
  object is "in" the set and cannot be found. The most confusing bug in this lesson.
- **Returning `False` instead of `NotImplemented`.** Breaks interoperability, breaks
  subclass comparison, and produces asymmetric equality: `a == b` `False` but `b == a`
  `True`.
- **Non-transitive equality.** Comparing with a tolerance (`abs(a-b) < eps`) is not
  transitive; put such a comparison in a named method, never in `__eq__`, because `set`
  and `dict` will misbehave in ways that are essentially undebuggable.
- **Persisting `hash()`.** PEP 456 randomization means it is not stable across runs. Use
  `hashlib` for anything durable.
- **`unsafe_hash=True`** to silence an error you did not understand.
- **Ordering a partial order.** §2.4.
- **Assuming `sorted()` validates your comparator.** It does not; `list.sort` uses Timsort,
  which with an inconsistent comparator can produce garbage or, in some implementations,
  raise `ValueError: list modified during sort`.

## 5. Exercises

### Warm-up (25 min)

**W1.** Write a class that violates the hash invariant. Insert an instance into a `set`,
mutate it, and demonstrate that `x in s` is `False` while `x in list(s)` is `True`.
Explain each result.

**W2.** Show that returning `False` from `__eq__` produces asymmetric equality, with a
two-class example.

**W3.** Explain, with reference to §2.6, why `{0: 'a', False: 'b', 0.0: 'c'}` has one entry
and what its value is.

### Core (2 h)

**C1 — The full `Money`.** Complete version 4. Requirements: `Decimal` internals with a
documented rounding policy; frozen and hashable; equality across currency correct;
addition/subtraction within currency; multiplication by an integer or `Decimal`;
`NotImplemented` everywhere it belongs; `__repr__` that round-trips through `eval`;
`__format__` supporting `f"{m:,.2f}"`. Write a test suite including at least one property
test asserting `a + b == b + a` and one asserting hash/eq consistency over generated
inputs.

**C2 — Equality audit.** Take three classes from a codebase you work on. For each,
determine: is `__eq__` reflexive, symmetric, transitive? Is the hash invariant held? Can
an instance mutate while hashed? Write up the findings. Where a class is broken, write the
smallest failing test that demonstrates it. (Reporting "all three were fine" is an
acceptable outcome only if you show the tests you wrote to establish it.)

**C3 — Tolerance comparison, done right.** Design an API for approximate equality of a
`Vector` type that does *not* break `set`/`dict`. Consider: a named method, a comparison
context manager, a wrapper type, and `math.isclose` semantics. Recommend one and justify
against the other three. Then explain why `pytest.approx` is designed the way it is.

**C4 — Hash distribution.** Write a hash function for a 3-field record two ways: `hash(tuple)`
and a hand-rolled XOR of field hashes. Generate 10⁶ realistic records, measure bucket
distribution and dict lookup time for both. Explain why XOR is a poor mixing function.

### Challenge

**X1.** Implement a `frozendict` that is hashable, immutable, and correct under
`==`, `hash`, `copy`, `pickle`, and `|` merging. Then find and document at least three
places where it cannot be made to behave exactly like `dict` (hint: `dict` subclass
special-casing in C, `__reversed__`, insertion-order guarantees under merge). Read PEP 416
(rejected) and PEP 603 and write 400 words on why a `frozendict` is not in the standard
library.

**X2.** Read the CPython `dict` implementation notes (`Objects/dictobject.c` header
comment) and explain: open addressing vs chaining, the compact-dict layout that gives
ordering for free, and what happens on resize. Then predict and measure the cost of
inserting keys with deliberately colliding hashes.

## 6. Self-check

1. State the hash invariant and one consequence of violating it.
2. Why does defining `__eq__` set `__hash__` to `None`?
3. Why return `NotImplemented` rather than `False`? What happens next?
4. Give the four-step dispatch order for `a == b`, including the subclass rule.
5. Why is `hash("abc")` different between runs, and what must you never do because of it?
6. State the three properties a total order requires. What breaks if `sort` gets a
   comparator lacking them?
7. Give the dataclass `__hash__` generation table.
8. Why is a tolerance-based `__eq__` dangerous?

## 7. Primary sources

- Language Reference §3.3.1 (Basic customization) — `__eq__`, `__hash__`, `__lt__` and
  friends. Two pages, dense, authoritative.
- PEP 456 — Secure and interchangeable hash algorithm. Read the *Attack* section.
- PEP 557 — Data Classes; the `__hash__` rules section specifically.
- Goldberg, "What Every Computer Scientist Should Know About Floating-Point Arithmetic"
  (1991), §1 — for the `Decimal` argument in §3.

---

**Previous:** [L04](L04-inheritance-super-and-the-mro.md) · **Next:**
[L06 — The Data Model: Protocols and Operator Dispatch](L06-data-model-and-operator-dispatch.md)
