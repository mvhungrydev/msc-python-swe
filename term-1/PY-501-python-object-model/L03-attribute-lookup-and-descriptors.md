# PY-501 · Lesson 03 — Attribute Lookup and the Descriptor Protocol

**Estimated study time:** 4 hours
**Prerequisites:** L01, L02

---

## 1. Orientation

Answer this before reading further:

```python
class C:
    x = 1

c = C()
c.__dict__["x"] = 2
print(c.x)          # ?

class D:
    @property
    def x(self): return 1

d = D()
d.__dict__["x"] = 2
print(d.x)          # ?
```

The first prints `2`. The second prints `1`. Same shape, opposite answer. If that seems
arbitrary, you are missing one rule — the data-descriptor priority rule — and it is the
rule that makes `property`, `classmethod`, `staticmethod`, `__slots__`, bound methods,
`functools.cached_property`, SQLAlchemy columns, Django fields, and every ORM's field
system work.

This lesson is the most mechanically important in the course. Attribute access is the
operation Python performs more than any other, and almost every framework you will ever
use customizes it.

## 2. Theory

### 2.1 The algorithm

Evaluating `obj.name` compiles to `LOAD_ATTR`, which calls
`type(obj).__getattribute__(obj, "name")`. For an object whose type does not override it,
that is `object.__getattribute__`, whose algorithm is:

```
1. meta_attr ← search type(obj).__mro__ for "name"        (class-level lookup)
2. if meta_attr is a DATA descriptor:
       return meta_attr.__get__(obj, type(obj))
3. if obj has a __dict__ and "name" in obj.__dict__:
       return obj.__dict__["name"]                        (instance dict wins here)
4. if meta_attr was found:
       if it is a NON-DATA descriptor:
            return meta_attr.__get__(obj, type(obj))
       else:
            return meta_attr                              (plain class attribute)
5. raise AttributeError
```

and *then*, outside `__getattribute__`, if step 5 raised:

```
6. if type(obj) defines __getattr__:
       return type(obj).__getattr__(obj, "name")
```

Commit this to memory. Nearly every attribute-related question in Python is answered by
locating the case in this list.

Definitions:

- A **descriptor** is any object whose *type* defines `__get__`.
- It is a **data descriptor** if its type also defines `__set__` or `__delete__`.
- Otherwise it is a **non-data descriptor**.

The priority order is therefore: **data descriptor → instance dict → non-data descriptor /
class attribute → `__getattr__`**. Re-read the opening puzzle: `property` defines
`__set__` (it raises `AttributeError` when there is no setter, but it *defines* it), so it
is a data descriptor and wins over the instance dict. A plain `1` is not a descriptor at
all, so the instance dict wins.

### 2.2 The descriptor protocol

```python
class Descriptor:
    def __set_name__(self, owner: type, name: str) -> None: ...   # called at class creation
    def __get__(self, obj, objtype=None): ...
    def __set__(self, obj, value) -> None: ...
    def __delete__(self, obj) -> None: ...
```

`__get__` receives `obj=None` when accessed on the *class* rather than an instance
(`C.x` rather than `c.x`). Handling that case is mandatory in practice — every
introspection tool, `help()`, and `inspect` will hit it.

`__set_name__` (PEP 487) is called by `type.__new__` for every descriptor in the class
namespace, passing the owning class and the attribute name. Before it existed, descriptors
had to be told their own name redundantly (`x = Field("x")`) or use a metaclass. Use it.

A complete, minimal, correct data descriptor:

```python
from typing import Any, overload

class Typed:
    """A validating attribute."""

    def __init__(self, expected: type) -> None:
        self.expected = expected

    def __set_name__(self, owner: type, name: str) -> None:
        self.name = name
        self.private = "_" + name          # where we actually store it

    def __get__(self, obj: Any, objtype: type | None = None) -> Any:
        if obj is None:
            return self                     # class access: return the descriptor itself
        try:
            return getattr(obj, self.private)
        except AttributeError:
            raise AttributeError(f"{self.name!r} is not set") from None

    def __set__(self, obj: Any, value: Any) -> None:
        if not isinstance(value, self.expected):
            raise TypeError(f"{self.name} must be {self.expected.__name__}")
        setattr(obj, self.private, value)
```

