# SE-511 · Lesson 07 — Generics, Variance, and Protocols

**Estimated study time:** 4 hours
**Prerequisites:** L06, PY-501 L04

---

## 1. Orientation

```python
class Animal: ...
class Dog(Animal): ...

def feed_all(animals: list[Animal]) -> None:
    animals.append(Cat())

dogs: list[Dog] = [Dog()]
feed_all(dogs)          # mypy: error — and it is right
```

`Dog` is a subtype of `Animal`, so why is `list[Dog]` not a subtype of `list[Animal]`?
Because if it were, `feed_all` could put a `Cat` into your list of dogs. The rule that
prevents this — **variance** — is the one piece of type theory every working engineer must
actually know, and the one most people learn by trial and error from checker errors.

This lesson makes it derivable rather than memorized.

## 2. Theory

### 2.1 Subtyping, stated properly

`S` is a subtype of `T` (written `S <: T`) if **a value of type `S` can be used anywhere a
value of type `T` is expected, without breaking anything.** This is the *Liskov substitution
principle* stated as a typing rule.

"Without breaking anything" means: every operation valid on `T` is valid on `S`, and the
guarantees `T` makes are still made by `S`.

### 2.2 Variance, derived

Given `S <: T`, how does a generic `F[S]` relate to `F[T]`?

- **Covariant**: `F[S] <: F[T]`. The container varies *with* the parameter.
- **Contravariant**: `F[T] <: F[S]`. Varies *against*.
- **Invariant**: no relation.

The rule that generates all of these, and which you should derive rather than memorize:

> **A type parameter that appears only in *output* positions is covariant.
> A type parameter that appears only in *input* positions is contravariant.
> A parameter in both is invariant.**

Apply it:

- `Sequence[T]` — `T` appears only as a return (`__getitem__ -> T`, `__iter__ ->
  Iterator[T]`). **Covariant.** `Sequence[Dog] <: Sequence[Animal]` — safe, because you can
  only read.
- `list[T]` — `T` appears as a return (`__getitem__`) *and* as a parameter (`append(x: T)`).
  **Invariant.** This is the opening example.
- `Callable[[T], R]` — `T` is a parameter, `R` a return. **Contravariant in `T`, covariant
  in `R`.** So `Callable[[Animal], Dog] <: Callable[[Dog], Animal]`: a function that accepts
  *more* and promises *more specifically* is substitutable for one that accepts less and
  promises less.
- `Iterable[T]`, `Iterator[T]`, `Mapping[K, V]` in `V`, `frozenset[T]` — covariant.
- `MutableSequence[T]`, `MutableMapping` in both, `set[T]` — invariant.
- `Mapping[K, V]` in `K` — invariant, because `K` appears in `__getitem__(k: K)` (input) and
  in `keys() -> KeysView[K]` (output).

**The design consequence**, which is the practical payoff: *annotate parameters with the
covariant read-only protocol whenever you only read.* `def total(xs: Sequence[Money])`
accepts `list[Money]`, `tuple[Money, ...]`, and — crucially — `list[USD]` if `USD <: Money`.
`def total(xs: list[Money])` accepts none of those. Choosing `Sequence` over `list` in a
signature is not style; it is the difference between a usable and an unusable API.

### 2.3 Declaring generics

Modern syntax (PEP 695, Python 3.12+):

```python
def first[T](xs: Sequence[T]) -> T:
    return xs[0]

class Box[T]:
    def __init__(self, item: T) -> None: self._item = item
    def get(self) -> T: return self._item

class Stack[T]:
    def push(self, item: T) -> None: ...
    def pop(self) -> T: ...
```

Variance under PEP 695 is **inferred** from usage, which removes an entire category of
error. `Box` above is covariant (read-only); `Stack` is invariant (both positions).

Pre-3.12 syntax, which you will still read constantly:

```python
from typing import TypeVar, Generic
T = TypeVar("T")
T_co = TypeVar("T_co", covariant=True)
T_contra = TypeVar("T_contra", contravariant=True)

class Box(Generic[T_co]):
    def get(self) -> T_co: ...
```

