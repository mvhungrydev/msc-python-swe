# PY-501 · Lesson 02 — Types, Instances, and Metatypes

**Estimated study time:** 3.5 hours
**Prerequisites:** L01

---

## 1. Orientation

```python
>>> type(3)
<class 'int'>
>>> type(int)
<class 'type'>
>>> type(type)
<class 'type'>
```

The regress terminates. `type` is its own type. That is not a cute fact — it is the load-
bearing structure of the whole object system, and once you can draw the diagram correctly
you will never again be confused by a metaclass error, an `isinstance` surprise, or the
question of why `object` and `type` seem to point at each other.

The other thing this lesson settles: **a class statement is not a declaration.** It is an
executable statement that runs code, builds a namespace, and calls a function. Everything
that looks like magic in class construction — `dataclasses`, ORMs, `Protocol`, `Enum`,
`abc` — is that one function call, parameterized.

## 2. Theory

### 2.1 Two different arrows

There are two relations in the object system and confusing them is the source of most
errors here.

- **instance-of** (`type(x) is C`, or `isinstance(x, C)`): what kind of thing is this?
- **subclass-of** (`issubclass(D, C)`): does this class specialize that one?

They are orthogonal. Draw them as different arrows. Here is the full picture for
`class Dog: pass; d = Dog()`:

```
                 instance-of              instance-of
        d  ──────────────────►  Dog  ──────────────────►  type
                                 │                          │
                                 │ subclass-of              │ subclass-of
                                 ▼                          ▼
                               object  ◄──────────────────  object
                                        (type is a subclass
                                         of object)
```

and completing the loop:

```
type(object)  is type          # object is an instance of type
type(type)    is type          # type is an instance of itself
issubclass(type, object)       # True
issubclass(object, type)       # False
object.__bases__               # ()  — object has no base
```

So `object` is at the top of the *subclass* lattice, and `type` is at the top of the
*instance-of* chain. They refer to each other, which is impossible to construct in pure
Python and is bootstrapped in C at interpreter start-up.

A **metaclass** is simply a class whose instances are classes. `type` is the default one.
Nothing more mysterious than that: `Dog` is to `type` as `d` is to `Dog`.

### 2.2 What `class` actually does

The statement

```python
class Dog(Animal, metaclass=Meta, tracked=True):
    legs = 4
    def speak(self): ...
```

is compiled into approximately this sequence:

1. **Resolve the metaclass.** Take the explicit `metaclass=` keyword if present; otherwise
   take the type of the first base; otherwise `type`. Then walk all bases and pick the
   *most derived* metaclass among them and the candidate. If two candidate metaclasses are
   unrelated by inheritance, raise `TypeError: metaclass conflict`.
2. **Prepare the namespace.** Call `Meta.__prepare__(name, bases, **kwds)`, which returns a
   mapping. `type.__prepare__` returns a plain `dict`. Returning something else lets you
   observe or reorder definitions — this is how `enum` detects duplicate members and how
   ordered-namespace tricks work.
3. **Execute the class body** in that mapping as its local namespace. Assignments,
   `def`s, and any other statements run now, in order, top to bottom. This is ordinary
   code execution — you can loop, branch, and call functions in a class body.
4. **Call the metaclass.** `Meta(name, bases, namespace, **kwds)`. Which, since `Meta` is
   itself a class, means `type(Meta).__call__(Meta, ...)`, which calls `Meta.__new__` and
   then `Meta.__init__`.
5. **Bind the result** to the name `Dog` in the enclosing namespace (an ordinary binding,
   per L01 §2.3).

You can do step 4 manually, which is the clearest way to internalize it:

```python
def speak(self):
    return "woof"

Dog = type("Dog", (Animal,), {"legs": 4, "speak": speak})
```

That is a complete, ordinary class. `class` is syntax for this.

The three-argument `type(name, bases, ns)` and the one-argument `type(obj)` are the same
callable behaving differently on arity — a wart, preserved for compatibility.

### 2.3 Class creation keywords and `__init_subclass__`

