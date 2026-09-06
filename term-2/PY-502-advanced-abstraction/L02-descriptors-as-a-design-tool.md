# PY-502 · Lesson 02 — Descriptors as a Design Tool

**Estimated study time:** 4 hours
**Prerequisites:** PY-501 L03, L05

---

## 1. Orientation

PY-501 L03 established the mechanism. This lesson is about the *design question*: given
that descriptors exist, when should you write one?

The honest answer is "rarely" — but the rare cases are important, and the reasoning
generalizes. A descriptor is the only way to attach behaviour to *attribute access itself*,
which means it is the right tool exactly when the behaviour belongs to the attribute rather
than to the object or to the caller.

## 2. Theory

### 2.1 The decision ladder

Before writing a descriptor, walk down this list and stop at the first thing that works:

1. **A plain attribute.** If there is no behaviour, there is nothing to attach.
2. **A `property`.** One attribute, one class, computed or validated. Cheap, obvious,
   readable, universally understood.
3. **A `dataclass` field with a validator in `__post_init__`.** Validation at construction
   only, which is usually all you need for an immutable value object.
4. **`__setattr__` on the class.** Cross-cutting behaviour over *all* attributes (freezing,
   change tracking, audit).
5. **A descriptor.** Behaviour attached to *one kind of attribute*, reused across many
   attributes and many classes.
6. **A metaclass.** Only if you also need to control class creation (L03).

The distinguishing question for step 5: **would I write this `property` more than three
times?** A descriptor's entire value proposition is factoring out repeated attribute
behaviour. One `Typed` descriptor replacing twelve near-identical properties is a clear win;
one descriptor used once is a `property` with extra ceremony and worse discoverability.

### 2.2 Where the state lives — the four strategies

The central design decision, with real consequences. Assume a descriptor named `x` on class
`C`, instance `c`.

**(a) `c.__dict__` under a mangled key** (`c.__dict__["_x"]`).

- Fast, picklable, `copy`-able, visible in `vars(c)`.
- Breaks with `__slots__` (no `__dict__`) unless the slot is declared.
- The key is reachable: `c._x = "bypass"` skips validation.
- **Default choice.** Use it unless something rules it out.

**(b) `c.__dict__` under the *same* name** — only for non-data descriptors.

- The descriptor fires once, writes `c.__dict__[name]`, and is thereafter bypassed by step
  3 of the lookup algorithm.
- This is `functools.cached_property`. Zero cost after first access.
- Cannot validate on write (there is no `__set__`), cannot invalidate without deleting the
  dict entry.

**(c) A `WeakKeyDictionary` on the descriptor.**

- Keeps everything inside the descriptor; nothing visible on the instance.
- **Three real hazards**: instances must be hashable and weak-referenceable; keying uses
  `__hash__`/`__eq__`, so two *equal* instances share state (catastrophic for value
  objects); and pickling loses the data entirely.
- Correct only for identity-semantics objects you control. Rarely worth it.

**(d) `__set_name__` plus a declared slot.**

```python
class Typed:
    def __set_name__(self, owner, name):
        self.name = name
        self.slot = f"__{name}"
    def __get__(self, obj, objtype=None):
        if obj is None: return self
        return getattr(obj, self.slot)
    def __set__(self, obj, value):
        validate(value)
        setattr(obj, self.slot, value)

class Point:
    __slots__ = ("__x", "__y")     # note: name-mangling applies in the class body
    x = Typed(float)
    y = Typed(float)
```

- Works with `__slots__`, memory-efficient, still picklable via `__getstate__`.
- Fiddly: name mangling in class bodies makes the slot names surprising, and the
  descriptor and the `__slots__` declaration must be kept in sync (which is exactly what
  a metaclass or `__init_subclass__` would automate — L03).

### 2.3 `__set_name__` and self-naming

PEP 487's `__set_name__(self, owner, name)` is called by `type.__new__` for every object in
the class namespace that defines it. It is what makes descriptors ergonomic:

