# PY-502 · Lesson 03 — Class Construction Hooks and Metaclasses

**Estimated study time:** 4 hours
**Prerequisites:** PY-501 L02, L04; PY-502 L02

---

## 1. Orientation

Tim Peters, on comp.lang.python in 2002:

> "Metaclasses are deeper magic than 99% of users should ever worry about. If you wonder
> whether you need them, you don't."

He was right, and PEP 487 (Python 3.6) made him more right by moving the two things people
actually wanted — a hook when a subclass is created, and a hook when a descriptor is named —
out of metaclasses and into ordinary class machinery.

This lesson teaches metaclasses properly, so that you can read the frameworks that use them,
and then teaches you the three or four cases where they remain necessary. The rest of the
time the answer is `__init_subclass__` or a class decorator.

## 2. Theory

### 2.1 The four hooks, in the order they fire

For `class D(B, metaclass=M, **kwds):`

1. **`M.__prepare__(name, bases, **kwds)`** → the mapping the class body executes in.
2. **The class body executes** in that mapping.
3. **`M.__new__(M, name, bases, ns, **kwds)`** → creates the class object. Inside
   `type.__new__`, two more things happen:
   - **`__set_name__(owner, name)`** is called on every namespace value that defines it.
   - **`__init_subclass__(cls, **kwds)`** is called on the *nearest base class* that defines
     it (implicitly a classmethod), receiving the new class.
4. **`M.__init__(cls, name, bases, ns, **kwds)`** → initializes it.
5. **Class decorators** apply, bottom-up, after the class object exists.
6. **The name is bound** in the enclosing namespace.

Knowing this order settles most "why didn't my hook see X?" questions. In particular:
`__init_subclass__` runs *before* class decorators, so a decorator-added attribute is
invisible to it; and `__set_name__` runs before `__init_subclass__`, so descriptors already
know their names by the time the subclass hook collects them. That ordering is deliberate
and it is what makes the field-system pattern of L02 §3 work.

### 2.2 `__init_subclass__`

```python
class Plugin:
    registry: dict[str, type["Plugin"]] = {}

    def __init_subclass__(cls, /, key: str | None = None, abstract: bool = False,
                          **kw: object) -> None:
        super().__init_subclass__(**kw)          # mandatory: cooperative chain
        if abstract:
            return
        k = key or cls.__name__.lower()
        if k in Plugin.registry:
            raise TypeError(f"duplicate plugin key {k!r}")
        Plugin.registry[k] = cls

class Csv(Plugin, key="csv"): ...
```

Properties:

- Implicitly a `classmethod`; `cls` is the **new subclass**, not the defining class.
- Does **not** run for the class that defines it — only for its subclasses.
- Class keywords flow in as keyword arguments.
- **Must call `super().__init_subclass__(**kw)`** or you truncate the cooperative chain
  (PY-501 L04 §2.4) and any other base's hook silently stops running.
- Composes with anything, including ABCs, Protocols, and other libraries' base classes.

This replaces the overwhelming majority of historical metaclass uses: registration,
validation of subclass definitions, computing derived class attributes, enforcing
implementation requirements, and collecting declarative field definitions.

### 2.3 Class decorators

```python
def frozen(cls: type) -> type:
    def __setattr__(self, name, value):
        raise AttributeError(f"{type(self).__name__} is immutable")
    cls.__setattr__ = __setattr__
    return cls

@frozen
class Config: ...
```

Properties:

- Run **after** the class is fully created, so they see everything, including what
  `__init_subclass__` did.
- **Not inherited.** A subclass of `Config` is not frozen unless separately decorated. This
  is sometimes the point and sometimes the bug.
- Explicit and visible at the definition site — greppable, no action at a distance.
- May return a *different* class (this is how `dataclass(slots=True)` works), which is
  surprising and must be documented.
- Compose by stacking, bottom-up. Order matters and is a frequent source of confusion;
  document the required order or make them order-independent.

`dataclasses.dataclass`, `functools.total_ordering`, `typing.runtime_checkable`, and
`attrs.define` are all class decorators. It is the mainstream mechanism.

### 2.4 Metaclasses: what only they can do

Four capabilities that `__init_subclass__` and class decorators cannot provide:

**1. Controlling the class body's namespace (`__prepare__`).** You need to observe
definitions *as they happen*, in order, or intercept name lookups inside the class body.

```python
class NoDuplicates(dict):
    def __setitem__(self, k, v):
        if k in self:
            raise TypeError(f"{k!r} defined twice in class body")
        super().__setitem__(k, v)

class StrictMeta(type):
    @classmethod
    def __prepare__(mcls, name, bases, **kw):
        return NoDuplicates()
```

`enum` uses exactly this to detect duplicate members and to allow `auto()` to know its
position. There is no other way to get this information; by the time
`__init_subclass__` runs, the namespace is a plain dict and the duplicate has already
overwritten the original.

**2. Customizing behaviour *of the class object itself*.** Special methods are looked up on
the type (PY-501 L03 §2.6), so to make `len(MyClass)`, `MyClass[x]`, `iter(MyClass)`, or
`MyClass1 | MyClass2` work, the method must be on the *metaclass*.

```python
class QueryableMeta(type):
    def __getitem__(cls, key):        # enables MyModel["field"]
        return cls._fields[key]
    def __repr__(cls):                # a nicer repr for the class, not instances
        return f"<Model {cls.__name__}>"
```

This is how `typing`'s subscription (`list[int]`) historically worked, how `Enum` supports
`Color["RED"]` and `len(Color)`, and how ORMs make `Model.field == 3` build a query.

**3. Intercepting instantiation.** `type.__call__` is what runs `__new__` then `__init__`
(PY-501 L02 §2.4). Overriding `__call__` on the metaclass lets you intervene around the
whole construction, e.g. for interning, pooling, or returning a proxy:

```python
class Singleton(type):
    _instances: dict[type, object] = {}
    def __call__(cls, *a, **kw):
        if cls not in cls._instances:
            cls._instances[cls] = super().__call__(*a, **kw)
        return cls._instances[cls]
```

(Do not use the Singleton pattern. It is a global with extra steps and it makes testing
miserable — SE-521 L04. It is here because it is the clearest illustration of the
mechanism.)

**4. Affecting `isinstance`/`issubclass`.** `__instancecheck__` and `__subclasscheck__` live
on the metaclass. This is `ABCMeta`'s registry and `Protocol`'s runtime checking.

If your requirement is not one of these four, you do not need a metaclass.

### 2.5 Why metaclasses do not compose

A class's metaclass must be a (non-strict) subclass of the metaclasses of all its bases.
Two libraries that each define an independent metaclass cannot be combined:

```python
class A(metaclass=MetaA): ...
class B(metaclass=MetaB): ...
class C(A, B): ...      # TypeError: metaclass conflict
```

The only fix is a metaclass inheriting from both — which the *user* must write, and which
requires both metaclasses to have been designed to cooperate (they were not). This is the
defining practical problem: a metaclass is a claim on a scarce, non-composable slot in every
class that uses it.

It also explains why `ABCMeta` shows up in so many hierarchies as a base for other
metaclasses: everyone who wants a metaclass *and* abstract methods must derive from it.

**Corollary for library authors:** if you export a base class with a metaclass, you have
constrained every one of your users' hierarchies. Do not do it unless §2.4 forces you.

### 2.6 Reading framework metaclasses

You will read more metaclasses than you write. The ones worth studying, and what each
teaches:

- **`enum.EnumMeta`/`EnumType`** — `__prepare__` for duplicate detection and member order;
  `__getitem__`, `__iter__`, `__len__`, `__contains__` on the metaclass for class-level
  behaviour; `__call__` for the `Color(1)` lookup form. A near-complete tour.
- **`abc.ABCMeta`** — `__instancecheck__`/`__subclasscheck__` with a cache, the registry,
  and `__abstractmethods__` collection.
- **`typing`'s `Protocol`** — how structural checking is bolted onto the nominal system.
- **Django's `ModelBase`** — the canonical declarative-ORM metaclass: collects fields from
  the namespace, builds `_meta`, registers with the app registry, and adds a manager.
  Written before `__init_subclass__` existed; a good exercise is to work out how much of it
  could be rewritten without a metaclass today. (Answer: most, but not the `Model.objects`
  descriptor interplay or the class-level query syntax.)