Extra keywords in the class header (`tracked=True` above) flow to the metaclass. Since
Python 3.6 you rarely need a metaclass for this: `__init_subclass__` is a hook on the
*parent* class that runs after each subclass is created, and it receives those keywords.

```python
class Plugin:
    registry: dict[str, type["Plugin"]] = {}

    def __init_subclass__(cls, /, key: str, **kw: object) -> None:
        super().__init_subclass__(**kw)
        Plugin.registry[key] = cls

class Csv(Plugin, key="csv"): ...
```

`__init_subclass__` is implicitly a classmethod. This covers perhaps 80% of the cases that
used to require a metaclass, at a fraction of the conceptual cost, and PY-502 L05 argues
that reaching for a metaclass when `__init_subclass__` would do is a design failure rather
than a display of skill.

### 2.4 Instance creation

```python
d = Dog()
```

`Dog` is an object; calling it means `type(Dog).__call__(Dog)`, i.e. `type.__call__`, whose
behaviour is:

1. `obj = cls.__new__(cls, *args, **kwargs)` — the **allocator**. A static method (special-
   cased; you do not need the decorator, but write it for clarity). Returns the new object.
2. **If** `isinstance(obj, cls)`, call `type(obj).__init__(obj, *args, **kwargs)` — the
   **initializer**. Returns `None`; returning anything else is a `TypeError`.
3. Return `obj`.

The conditional in step 2 is important and frequently exploited: a `__new__` that returns
an object of a *different* type suppresses `__init__` entirely. That is how singleton and
interning patterns are built, and it is how `int.__new__` can return a cached small int.

```python
class Meters(float):
    def __new__(cls, value: float) -> "Meters":
        return super().__new__(cls, value)     # float is immutable: value must be set here
```

**Rule of thumb:** immutable types customize `__new__` because by the time `__init__` runs
the value is already fixed; mutable types customize `__init__`. Getting this backwards is
the standard error when subclassing `tuple`, `str`, `int`, or `float`.

### 2.5 Instance layout: `__dict__` and `__slots__`

By default, instances carry a `__dict__` — a per-instance mapping for attributes. That is
what makes `d.anything = 1` work on an arbitrary object.

```python
class Point:
    __slots__ = ("x", "y")
```

`__slots__` tells the type constructor to allocate fixed storage descriptors for exactly
those names and **not** to create `__dict__`. Consequences:

- Attribute access goes through a descriptor into a fixed offset instead of a dict lookup:
  modestly faster, and substantially smaller. On CPython the saving is typically 40–60% of
  per-instance memory for small objects — PY-602 L04 measures it properly.
- `p.z = 1` raises `AttributeError`. This is a side effect, not the purpose, and relying on
  it as an integrity mechanism is fragile: any subclass without `__slots__` reintroduces
  `__dict__` and the restriction silently vanishes.
- Multiple inheritance from two slotted classes with non-empty slots raises `TypeError`
  (layout conflict). Empty `__slots__ = ()` is compatible and is the right choice for
  mixins.
- `__slots__` and class-level defaults collide: you cannot have both a slot named `x` and a
  class attribute `x`, because the slot descriptor occupies the class-dict entry.

`dataclasses.dataclass(slots=True)` builds a slotted class by constructing a *new* class,
which matters if anything captured a reference to the original — a genuinely surprising
behaviour worth knowing before you meet it in a decorator stack.

### 2.6 `isinstance`, `issubclass`, and virtual subclassing

`isinstance(x, C)` does not simply check `type(x) is C` or walk `__mro__`. It calls
`type(C).__instancecheck__(C, x)`. The default implementation on `type` does the MRO walk;
`abc.ABCMeta` overrides it to consult a registry.

```python
from collections.abc import Sized

class Bag:
    def __len__(self) -> int: return 0

isinstance(Bag(), Sized)      # True — Bag never mentions Sized
```

`Sized` implements `__subclasshook__` to answer yes for any class defining `__len__`. This
is *structural* checking bolted onto a nominal system, and it is the ancestor of
`typing.Protocol` (PY-502 L02).

Explicit virtual subclassing:

```python
import abc

class Serializer(abc.ABC):
    @abc.abstractmethod
    def dump(self, obj: object) -> bytes: ...

Serializer.register(bytes)     # bytes is now a "virtual subclass"
issubclass(bytes, Serializer)  # True
```