```python
class Field:
    def __set_name__(self, owner: type, name: str) -> None:
        self.name = name
        self.owner = owner
        owner._fields = {**getattr(owner, "_fields", {}), name: self}
```

Two things to know:

- It runs **only** for objects present in the class body at creation time. Assigning a
  descriptor to a class *afterwards* (`C.z = Typed(int)`) does **not** call `__set_name__`,
  and your descriptor will have no name. If you support dynamic assignment, call it
  manually.
- It runs before the class object is fully returned, so the class exists but decorators
  applied to it have not run yet.

### 2.4 Descriptors and inheritance

A descriptor lives in one class's `__dict__` and is found via the MRO. Consequences:

- **Subclasses share the descriptor instance.** Any state on the descriptor is shared
  across the whole hierarchy — which is why strategy (a)/(d) storage per-instance is
  essential.
- **A subclass can shadow a descriptor** with a plain value, and the plain value wins for
  that subclass (it is earlier in the MRO). This is occasionally useful and usually a bug;
  a `__init_subclass__` check can forbid it.
- **`super()` does not help.** There is no "call the parent descriptor" idiom;
  if you need composition, hold the inner descriptor explicitly:

```python
class Logged:
    def __init__(self, inner): self.inner = inner
    def __set_name__(self, owner, name):
        self.name = name
        if hasattr(self.inner, "__set_name__"):
            self.inner.__set_name__(owner, name)
    def __get__(self, obj, objtype=None):
        if obj is None: return self
        v = self.inner.__get__(obj, objtype)
        log.debug("read %s -> %r", self.name, v)
        return v
    def __set__(self, obj, value):
        log.debug("write %s <- %r", self.name, value)
        self.inner.__set__(obj, value)
```

Descriptors compose by *wrapping*, not by inheritance. This is a good example of the
general principle (PY-501 L04 §2.7) and it works cleanly here.

### 2.5 Interactions you must get right

**`__slots__`** — strategy (a) fails; use (d) or document the restriction.

**Pickling** — strategies (a) and (d) survive if the state is in `__dict__`/slots;
strategy (c) does not. Also: unpickling does not call `__init__` (PY-501 L06 §2.7), so
validation performed in `__set__` *is* re-run on `__setstate__`'s default behaviour only if
`__setstate__` uses `setattr`. The default implementation updates `__dict__` directly,
**bypassing your descriptor**. If validation on load matters, implement `__setstate__`
explicitly.

**`copy` / `deepcopy`** — same analysis.

**`dataclasses`** — a descriptor as a field default has specific, surprising semantics: the
dataclass machinery calls `descriptor.__get__(None, cls)` to obtain the default, and the
generated `__init__` assigns through the descriptor. This works, and the interaction is
worth reading the `dataclasses` documentation section on descriptor-typed fields before
relying on it.

**`__init_subclass__`** — the natural partner: use it to collect the descriptors declared in
a subclass and build a schema, generate `__slots__`, or verify none was shadowed (L03).

