# PY-502 · Lesson 01 — Duck Typing, Protocols, and Interface Design

**Estimated study time:** 3.5 hours
**Prerequisites:** PY-501 L02, L06; SE-511 L07

---

## 1. Orientation

Python has three ways to say "this thing must support these operations":

```python
def render(f):                              # 1. duck typing — a convention
    f.write(header())

def render(f: SupportsWrite[str]):          # 2. Protocol — structural, checked statically
    f.write(header())

def render(f: Writer):                      # 3. ABC — nominal, enforced at instantiation
    f.write(header())
```

All three work. They differ in *who bears the cost* and *when the error appears*, and those
differences determine which is right. Choosing by habit rather than by criteria is how
codebases end up with an ABC hierarchy nobody can extend and a set of duck-typed functions
nobody can call safely.

## 2. Theory

### 2.1 What an interface is for

An interface exists to let a *consumer* state its requirements without naming a *provider*.
That framing settles most of the design questions that follow.

Two properties of a good interface:

**Narrow.** It demands as little as possible. A function that needs only `write` should
require only `write`, not `TextIOBase`. Every additional requirement excludes a possible
implementation — including your test double, which is why over-wide interfaces show up
first as testing pain (SE-511 L01 §2.4).

**Deep.** Ousterhout's term: the ratio of functionality provided to interface surface
exposed. A deep module has a small interface hiding substantial implementation. A shallow
module — a class with eight methods that each forward to one call — costs more in
comprehension than it saves.

These pull in opposite directions and the tension is real. Narrow says *require less*; deep
says *provide more per unit of surface*. The resolution: narrow the *requirements* you place
on collaborators; deepen the *services* you offer to callers.

### 2.2 Duck typing: what it actually gives up

```python
def process(source):
    for line in source:
        yield line.strip().upper()
```

Works on a file, a list, a generator, a socket wrapper, anything iterable of strings. That
generality is real value: `process` will work with types that did not exist when it was
written.

What it gives up:

- **No documentation of the requirement.** A reader must read the body to learn that
  `source` must be iterable and yield objects with `strip` and `upper`.
- **No checking.** The failure appears at runtime, deep in the loop, possibly in production,
  possibly after side effects.
- **Poor error messages.** `AttributeError: 'int' object has no attribute 'strip'` names
  neither the caller nor the requirement.
- **Fragile evolution.** Adding a requirement (`source.name` for an error message) silently
  breaks callers whose objects lack it.

Duck typing remains correct for small, local, private functions where the cost of
formalization exceeds the benefit. It is wrong at module and package boundaries.

### 2.3 Protocols: structural typing, checked

```python
from typing import Protocol

class LineSource(Protocol):
    def __iter__(self) -> Iterator[str]: ...
    @property
    def name(self) -> str: ...

def process(source: LineSource) -> Iterator[str]:
    ...
```

This keeps duck typing's essential property — no provider needs to know about you, and
types written before yours can satisfy it — while adding a checked, documented statement of
the requirement.

The property that makes Protocols architecturally significant: **the consumer defines the
interface, in the consumer's module.** The dependency arrow points from provider to
interface, and the interface lives with the code that needs it. With an ABC, the consumer
must import the ABC from somewhere, and every provider must import it too — a shared
dependency that is exactly what you were trying to avoid.

Concretely, in a layered application:

```python
# domain/ports.py — owned by the domain, importing nothing from infrastructure
class OrderRepository(Protocol):
    def get(self, id: OrderId) -> Order | None: ...
    def save(self, order: Order) -> None: ...

# infrastructure/postgres.py — imports domain, domain imports nothing from here
class PostgresOrderRepository:
    def get(self, id: OrderId) -> Order | None: ...
    def save(self, order: Order) -> None: ...
```

`PostgresOrderRepository` does not inherit from anything and does not import `ports.py`.
The type checker verifies the match at the wiring point. This is the Dependency Inversion
Principle with no runtime coupling at all, and it is the single most useful application of
`Protocol` in real systems.

**Practical details:**

- Protocols may include attributes, properties, class variables, and other Protocols.
- A *mutable* attribute in a Protocol makes it invariant in that attribute's type (SE-511
  L07 §2.2). Prefer read-only properties in Protocols unless mutation is genuinely part of
  the contract.
- `@runtime_checkable` enables `isinstance`, checking **method presence only** — not
  signatures, not attributes for non-data protocols. It is a smoke test, not verification.
- Protocols can have default method implementations, usable only by explicit subclasses.
  This is a legitimate hybrid but it blurs the "consumer-defined" property; use sparingly.
- A Protocol in a *return* position is almost always a mistake: you know the concrete type,
  so promise it.

### 2.4 ABCs: when nominal is right

```python
import abc

class Codec(abc.ABC):
    @abc.abstractmethod
    def encode(self, obj: object) -> bytes: ...

    @abc.abstractmethod
    def decode(self, data: bytes) -> object: ...

    def encode_many(self, objs: Iterable[object]) -> bytes:
        return b"".join(self.encode(o) for o in objs)      # shared implementation
```