Note what registration does *not* do: it does not add methods, does not affect the MRO,
and does not check that `bytes` actually has `dump`. It is a claim, not a proof. Use it
sparingly and never as a substitute for actually satisfying the interface.

**Style rule:** prefer `isinstance` over `type(x) is C` — except when you specifically mean
"exactly this type and not a subclass", which is rare but real (e.g. `bool` is a subclass
of `int`, so `isinstance(True, int)` is `True` and that is occasionally the wrong answer).

### 2.7 Subclassing built-ins: a warning

```python
class MyDict(dict):
    def __setitem__(self, k, v):
        print("set", k)
        super().__setitem__(k, v)

d = MyDict()
d["a"] = 1        # prints
d.update(b=2)     # does NOT print
```

`dict.update` is implemented in C and calls the internal insertion routine directly, not
your `__setitem__`. The same holds for `dict.__init__`, `list.extend`, `set` operations,
and many others. Subclassing a C built-in gives you *some* of the extension points, and
which ones is unspecified and version-dependent.

The fix is to subclass the pure-Python wrapper written for this purpose:

```python
from collections import UserDict

class MyDict(UserDict):     # all mutation routes through __setitem__
    ...
```

or, better, to *contain* rather than inherit — hold a `dict` and expose the interface you
actually want. Inheriting from a built-in to add one behaviour buys you an enormous
surface area of inherited semantics you have not thought about. SE-521 L03 makes the
general argument; this is its most concrete instance.

## 3. Construction: a class registry, three ways

**Version 1 — a metaclass.** The traditional approach:

```python
class RegistryMeta(type):
    registry: dict[str, type] = {}

    def __new__(mcls, name, bases, ns, **kw):
        cls = super().__new__(mcls, name, bases, ns, **kw)
        if bases:                        # skip the base class itself
            RegistryMeta.registry[name.lower()] = cls
        return cls

class Handler(metaclass=RegistryMeta): ...
class JsonHandler(Handler): ...
```

Works. Now try to combine `Handler` with an `abc.ABC`: `TypeError: metaclass conflict`,
because `ABCMeta` and `RegistryMeta` are unrelated. You must write
`class RegistryMeta(abc.ABCMeta)` — and now your registry is coupled to `abc`, and the next
library that wants a metaclass will conflict with you. **Metaclasses do not compose.** That
is their defining practical weakness.

**Version 2 — `__init_subclass__`.** Same behaviour, no metaclass, composes with everything:

```python
class Handler:
    registry: dict[str, type["Handler"]] = {}

    def __init_subclass__(cls, /, name: str | None = None, **kw: object) -> None:
        super().__init_subclass__(**kw)
        Handler.registry[name or cls.__name__.lower()] = cls
```

`Handler` may now also inherit from `abc.ABC`, a `Protocol`, or anything else.

**Version 3 — an explicit decorator.** No implicit behaviour at all:

```python
registry: dict[str, type] = {}

def handles(name: str):
    def deco(cls: type) -> type:
        registry[name] = cls
        return cls
    return deco

@handles("json")
class JsonHandler: ...
```

Version 3 is the most readable and the most greppable: the registration is visible at the
call site rather than inherited invisibly. Version 2 is right when registration must be
*mandatory* — a subclass cannot forget to be registered. Version 1 is right when you must
also control the namespace during class body execution (`__prepare__`), which is rare.

Write all three. The exercise is not the code; it is being able to state the decision rule.

## 4. Failure modes

- **Metaclass conflict.** As above. If you see it, the answer is almost always to delete a
  metaclass, not to add another layer.
- **`__init__` silently skipped.** A `__new__` returning an object not an instance of `cls`
  suppresses `__init__`. Debugging sessions have been lost to this.
- **Mutable class attributes.** L01 §4.2; the class body runs once.
- **`type(x) == C` in a `Protocol`-shaped world.** Breaks duck typing and breaks subclasses.
- **Assuming `__slots__` is a security boundary.** It is a memory optimization with a
  side effect.