**Class-level access** — `C.x` calls `__get__(None, C)`. Returning `self` is conventional
and lets introspection tools find the descriptor. Returning something else (a "field
descriptor object" describing the field) is a design choice some ORMs make so that
`Model.name == "x"` builds a query expression — powerful, and it means `C.x` and `c.x` have
completely different types, which confuses type checkers and readers alike. If you do it,
document it loudly.

### 2.6 Typing descriptors

The checkers understand descriptors, but you must be explicit:

```python
from typing import overload, Any, Self

class Typed[T]:
    def __init__(self, type_: type[T]) -> None: ...

    @overload
    def __get__(self, obj: None, objtype: type) -> "Typed[T]": ...
    @overload
    def __get__(self, obj: object, objtype: type | None = None) -> T: ...
    def __get__(self, obj, objtype=None): ...

    def __set__(self, obj: object, value: T) -> None: ...
```

The overload pair is what tells the checker that `C.x` is the descriptor and `c.x` is a
`T`. Without it, every access is `Any` and the whole class loses its typing.

PEP 681 (`@dataclass_transform`) is the mechanism that lets `attrs`, `pydantic`, and your
own field framework tell type checkers "this decorator generates an `__init__` from these
fields". If you build a declarative framework (Problem Set 1), you need it or your users
get no completion and no checking.

## 3. Construction: a validated, typed, introspectable field system

Build it in three passes. The goal is a system where this works and type-checks:

```python
class Account(Record):
    name  = Str(min_len=1, max_len=80)
    email = Str(pattern=EMAIL_RE)
    limit = Int(min_value=0, default=1000)

a = Account(name="Mike", email="m@example.com")
a.limit                # 1000, typed int
a.limit = -5           # ValueError
Account.fields()       # {'name': Str(...), 'email': Str(...), 'limit': Int(...)}
```

**Pass 1 — the descriptors.** A `Field` base with `__set_name__`, storage strategy (a),
`__get__`/`__set__`, and a `validate` hook. Subclasses `Str`, `Int`, `Decimal`, `Enum`.

Design decisions to make and record:

- **Where does the default live?** On the descriptor (shared, so it must be immutable or a
  factory) or generated into `__init__`? Choose and say why. (Hint: PY-501 L01 §4.1 is the
  same problem.)
- **Is a missing value an `AttributeError` or a `None`?** Choose. `AttributeError` is
  honest; `None` is convenient and starts an infection of optional types.
- **Does validation run on read?** No. But say why, because "we only validate on write"
  fails if anything bypasses `__set__` (strategy (a)'s private key, `__setstate__`, direct
  `__dict__` manipulation).

**Pass 2 — the class hook.** `Record.__init_subclass__` collects fields, builds
`cls._fields`, generates an `__init__` accepting them as keyword arguments, generates
`__repr__` and `__eq__`. No metaclass.

The `__init__` generation is the interesting part. Two approaches:

```python
# (a) a closure — simple, but the signature is (*args, **kwargs)
def make_init(fields):
    def __init__(self, **kw):
        for name, f in fields.items():
            setattr(self, name, kw.pop(name, f.default))
        if kw: raise TypeError(f"unexpected: {sorted(kw)}")
    return __init__

# (b) generated source, exec'd — a real signature, better tracebacks, better introspection
def make_init(fields):
    args = ", ".join(f"{n}={n}_default" for n in fields)
    src = f"def __init__(self, *, {args}):\n" + "".join(
        f"    self.{n} = {n}\n" for n in fields)
    ns: dict[str, object] = {f"{n}_default": f.default for n, f in fields.items()}
    exec(src, ns)
    return ns["__init__"]
```

(b) is what `dataclasses` does, and PY-501 L07 §2.8 explains why the `exec`-with-explicit-
namespace shape is necessary. Implement both, then compare: `inspect.signature`, the
traceback on a bad call, IDE completion, and import-time cost. That comparison is the
deliverable, not the code.

**Pass 3 — typing.** Add the `__get__` overloads of §2.6 and `@dataclass_transform` on
`Record`, so that `Account(name=..., email=...)` type-checks and `a.limit` is `int`.
Verify with both mypy and pyright — they diverge here, which is instructive.

Then the closing question: **you have just rebuilt a worse `dataclasses` / `attrs` /
`pydantic`.** What, specifically, does yours do that they do not? If the answer is
"nothing", that is the correct and valuable finding, and L08 will make the comparison
properly. Building it once is how you earn the right to choose between them.

## 4. Failure modes

- **State on the descriptor.** Shared across all instances. The first bug everyone writes.
- **`WeakKeyDictionary` with `__eq__`-defining classes.** Equal instances share state.
- **Forgetting `obj is None`.** Breaks `help()`, `inspect`, and every documentation tool.
- **Assuming `__set_name__` runs for post-hoc assignment.** It does not.
- **Validation bypassed by `__setstate__`.** §2.5.
- **A descriptor used once.** Should be a `property`.
- **Untyped `__get__`.** Every attribute becomes `Any` and the class silently loses all
  checking.
- **`cached_property` on a slotted class.** Fails, and cannot be fixed.
- **`cached_property` concurrency.** Two threads may both compute (PY-601 L05).
- **Descriptors that make `C.x` and `c.x` different types** without documenting it.

## 5. Exercises

### Warm-up (25 min)

**W1.** Write the shared-state bug of §4 and demonstrate it. Then fix it with each of
strategies (a), (c), and (d), and state one thing each fix broke.

**W2.** Show that `__set_name__` is not called when a descriptor is assigned to a class
after creation. Write the workaround.

**W3.** Add the `__get__` overloads to a descriptor and show, with mypy, the difference in
what the checker knows before and after.

### Core (2.5 h)

**C1 — The field system.** Complete §3, all three passes. Deliverable: the code, tests, both
`__init__` generation strategies with the comparison table, mypy and pyright both clean, and
a 600-word note answering the closing question honestly.

**C2 — Descriptor composition.** Implement `Logged`, `Cached`, and `Deprecated` wrapper
descriptors that compose over any inner descriptor, in any order. Write tests proving order
independence where it should hold and documenting where it does not. Then explain why
composition-by-wrapping works here where inheritance would not.

**C3 — Round-trip integrity.** Take your field system and make it survive `copy`,
`deepcopy`, and `pickle` with validation re-applied. Write property tests
(SE-511 L04) asserting `loads(dumps(x)) == x` and that a corrupted pickle raises rather
than producing an invalid object. Report every mechanism you had to implement and why.

**C4 — Read the source.** Read `functools.cached_property`, `dataclasses`' descriptor-typed
field handling, and `property`'s pure-Python equivalent from the Descriptor HowTo. Write
800 words on the design decisions each makes about storage, typing, and thread safety, and
where they disagree with each other.

### Challenge

**X1.** Implement `AutoSlots`: an `__init_subclass__` that generates `__slots__` from the
declared fields, handling inheritance (no duplicates), the name-mangling issue of §2.2(d),
and the class-attribute collision. Then explain why `dataclass(slots=True)` returns a *new
class* and construct a program where that distinction is observable.

**X2.** Build a descriptor-based change-tracking system: an object records which fields were
modified since load, for use in generating minimal `UPDATE` statements. Handle: nested
objects, collections (which mutate without `__set__` firing — this is the hard part),
rollback, and concurrency. Write up what you could not solve and why. The collections
problem is genuinely hard and how ORMs solve it (instrumented collection types) is worth
discovering yourself first.

## 6. Self-check

1. Give the six-step decision ladder and the question that selects a descriptor.
2. Name the four storage strategies with one fatal flaw each.
3. Why is `cached_property` deliberately a non-data descriptor?
4. When is `__set_name__` *not* called?
5. Why do descriptors compose by wrapping rather than inheritance?
6. What bypasses `__set__` validation, and what do you do about it?
7. What do the `__get__` overloads tell the type checker, and what happens without them?
8. What does `@dataclass_transform` do and who needs it?

## 7. Primary sources

- Hettinger, "Descriptor HowTo Guide" — the pure-Python equivalents especially.
- PEP 487 (`__set_name__`), PEP 681 (`@dataclass_transform`).
- CPython `Objects/typeobject.c`, `type_new`'s `__set_name__` loop.
- `dataclasses` source — the field/descriptor interaction and `_create_fn`.

---

**Previous:** [L01](L01-duck-typing-protocols-interface-design.md) · **Next:**
[L03 — Class Construction Hooks and Metaclasses](L03-class-construction-and-metaclasses.md)