Use an ABC when:

- **You have implementation to share.** `encode_many` is written once. A Protocol cannot
  give you this without the hybrid form.
- **You want instantiation-time failure.** `TypeError: Can't instantiate abstract class` at
  construction beats `AttributeError` at first use, and beats a type error nobody ran the
  checker to see.
- **The relationship is genuinely an "is-a"** and you want it visible in `__mro__`, in
  `isinstance` checks, and in documentation.
- **You are defining a taxonomy for others to extend**, and the explicit inheritance
  declaration is useful information ("this class intends to be a Codec").

Costs: nominal coupling (every implementer imports and inherits from you), a metaclass
(`ABCMeta`) that does not compose with other metaclasses (PY-501 L02 §3), and the
inheritance-for-reuse hazard (PY-501 L04 §2.7) if the shared methods grow.

### 2.5 The decision rule

> **Protocol for what you consume. ABC for what you share. Duck typing for what is small
> and local.**

Expanded, with the questions that actually decide it:

| Question | If yes |
|---|---|
| Do I need to share concrete implementation? | ABC |
| Must implementers be unable to *forget* something? | ABC (instantiation error) |
| Will implementations be written by people who cannot import my package? | Protocol |
| Do I want existing third-party types to satisfy it? | Protocol |
| Is this a boundary between architectural layers? | Protocol, defined in the inner layer |
| Is this a five-line private helper? | Duck typing |
| Do I need `isinstance` dispatch at runtime? | ABC (or `runtime_checkable`, weakly) |

The standard library uses both, correctly: `collections.abc` classes are ABCs *and* provide
`__subclasshook__` so structural matching works too — the best of both, at the cost of
implementation complexity you do not need to replicate.

### 2.6 Single dispatch: interfaces without inheritance

A third form of polymorphism, often forgotten:

```python
from functools import singledispatch

@singledispatch
def render(node: object) -> str:
    raise TypeError(f"no renderer for {type(node).__name__}")

@render.register
def _(node: Paragraph) -> str: return f"<p>{node.text}</p>"

@render.register
def _(node: Heading) -> str: return f"<h{node.level}>{node.text}</h{node.level}>"
```

This puts the operation *outside* the types. Its value: you can add operations to types you
do not own, and you can keep a rendering concern out of the domain model entirely.

Its cost: you cannot add a *type* without touching the dispatch site's module — the
converse of the class hierarchy trade-off. This is the **expression problem** (CS-641 L06):
class-based polymorphism makes adding types easy and adding operations hard; functional
dispatch makes adding operations easy and adding types hard. Neither is better; choose based
on which axis your system actually varies along.

`singledispatchmethod` provides the same for methods. Note that dispatch is on the *runtime
type of the first argument only*, using the MRO — so it interacts with inheritance in the
expected way and cannot dispatch on a Protocol unless the Protocol is `runtime_checkable`
and registered explicitly.

### 2.7 Interface evolution

An interface is a promise, and the question that determines its design is *how will this
change?*

- **Adding a method to a Protocol** breaks every implementer at check time. Mitigate by
  splitting into small Protocols and composing (`class RW(Readable, Writable, Protocol)`),
  which is the interface segregation principle arrived at from the evolution direction.
- **Adding a method to an ABC** breaks implementers at *instantiation* time if abstract, or
  not at all if it has a default. That is a real advantage of ABCs for interfaces you expect
  to grow.
- **Adding a parameter** breaks implementers either way unless it is keyword-with-default.
  Design methods with keyword-only parameters after the essential ones, so that additions
  are non-breaking.
- **Optional capabilities** are better expressed as a separate Protocol tested with
  `isinstance` (or `hasattr`) than as a method that some implementations raise
  `NotImplementedError` from — the latter moves the failure to run time and gives the caller
  no way to check first.

## 3. Construction: an interface, four ways

Take one requirement — "something that can store and retrieve blobs by key" — and build it
four ways, then compare.

**1. Duck typing.**

```python
def cache_result(store, key: str, compute: Callable[[], bytes]) -> bytes:
    try:
        return store.get(key)
    except KeyError:
        value = compute()
        store.put(key, value)
        return value
```

Two lines of requirement, undocumented. Note the subtlety: this also requires that `get`
raise `KeyError` and not return `None` — a *behavioural* requirement that no mechanism in
this lesson can express. Hold that thought.

**2. Protocol.**

```python
class BlobStore(Protocol):
    def get(self, key: str) -> bytes: ...
    def put(self, key: str, value: bytes) -> None: ...
```

Now the shape is documented and checked. The `KeyError` requirement is still only in the
docstring.

**3. ABC with shared behaviour.**

```python
class BlobStore(abc.ABC):
    @abc.abstractmethod
    def get(self, key: str) -> bytes: ...
    @abc.abstractmethod
    def put(self, key: str, value: bytes) -> None: ...

    def get_or_default(self, key: str, default: bytes) -> bytes:
        try:
            return self.get(key)
        except KeyError:
            return default

    def get_many(self, keys: Iterable[str]) -> dict[str, bytes]:
        return {k: v for k in keys if (v := self._maybe(k)) is not None}
```