- **SQLAlchemy's declarative base** — similar, and has since moved partly to
  `__init_subclass__` and `@dataclass_transform`, which is a well-documented case study in
  exactly this lesson's argument.

### 2.7 Type checkers and generated members

Anything a metaclass or decorator *generates* is invisible to a static type checker unless
you tell it. Three mechanisms:

- **`@dataclass_transform`** (PEP 681) — declares that a decorator, base class, or metaclass
  synthesizes `__init__` from field declarations. The checkers special-case it. This is the
  supported path for declarative frameworks and you should use it.
- **Stub files** (`.pyi`) — hand-write the generated surface.
- **A checker plugin** — mypy plugins exist for Django, SQLAlchemy, and `attrs`. Expensive
  to write and maintain; the reason `@dataclass_transform` was standardized.

If your metaclass generates members that no mechanism can express, your users lose
completion and checking on those members. **That is a real cost and it belongs in the
decision.** It is, in practice, the strongest argument against clever class generation in a
typed codebase.

## 3. Construction: one requirement, four implementations

Requirement: *every subclass of `Handler` must declare a `content_type`, must implement
`handle`, must be registered by content type, and duplicates must be an error at import
time.*

**1. Metaclass.**

```python
class HandlerMeta(type):
    registry: dict[str, type] = {}
    def __new__(mcls, name, bases, ns, **kw):
        cls = super().__new__(mcls, name, bases, ns, **kw)
        if bases:
            ct = ns.get("content_type")
            if not ct: raise TypeError(f"{name} must declare content_type")
            if "handle" not in ns: raise TypeError(f"{name} must implement handle")
            if ct in mcls.registry: raise TypeError(f"duplicate {ct}")
            mcls.registry[ct] = cls
        return cls
```

Works. Cannot be combined with `abc.ABC` without deriving from `ABCMeta`. Constrains every
user hierarchy.

**2. `__init_subclass__`.**

```python
class Handler:
    registry: dict[str, type["Handler"]] = {}
    content_type: ClassVar[str]

    def __init_subclass__(cls, **kw):
        super().__init_subclass__(**kw)
        ct = getattr(cls, "content_type", None)
        if not ct: raise TypeError(f"{cls.__name__} must declare content_type")
        if cls.handle is Handler.handle:
            raise TypeError(f"{cls.__name__} must implement handle")
        if ct in Handler.registry: raise TypeError(f"duplicate {ct}")
        Handler.registry[ct] = cls
```