### 2.3 Where to store the value

The `Typed` descriptor above stores into `obj._name`. There are three standard strategies
and each has a real cost:

**Instance dict under a different key** (as above). Simple, fast, works with `__slots__`
only if you also declare the private slot. Downside: the private name is visible and
mutable; a caller can bypass validation with `obj._x = "bad"`.

**A `WeakKeyDictionary` on the descriptor.** Keeps per-instance state entirely inside the
descriptor. Downside: requires the instances to be hashable and weak-referenceable, adds a
dict lookup, and — the subtle one — uses `__eq__`/`__hash__`, so two *equal* instances
collide. `WeakKeyDictionary` keys by hash and equality, not identity. For a value-like
class with `__eq__` defined, this silently shares state between distinct objects. This bug
is nasty and common.

**The instance dict under the same name.** Only possible for *non-data* descriptors: the
descriptor computes once, writes `obj.__dict__[name]`, and thereafter step 3 of the
algorithm short-circuits it. This is exactly `functools.cached_property`:

```python
class cached_property:
    def __init__(self, func): self.func = func
    def __set_name__(self, owner, name): self.name = name
    def __get__(self, obj, objtype=None):
        if obj is None: return self
        value = self.func(obj)
        obj.__dict__[self.name] = value    # subsequent accesses never reach here
        return value
```

Notice it defines no `__set__`. That is not an oversight — it is the mechanism. Adding a
`__set__` would make it a data descriptor and it would intercept every access forever,
destroying the caching. Consequently `cached_property` does not work on classes with
`__slots__` and no `__dict__`, and this is not a bug that can be fixed.

### 2.4 Functions are descriptors: how methods work

This is the payoff. `function` defines `__get__`:

```python
class C:
    def f(self): ...

C.f            # <function C.f at 0x...>   — obj is None, __get__ returns the function
C().f          # <bound method C.f of ...> — __get__ returns a bound method
```

`function.__get__(obj, objtype)` returns a `MethodType(func, obj)` when `obj` is not
`None`. A bound method is a tiny object holding `__func__` and `__self__`; calling it calls
`__func__(__self__, *args)`.

So: **there is no such thing as a method in Python.** There are functions stored in class
dicts, and a non-data descriptor protocol that binds them on access. Everything follows:

- `c.f` creates a *new* bound method object each time. `c.f is c.f` is `False`. This is why
  you cannot use `is` to compare method references and why registering `c.f` as a callback
  and later removing it by identity fails.
- Assigning a function to an *instance* does not bind it:
  `c.g = lambda self: 1` then `c.g()` → `TypeError`, missing argument. Instance dict entries
  are returned as-is (step 3); no descriptor protocol runs. Monkey-patching an instance
  requires `types.MethodType(func, c)`.
- `staticmethod` is a descriptor whose `__get__` returns the underlying function
  unchanged. `classmethod` is one whose `__get__` binds to the *class* rather than the
  instance.
- Because functions are non-data descriptors, an instance attribute shadows a method. That
  is occasionally useful and usually a bug.

### 2.5 `__getattr__`, `__getattribute__`, `__setattr__`

`__getattr__` is a *fallback*, invoked only when normal lookup raised `AttributeError`. It
is cheap: no cost on the common path. Use it for proxies, lazy modules, and dynamic
attributes.

`__getattribute__` is invoked for *every* access. Overriding it is a serious act:

```python
class Loud:
    def __getattribute__(self, name):
        print("get", name)
        return object.__getattribute__(self, name)   # NOT self.__dict__[name] — infinite recursion
```

The recursion trap is the standard failure: any attribute access inside
`__getattribute__` re-enters it. Always delegate to `object.__getattribute__` or
`super().__getattribute__`.

`__setattr__` intercepts every attribute *store*, with the same recursion trap:

```python
class Frozen:
    def __setattr__(self, name, value):
        if getattr(self, "_initialized", False):
            raise AttributeError(f"{type(self).__name__} is immutable")
        object.__setattr__(self, name, value)
```

This is how `dataclasses.dataclass(frozen=True)` works — it generates a `__setattr__` that
raises, and its own `__init__` uses `object.__setattr__` to bypass it.