The shared methods are the argument. If there are none, the ABC is pure cost.

**4. The behavioural contract, as a test.** The thing none of the above expresses:

```python
class BlobStoreContract:
    @pytest.fixture
    def store(self) -> BlobStore: raise NotImplementedError

    def test_get_after_put_returns_value(self, store): ...
    def test_get_missing_raises_keyerror(self, store): ...
    def test_put_overwrites(self, store): ...
    def test_keys_are_case_sensitive(self, store): ...
    def test_empty_value_is_allowed(self, store): ...
```

**This is the real interface.** The `Protocol` states the shape; the contract test states
the behaviour. Both are needed and only one of them is in the type system. Every
implementation — including the in-memory fake — subclasses the contract (SE-511 L02 §2.6).

The conclusion to internalize: *Python's type system can express structure, not behaviour.*
Liskov substitutability is a behavioural property, and no annotation checks it. The contract
test is how you check it, and a codebase with Protocols but no contract tests has done half
the job.

## 4. Failure modes

- **ABC by default.** Reflexively inheriting from `abc.ABC` for interfaces with no shared
  implementation. Pure cost: a metaclass, a nominal dependency, no benefit.
- **Fat interfaces.** A `Repository` Protocol with fourteen methods that no test double can
  reasonably implement. Segregate.
- **Protocols in return positions.** §2.3.
- **`runtime_checkable` as verification.** Method names only.
- **`NotImplementedError` for optional capability.** §2.7.
- **Duck typing at package boundaries.** The error surfaces in the caller's code with no
  indication of the requirement.
- **Forgetting the behavioural contract.** §3 (4). The most consequential item here.
- **Protocol with mutable attributes.** Invariance where you wanted covariance.
- **`singledispatch` on a type you will add to often.** Wrong side of the expression
  problem.

## 5. Exercises

### Warm-up (25 min)

**W1.** Take three functions from your code that duck-type their arguments. Write a Protocol
for each. Report anything you discovered about the requirement that you had not noticed.

**W2.** Show that `@runtime_checkable` `isinstance` passes for a class with the right method
names but wrong signatures.

**W3.** Convert an ABC with no shared implementation to a Protocol. Report every import that
could then be deleted.

### Core (2 h)

**C1 — Four ways.** Complete §3 for an interface in your own system. Deliverable: all four
artifacts, two implementations passing the contract, and a 500-word note applying the §2.5
decision table and stating which you would ship.

**C2 — Segregate a fat interface.** Find an interface with more than six methods. Split it
into small Protocols by *who uses what* (build the usage matrix first: methods × call
sites). Report the matrix and the resulting decomposition. Then measure: how much smaller is
the smallest test double you now need?

**C3 — Invert a dependency.** Take a module that imports a concrete class from an outer
layer. Define a Protocol in the importing module, remove the import, and wire at the
composition root. Verify with `import-linter` (SE-511 L08) that the layer rule now holds.

**C4 — The expression problem, empirically.** Implement a small AST evaluator two ways:
class hierarchy with visitor methods, and `singledispatch` functions. Then perform two
changes to each: add a node type, and add an operation. Report the diff size for each of the
four combinations. Conclude with the rule you would give a colleague.

### Challenge

**X1.** Read PEP 544 in full, plus the *Typing Specification*'s protocol chapter. Implement
a `check_protocol(cls, proto) -> Report` that verifies structurally *including signatures*
(use `inspect.signature` and compare parameter kinds, names, and annotations). Report what
you cannot check and why. Compare against mypy's verdict on 20 cases.

**X2.** Take a well-known library interface (`collections.abc.MutableMapping`,
`io.IOBase`, `sqlalchemy`'s dialect API). Write its behavioural contract as a test suite,
from the documentation. Run it against two real implementations. Report every divergence —
there will be several, and each is either a documentation bug or an implementation bug.

## 6. Self-check

1. Give the two properties of a good interface and explain the tension between them.
2. Name four things duck typing gives up.
3. Why does a Protocol let the consumer own the interface, and why does that matter
   architecturally?
4. Give four criteria that select an ABC over a Protocol.
5. What does `@runtime_checkable` check?
6. State the expression problem and say which polymorphism suits which axis of change.
7. Why is `NotImplementedError` for optional capability a poor design?
8. What can Python's type system not express about an interface, and what fills the gap?

## 7. Primary sources

- PEP 544 (Protocols) and PEP 3119 (ABCs). Read 544's rationale.
- Ousterhout, *A Philosophy of Software Design*, ch. 4 ("Modules Should Be Deep") and ch. 6.
- Wadler, "The Expression Problem" (1998 email, one page).
- Liskov & Wing (1994) — the behavioural contract, again.

---

**Next:** [L02 — Descriptors as a Design Tool](L02-descriptors-as-a-design-tool.md)