The naming convention (`_co`, `_contra`) is enforced by convention only, but follow it —
it is how readers know without checking.

**Bounds and constraints:**

```python
def largest[T: (int, float, str)](xs: Sequence[T]) -> T:   # constrained: exactly one of these
    ...

def clamp[T: Comparable](x: T, lo: T, hi: T) -> T:          # bounded: any subtype of Comparable
    ...
```

*Constrained* means the parameter must be exactly one of the listed types (no unification
across them). *Bounded* means any subtype. Bounds are almost always what you want;
constraints are for cases like `AnyStr` where behaviour genuinely differs per type.

### 2.4 Protocols: structural typing

`typing.Protocol` (PEP 544) gives Python static duck typing:

```python
from typing import Protocol

class SupportsClose(Protocol):
    def close(self) -> None: ...

def cleanup(x: SupportsClose) -> None:
    x.close()

cleanup(open("f"))          # fine — file has close(), never heard of SupportsClose
```

No inheritance, no registration. The checker verifies structurally that the argument's type
has a compatible `close`.

**When to use a Protocol rather than an ABC** — the rule that actually decides it:

> **Define a Protocol for what you *consume*. Define an ABC for what you *share*.**

If you are writing a function that needs "something with a `read` method", a Protocol is
right: it imposes nothing on callers, works with types that predate you, and keeps the
dependency arrow pointing the right way (the *consumer* owns the interface — this is the
Dependency Inversion Principle expressed in the type system, SE-521 L03).

If you have five implementations sharing a template method and an invariant, an ABC is
right: you are sharing code, not just describing a shape.

**Protocol details that catch people:**

- Protocols with non-method members (attributes) are matched against attributes, and a
  *mutable* attribute in a Protocol makes it invariant in that attribute's type.
- `@runtime_checkable` allows `isinstance`, but it only checks **method presence**, not
  signatures, and not attributes for non-data protocols. It is a weak, cheap check; do not
  mistake it for verification.
- Protocols can inherit from other Protocols and can have default implementations (which
  are then usable only by explicit subclasses).
- A Protocol used as a *return* type is usually a mistake: you are promising less than you
  have, for no reason. Protocols belong in parameter positions.

**`Self`** (PEP 673) for fluent APIs and constructors:

```python
from typing import Self

class Builder:
    def with_name(self, n: str) -> Self:      # subclasses get their own type back
        self._name = n
        return self
```

### 2.5 Overloads, `ParamSpec`, and decorators

**`@overload`** for functions whose return type depends on argument types or values:

```python
from typing import overload, Literal

@overload
def get(key: str, default: None = None) -> str | None: ...
@overload
def get(key: str, default: str) -> str: ...
def get(key: str, default: str | None = None) -> str | None:
    ...
```

Rules: overload stubs have no bodies (`...`), the implementation is not itself an overload
and must be compatible with all of them, and **order matters** — the checker takes the first
matching overload. Overlapping overloads whose returns are incompatible are an error worth
listening to.

Overloads are how you express `open()` returning `TextIOWrapper` or `BufferedReader`
depending on `mode`, using `Literal`. When you find yourself writing three overloads,
consider whether it should be three functions; sometimes yes, sometimes the API genuinely
needs it.

**`ParamSpec`** (PEP 612) for decorators that preserve signatures:

```python
def logged[**P, R](fn: Callable[P, R]) -> Callable[P, R]:
    @functools.wraps(fn)
    def wrapper(*args: P.args, **kwargs: P.kwargs) -> R:
        log.info("calling %s", fn.__name__)
        return fn(*args, **kwargs)
    return wrapper
```

Without `ParamSpec`, the best you could write was `Callable[..., R]`, which erases every
argument type — so decorating a function destroyed its type information, which is why
typed codebases used to avoid decorators. `Concatenate` handles decorators that add or
remove leading parameters:

```python
def with_session[**P, R](
    fn: Callable[Concatenate[Session, P], R]
) -> Callable[P, R]: ...
```

**`TypeVarTuple`** (PEP 646) for variadic generics — shapes in array libraries, and
`*args` of heterogeneous types. Rarely needed in application code; know it exists.

### 2.6 When the checkers disagree

`mypy` and `pyright` disagree, sometimes substantially. Sources of disagreement:

- Inference strength (pyright infers more aggressively, especially for literals and
  narrowing).
- Unsoundness allowances differ.
- Speed of adopting new PEPs.
- Bugs, in both.

**Running both is genuinely worthwhile** on a library, because their union catches more and
because your users may run either. The typing spec (the consolidated *Typing Specification*
that the PEPs now feed into) is the arbiter: when they disagree, read it. Where the spec is
silent, the disagreement is a legitimate implementation choice and you should write code
that satisfies both.

For a **library**, add `py.typed` (PEP 561) so your annotations are visible to consumers.
Without it, everything you export is `Any` to them, and all your typing effort is invisible
outside your own repository. This is a one-line file that a surprising number of typed
libraries forget.

## 3. Construction: a typed repository abstraction

Build the interface that appears in every application, with the types doing real work.

**Version 1 — untyped, and everyone's first attempt.**

```python
class Repository:
    def get(self, id): ...
    def save(self, entity): ...
```

Nothing checked. `repo.get(order_id)` returning a `User` is invisible.

**Version 2 — generic over the entity.**

```python
class Repository[E]:
    def get(self, id: str) -> E | None: ...
    def save(self, entity: E) -> None: ...
```

Better. `Repository[Order].get(...)` returns `Order | None`. But `id: str` is wrong: order
ids and user ids are different things, and passing one where the other belongs is a real
bug class.

**Version 3 — generic over entity and id, with a bound.**

```python
from typing import Protocol

class Entity[Id](Protocol):
    @property
    def id(self) -> Id: ...

class Repository[E: Entity[Any], Id](Protocol):
    def get(self, id: Id) -> E | None: ...
    def save(self, entity: E) -> None: ...
    def delete(self, id: Id) -> None: ...
```

Now consider variance. `E` appears as a return (`get`) and a parameter (`save`) —
**invariant**, correctly: a `Repository[Dog]` is not a `Repository[Animal]`, because you
could `save(Cat())` through the latter view.

Split the reads from the writes and the variance improves:

```python
class ReadRepository[E_co, Id](Protocol):
    def get(self, id: Id) -> E_co | None: ...          # covariant in E

class WriteRepository[E_contra, Id](Protocol):
    def save(self, entity: E_contra) -> None: ...      # contravariant in E
```

This is CQRS falling out of the variance rules — the read model and the write model want
different variances *because they are different interfaces*. That is not a coincidence, and
noticing it is the kind of connection this program exists to build.

**Version 4 — `NewType` ids and the query problem.**

```python
OrderId = NewType("OrderId", str)
UserId = NewType("UserId", str)

class OrderRepository(ReadRepository[Order, OrderId], WriteRepository[Order, OrderId],
                      Protocol):
    def find_by_customer(self, customer: UserId) -> Sequence[Order]: ...
```

Note `Sequence`, not `list`: covariant, so an implementation may return a tuple or a
subclass-typed list.

Then the hard question, which the types make visible: **`find_by_customer` does not
generalize.** Every repository grows bespoke query methods, and no generic `Repository[E]`
can express them. That is a real architectural finding — the generic repository pattern
mostly does not pay for itself, and SE-521 L07 argues it. The type system made you notice.

## 4. Failure modes

- **`list` in parameter positions.** §2.2. Rejects tuples, rejects covariant element types.
- **Declaring a `TypeVar` covariant when it appears in an input position.** The checker will
  usually catch it; when it does not, you have introduced unsoundness by hand.