- **Inheriting from `dict`/`list`/`str` for behaviour.** §2.7.
- **`super().__init_subclass__(**kw)` forgotten.** Breaks cooperative chains; a subclass
  further down never gets its hook called. The same cooperative discipline as L04's
  `super()` rules.

## 5. Exercises

### Warm-up (20 min)

**W1.** Draw the instance-of and subclass-of arrows for: `True`, `bool`, `int`, `object`,
`type`, and `abc.ABCMeta`. Verify each edge in the REPL.

**W2.** Write a class whose `__new__` returns an `int`. Show that `__init__` does not run.
Then explain the exact condition in `type.__call__` that causes this.

**W3.** Construct a `TypeError: metaclass conflict` in three lines. Then fix it two
different ways.

### Core (2 hours)

**C1 — Reimplement `class`.** Write `make_class(name, bases, body_fn, metaclass=None,
**kwds)` that takes a function `body_fn(ns: dict) -> None` playing the role of the class
body, and reproduces the full protocol of §2.2: metaclass resolution (including the
"most derived" rule and the conflict error), `__prepare__`, body execution, metaclass call.
Test it against real `class` statements for at least six hierarchies, including one with
`abc.ABCMeta` and one where the metaclass must be derived from a base rather than given.

**C2 — Slots, measured.** Build a class with five attributes, in four variants: plain,
`__slots__`, `dataclass`, `dataclass(slots=True)`. Measure per-instance memory for 10⁶
instances (use `tracemalloc` or `memray`; `sys.getsizeof` alone will mislead you and part
of the exercise is explaining why). Measure attribute access time. Report a table with
medians and spread, and state which differences you would consider decision-relevant.

**C3 — The `UserDict` argument.** Take `MyDict` from §2.7. Enumerate every `dict` method
that bypasses `__setitem__` on your Python version — empirically, by instrumenting. Then
write the same class three ways (subclass `dict`, subclass `UserDict`, compose) and write
one test suite that all three must pass. Report which passed and what that tells you about
inheritance from C types.

**C4 — Virtual subclass hazards.** Register a class with `abc.ABC.register` that does *not*
implement the abstract methods. Show that `issubclass` says yes and calling fails. Then
write a `Protocol` (see `typing.runtime_checkable`) that expresses the same interface, and
compare: which errors surface at type-check time, which at runtime, and which not at all?
Write 300 words on when nominal registration is nonetheless the right tool.

### Challenge

**X1.** Implement an `AutoSlots` mechanism that inspects a class's annotations and
generates `__slots__` automatically, correctly handling inheritance (no duplicate slots),
defaults, and the class-attribute collision of §2.5. Then explain, in writing, why
`dataclass(slots=True)` returns a *new class* rather than mutating in place, and construct
a program where that distinction changes the observable result.

**X2.** Read `Objects/typeobject.c`, function `type_new`. Produce a one-page annotated
outline of what it does, in order. You will not understand all of it; identify the three
parts you did not understand and write down precisely what you would need to learn to
understand them.

## 6. Self-check

1. Why is `type(type) is type`, and why can it not be constructed in pure Python?
2. State the five steps of executing a `class` statement.
3. What determines the metaclass when none is given explicitly?
4. Give the exact condition under which `__init__` is not called after `__new__`.
5. When should a type customize `__new__` rather than `__init__`?
6. What does `abc.ABC.register` do, and what does it not do?
7. Why do metaclasses not compose, and what should you use instead?
8. Name three `dict` operations that bypass a subclass's `__setitem__`.

## 7. Primary sources

- Language Reference §3.3.3 (Customizing class creation) and §3.4 (Coroutines... skip);
  read the *Metaclasses* subsection closely.
- PEP 3115 (metaclasses in Python 3000) and PEP 487 (`__init_subclass__`,
  `__set_name__`). PEP 487's motivation section is the best short argument against
  metaclasses that exists.
- Ramalho, *Fluent Python* 2e, ch. 24.

---

**Previous:** [L01](L01-names-objects-and-binding.md) · **Next:**
[L03 — Attribute Lookup and the Descriptor Protocol](L03-attribute-lookup-and-descriptors.md)