Note the asymmetry: there is a `__getattr__` fallback but **no `__setattr__` fallback**.
Stores always go through `__setattr__`, which by default consults data descriptors then
writes the instance dict.

### 2.6 Special methods bypass the instance

One rule that catches everyone: for **implicit** invocations of special methods, Python
looks them up on the *type*, not the instance, and skips `__getattribute__` entirely.

```python
class C: pass
c = C()
c.__len__ = lambda: 5
len(c)          # TypeError: object of type 'C' has no len()
```

`len(c)` compiles to a call through the type's `tp_as_sequence` slot; it never consults
`c.__dict__`. Same for `+`, `[]`, `with`, `in`, iteration, `repr` in f-strings, and the
rest. The reason is performance — these are C-level slot lookups — and the consequence is
that special methods must be defined on the class.

This also means `__getattr__` cannot conjure special methods. A proxy class that forwards
`__getattr__` to a wrapped object will forward `obj.foo` but **not** `len(obj)` or
`obj[0]`. Proxies must explicitly define every special method they intend to forward; this
is why `unittest.mock.MagicMock` exists as a separate class from `Mock`.

## 3. Construction: a unit-carrying attribute

Build a descriptor that stores a physical quantity, validates its unit, and converts on
read. Incrementally.

**Version 1 — naive, and broken.**

```python
class Unit:
    def __init__(self, unit): self.unit = unit
    def __get__(self, obj, objtype=None): return self.value
    def __set__(self, obj, value): self.value = value
```

```python
class Reading:
    temp = Unit("C")

a, b = Reading(), Reading()
a.temp = 20
b.temp = 30
a.temp          # 30  — WRONG
```

The descriptor is a **class** attribute: there is one instance of it, shared by every
`Reading`. Storing state on `self` (the descriptor) stores it per-class, not per-instance.
This is the first mistake everyone makes, and it is worth making once.

**Version 2 — per-instance storage via `__set_name__`.**

```python
class Unit:
    def __init__(self, unit: str) -> None:
        self.unit = unit

    def __set_name__(self, owner: type, name: str) -> None:
        self._store = f"__unit_{name}"

    def __get__(self, obj, objtype=None):
        if obj is None:
            return self
        try:
            return obj.__dict__[self._store]
        except KeyError:
            raise AttributeError(self._store) from None

    def __set__(self, obj, value) -> None:
        if not isinstance(value, (int, float)):
            raise TypeError("numeric required")
        obj.__dict__[self._store] = float(value)
```

Correct per-instance behaviour. Now the questions a graduate answer must address:

- **Does it work with `__slots__`?** No — there is no `__dict__`. Fix: fall back to
  `getattr/setattr` on a declared private slot, or document the restriction. Choose and
  justify.
- **Does it survive pickling?** The private key is in `__dict__`, so yes by default. But if
  you switch to a `WeakKeyDictionary` it does not. This is a real argument for the
  instance-dict strategy.
- **Does `Reading.temp` return something useful?** Currently the descriptor. Should it
  return a description of the unit instead? Whatever you choose, be consistent with what
  `help()` and `inspect.signature` expect.

**Version 3 — add conversion and make it composable.** Extend so `Reading(temp_f=68).temp`
returns 20.0 with `Unit("C", accepts={"F": lambda f: (f - 32) * 5 / 9})`. Then ask whether
the conversion belongs in the descriptor at all, or whether the descriptor should hold a
`Quantity` value object and stay dumb. Argue it in your write-up. (SE-521 L04 will give you
the vocabulary — this is a question about where behaviour belongs, not about descriptors.)

## 4. Failure modes

- **Per-descriptor state.** §3 version 1. The descriptor is shared.
- **`WeakKeyDictionary` with `__eq__`-defining classes.** Two equal instances share state.
  Use `WeakKeyDictionary` only for classes with identity semantics, or key on `id()` with
  a `weakref.finalize` cleanup.
- **Recursion in `__getattribute__` / `__setattr__`.** Always delegate to `object.`/`super().`
- **Expecting `__getattr__` to catch special methods.** §2.6.
- **`cached_property` on a slotted class.** Fails at write time with a confusing
  `AttributeError`. Also: `cached_property` is *not* thread-safe against duplicate
  computation — concurrent first accesses may both compute. That is usually fine and
  occasionally not (PY-601 L05).