- **`runtime_checkable` as verification.** Checks method names only.
- **Protocols in return positions.** §2.4.
- **Forgetting `py.typed`.** Your library is untyped to everyone else.
- **`Callable[..., Any]` on decorators.** Use `ParamSpec`.
- **Overload order.** First match wins; putting the general case first makes the specific
  ones dead.
- **Over-generalizing.** `Repository[E, Id, Q, F]` with four parameters is unreadable and
  usually wrong. If a generic needs more than two parameters, question the abstraction.
- **`Any` in a bound** (`E: Entity[Any]` above) — sometimes necessary, always a small
  unsoundness. Note it.

## 5. Exercises

### Warm-up (25 min)

**W1.** Derive from first principles whether `Mapping[K, V]` is covariant, contravariant, or
invariant in each of `K` and `V`. Verify with mypy.

**W2.** Write a covariant generic that mypy rejects because the parameter appears in an
input position. Read the error and explain it in one sentence.

**W3.** Write a decorator with and without `ParamSpec` and show the difference in what the
checker knows about the decorated function.

### Core (2.5 h)

**C1 — The repository.** Complete §3 through version 4. Deliverable: the protocols, one real
implementation, one in-memory fake, the contract test (L02 §2.6) parameterized over both,
and a 500-word note on the variance decisions and the generic-repository critique.

**C2 — Protocol-ize a dependency.** Take a place in your code where you depend on a
concrete class from another module. Replace the dependency with a Protocol *defined in the
consuming module*. Verify nothing else changed. Then write 300 words on how this changes
the direction of the dependency arrow and why that matters (this is the argument SE-521 L03
will formalize).

**C3 — Overloads for a real API.** Find a function in your code whose return type depends on
an argument (a `parse(s, strict=True)` that raises vs returns `None`, a `get(k, default)`).
Write correct overloads. Verify the checker narrows correctly at three call sites. Then
argue whether it should have been two functions instead.

**C4 — Two checkers.** Run `mypy` and `pyright` on a real project with comparable
strictness. Catalogue every disagreement. For each, determine which is right by consulting
the typing specification, and file the difference as a note. Report the count and the
categories.

### Challenge

**X1.** Read Cardelli & Wegner, "On Understanding Types, Data Abstraction, and
Polymorphism" (1985), §1–3, and Liskov & Wing (1994). Write 1,200 words connecting their
framework to Python's: what is parametric polymorphism here, what is inclusion
polymorphism, where does structural typing fit, and what does Python's system fail to
express that theirs does?

**X2.** Implement a small typed effect-tracking pattern: a `Result[T, E]` type with
`map`, `and_then`, and exhaustive matching, generic in both parameters with correct
variance. Then use it in a real module and write 500 words comparing it to exceptions on:
type-level visibility of failure, ergonomics, and interaction with the standard library.
Take a position.

## 6. Self-check

1. State the substitution rule that defines subtyping.
2. Give the input/output rule for variance and derive `Callable`'s variance from it.
3. Why is `list[T]` invariant and `Sequence[T]` covariant?
4. State the rule for choosing between a Protocol and an ABC.
5. What does `@runtime_checkable` actually check?
6. What problem does `ParamSpec` solve, and what was the situation before it?
7. What does `py.typed` do and what happens without it?
8. Explain why splitting a repository into read and write interfaces changes its variance.

## 7. Primary sources

- PEP 483, 484, 544 (Protocols), 612 (`ParamSpec`), 646 (`TypeVarTuple`), 673 (`Self`),
  695 (type parameter syntax), 561 (`py.typed`).
- The Python *Typing Specification* (typing.readthedocs.io/en/latest/spec/) — now the
  authoritative document; the variance and protocol chapters especially.
- Cardelli & Wegner, "On Understanding Types, Data Abstraction, and Polymorphism"
  (Computing Surveys, 1985).
- Liskov & Wing, "A Behavioral Notion of Subtyping" (TOPLAS, 1994).

---

**Previous:** [L06](L06-gradual-typing.md) · **Next:**
[L08 — Static Analysis Beyond Types](L08-static-analysis.md)