Composes with everything. Note the subtle difference: `getattr(cls, ...)` sees **inherited**
`content_type`, where the metaclass version checked `ns` (the class's own body). Which is
correct depends on whether an intermediate abstract subclass should be allowed to omit it.
*This is the kind of question that only surfaces when you implement it twice*, and it is the
point of the exercise.

**3. ABC + explicit registration.**

```python
class Handler(abc.ABC):
    @property
    @abc.abstractmethod
    def content_type(self) -> str: ...
    @abc.abstractmethod
    def handle(self, body: bytes) -> Response: ...

registry: dict[str, type[Handler]] = {}
def register(ct: str):
    def deco(cls: type[Handler]) -> type[Handler]:
        if ct in registry: raise TypeError(f"duplicate {ct}")
        registry[ct] = cls
        return cls
    return deco

@register("application/json")
class JsonHandler(Handler): ...
```

The abstract-method requirement fails at *instantiation*, not import — later, but with a
clearer message. Registration is explicit at the definition site.

**4. Explicit configuration, no magic at all.**

```python
HANDLERS: dict[str, type[Handler]] = {
    "application/json": JsonHandler,
    "text/csv": CsvHandler,
}
```

One place to look. Trivially greppable. Impossible to get an import-order bug. Fails at
startup if a name is wrong. The cost: someone can add a handler and forget to register it.

**The comparison is the deliverable.** For each: when does the error appear? Can it be
combined with other libraries? What does a type checker know? Can a reader find all
handlers with grep? What happens if a module is never imported (a real and nasty failure
mode for 1–3: unregistered handlers because nobody imported the module)?

That last point deserves emphasis: **implicit registration depends on import side effects**,
which means it depends on import order and on modules being imported at all. Every
plugin-registry bug you will ever debug is this bug. Option 4 does not have it.

## 4. Failure modes

- **A metaclass where `__init_subclass__` would do.** The default error.
- **Forgetting `super().__init_subclass__(**kw)`.** Truncates the chain; another library's
  hook silently stops.
- **Metaclass conflict** in user code caused by your library.
- **Import-side-effect registration.** §3. Add an explicit `load_plugins()` or use entry
  points.
- **Expecting `__init_subclass__` to see decorator-added attributes.** Ordering, §2.1.
- **Generated members invisible to type checkers.** §2.7.
- **`__prepare__` returning something whose `__setitem__` has side effects you did not
  intend.** The class body executes in it; every assignment, `def`, and import lands there.
- **A metaclass that also defines instance behaviour.** Confusing `self` (a class) with
  `self` (an instance) is the classic metaclass bug; the metaclass's methods take the
  *class* as their first argument.
- **Singleton via metaclass.** Global state, untestable, and now inheritable.

## 5. Exercises

### Warm-up (25 min)

**W1.** Instrument all six steps of §2.1 with prints and verify the order empirically.

**W2.** Produce a metaclass conflict, then resolve it three ways: derive a combined
metaclass, remove one metaclass, and restructure to avoid multiple inheritance.

**W3.** Write a metaclass that makes `len(MyClass)` and `MyClass["x"]` work. Explain why the
methods cannot go on the class itself.

### Core (2.5 h)

**C1 — Four implementations.** Complete §3 with all four, and produce the comparison table.
Deliverable: code, table, and a 600-word note recommending one for a library and one for an
application, with different answers if your reasoning supports it.

**C2 — Read `EnumType`.** Read `enum.py` in the standard library. Produce an annotated
account of: what `__prepare__` returns and why; how `auto()` knows its value; how aliases
are detected; how `Color(1)` and `Color["RED"]` differ in mechanism; and what
`_missing_` is for. Then identify one thing in it that could be done without a metaclass
today and one that could not.

**C3 — Declarative without magic.** Take a declarative API you use (a Django model, a
pydantic model, a `click` command group). Reimplement a small version using
`__init_subclass__` + descriptors + `@dataclass_transform`, no metaclass. Report what you
could not reproduce.

**C4 — The import-order bug.** Construct a plugin system with implicit registration and
demonstrate a real failure caused by a module not being imported. Then fix it three ways:
explicit imports in `__init__.py`, a `pkgutil` walk, and `importlib.metadata` entry points.
Compare on: startup cost, error clarity, and whether third parties can add plugins.

### Challenge

**X1.** Implement a metaclass-free version of `abc.ABC`: abstract-method collection and
instantiation prevention using `__init_subclass__` and `__new__` only. Then explain what you
lose (`isinstance` hooks, `register`) and whether it matters. Measure the class-creation and
instantiation cost of both.

**X2.** Write a tool that, given a package, reports every metaclass in use, what it does
(registration / namespace control / class-level protocol / instance interception), and
whether it could be replaced by `__init_subclass__` or a class decorator. Run it on three
large open-source projects and report the distribution.

## 6. Self-check

1. Give the six steps of class creation in order, including where `__set_name__` and
   `__init_subclass__` fire.
2. Name the four things only a metaclass can do.
3. Why do metaclasses not compose, and what does that cost a library's users?
4. What is the difference between `__init_subclass__` and a class decorator with respect to
   inheritance and ordering?
5. Why must `len(MyClass)` be defined on the metaclass?
6. Give the failure mode of implicit registration and three fixes.
7. What does `@dataclass_transform` solve, and what is the alternative?
8. Quote Tim Peters's rule and say when it is wrong.

## 7. Primary sources

- PEP 487 — read the *Motivation* in full; it is the argument of this lesson, made by the
  people who fixed it.
- PEP 3115 (`__prepare__`), PEP 681 (`@dataclass_transform`).
- CPython `enum.py` and `abc.py` — both readable, both instructive.
- Ramalho, *Fluent Python* 2e, ch. 24.

---

**Previous:** [L02](L02-descriptors-as-a-design-tool.md) · **Next:**
[L04 — Decorators: Design, Composition, and Typing](L04-decorators.md)