- **Shadowing a method with an instance attribute.** Functions are non-data descriptors, so
  `obj.method = something` wins. Almost always a bug; a data descriptor would have
  prevented it.
- **Forgetting `obj is None`.** Every tool that introspects your class will hit class-level
  access.

## 5. Exercises

### Warm-up (25 min)

**W1.** For each of the five steps of §2.1, write a three-line program whose behaviour is
explained by that step and no other.

**W2.** Show experimentally that `c.f is c.f` is `False` for a method. Then explain what
`c.__class__.f is c.__class__.f` gives, and why.

**W3.** Implement `staticmethod` and `classmethod` yourself, in five lines each. Verify
against the built-ins on a class with inheritance.

### Core (2.5 h)

**C1 — An attribute-access tracer** *(this is Problem Set 1's centrepiece; start it here.)*
Write `trace_lookup(obj, name)` that reports, step by step, exactly which branch of the
§2.1 algorithm resolved the access: the class in the MRO where the attribute was found,
whether it was a data/non-data descriptor, whether the instance dict was consulted, and
whether `__getattr__` fired. Test against: a plain attribute, a `property`, a method, a
`staticmethod`, `__slots__`, a `cached_property` before and after first access, an
`__getattr__` fallback, and a class using `__getattribute__`.

**C2 — Validated fields without a metaclass.** Build a small field system: descriptors
`Int(min=, max=)`, `Str(pattern=)`, `Enum(choices=)`, plus a base class that uses
`__init_subclass__` to (a) collect the fields, (b) generate an `__init__` accepting them as
keyword arguments, (c) generate a `__repr__`. No metaclass. Compare your result with
`dataclasses` on: lines of code, error message quality, IDE/`mypy` support. Write 400 words
on what `dataclasses` gives up by being annotation-driven rather than descriptor-driven.

**C3 — A correct proxy.** Write `Proxy(target)` that forwards attribute access *and* the
special methods `len`, `getitem`, `iter`, `contains`, `str`, `repr`, `eq`, `hash`, `bool`,
and context-manager entry/exit. Then write a test that distinguishes your proxy from the
target in at least three observable ways, and document them — a perfect proxy is
impossible in CPython and knowing *how* it is impossible is the point.

**C4 — Thread-unsafe caching.** Demonstrate `functools.cached_property` computing twice
under concurrency. Then write a version that does not, and measure what the fix costs on
the uncontended path. State the conditions under which you would accept the cost.

### Challenge

**X1.** Implement `__getattribute__` for a class such that attribute lookup goes through a
*user-supplied* MRO order (e.g. reversed). Then explain, with reference to §2.6, exactly
which language features stop working and why.

**X2.** Read `Objects/object.c`, `_PyObject_GenericGetAttrWithDict`. Map each branch to a
step in §2.1. Identify one thing the C code does that the documented algorithm does not
mention, and write it up.

## 6. Self-check

1. State the six steps of attribute lookup in order.
2. Define data descriptor and non-data descriptor, and state the priority rule.
3. Why is `cached_property` deliberately *not* a data descriptor?
4. How does a bound method come to exist? What object does `c.f` actually return?
5. Why does assigning a function to an instance not create a method?
6. Give the recursion trap in `__getattribute__` and its fix.
7. Why does `len(c)` ignore `c.__dict__["__len__"]`?
8. Name two problems with storing descriptor state in a `WeakKeyDictionary`.

## 7. Primary sources

- Raymond Hettinger, "Descriptor HowTo Guide" (Python docs). The pure-Python equivalents
  it gives for `property`, `staticmethod` and `classmethod` are worth reading line by line.
- Language Reference §3.3.2 (Implementing descriptors) and §3.3.2.1 (Invoking descriptors).
- PEP 487 — `__set_name__`.
- Ramalho, *Fluent Python* 2e, ch. 23.

---

**Previous:** [L02](L02-types-instances-and-metatypes.md) · **Next:**
[L04 — Inheritance, `super()`, and the MRO](L04-inheritance-super-and-the-mro.md)
