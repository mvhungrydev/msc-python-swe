# PY-501 · Lesson 04 — Inheritance, `super()`, and the MRO

**Estimated study time:** 4 hours
**Prerequisites:** L02, L03

---

## 1. Orientation

`super()` is the most widely misunderstood construct in Python. The common mental model —
"`super()` calls my parent class" — is wrong, and the ways in which it is wrong are exactly
the ways multiple inheritance breaks.

The correct model: **`super()` returns a proxy that dispatches to the next class in the
MRO of `type(self)`, after the class in which the calling method is defined.** Note what is
in that sentence: `type(self)`, not the class where the method lives. `super()` in class `B`
may dispatch to a class that `B` has never heard of, defined by someone else, years later.

That is not a defect. It is the entire design — cooperative multiple inheritance — and
once you see it, mixins and `__init_subclass__` chains stop being mysterious.

## 2. Theory

### 2.1 The MRO

The **method resolution order** of a class is a linear sequence of classes, searched left
to right for attribute lookup (L03 §2.1 step 1). It is computed once at class creation and
stored on `C.__mro__`.

For single inheritance it is the obvious chain. For multiple inheritance, Python uses
**C3 linearization**, adopted in Python 2.3 (from the Dylan language). C3 guarantees three
properties:

1. **Consistency with the local precedence order.** If `class C(A, B)`, then `A` precedes
   `B` in `C.__mro__`.
2. **Monotonicity.** If `A` precedes `B` in the MRO of some class, then `A` precedes `B` in
   the MRO of every subclass of that class. Orders never get reshuffled as you go deeper.
3. **The extended precedence graph is respected** — a class always precedes its own bases.

A hierarchy for which no order satisfies all three has **no linearization**, and Python
refuses to create the class:

```python
class A: pass
class B: pass
class X(A, B): pass
class Y(B, A): pass
class Z(X, Y): pass
# TypeError: Cannot create a consistent method resolution order (MRO) for bases X, Y
```

`X` requires A before B; `Y` requires B before A; `Z` needs both. This is a *design* error
being caught at class-creation time, which is exactly where you want it.

### 2.2 Computing C3 by hand

The algorithm. Write `L[C]` for the linearization of `C`.

```
L[object] = [object]
L[C(B1, ..., Bn)] = C + merge(L[B1], ..., L[Bn], [B1, ..., Bn])
```

`merge` repeatedly takes the **head** of the first remaining list that does **not** appear
in the *tail* (anything but the first position) of any other list, appends it to the
output, and removes it from all lists. If no such head exists, the merge fails and so does
the class creation.

Worked example — the diamond:

```
      object
      /    \
     B      C
      \    /
        D          class B(object), C(object), D(B, C)
```

```
L[B] = [B, object]
L[C] = [C, object]
L[D] = D + merge([B, object], [C, object], [B, C])

  head B: appears in tail of others? [C,object] no, [B,C] — B is head there, not tail. Take B.
  remaining: merge([object], [C, object], [C])
  head object: appears in tail of [C, object]. Reject.
  next list head C: in tails? no. Take C.
  remaining: merge([object], [object], [])
  head object: take it.

L[D] = [D, B, C, object]
```

Verify with `D.__mro__`. Do the same by hand for the classic five-class "grid" and for a
hierarchy with a mixin, before you look at the answer. This is the one piece of the course
where hand-computation genuinely builds intuition, because it makes visible that the MRO is
a *global* property of the hierarchy, not a local property of any class.

### 2.3 What `super()` actually is

`super()` with no arguments is compiled sugar for `super(__class__, self)`, where
`__class__` is an implicit closure cell inserted by the compiler into any method that
mentions `super` or `__class__`. (You can see the cell: a method using `super()` has
`__class__` in its `__code__.co_freevars`.)

`super(C, obj)` returns a proxy object. Attribute lookup on that proxy:

1. Take `mro = type(obj).__mro__`.
2. Find the index of `C` in it.
3. Search `mro[index+1:]` for the attribute.
4. If found and it is a descriptor, invoke `__get__(obj, type(obj))`.

Three consequences that dissolve most confusion:

- **The MRO is that of `type(obj)`, not of `C`.** A method in `C` calling `super()` may
  land in a class that is not a base of `C` at all.
- **`super()` is not "the parent".** In a diamond, `B.method` calling `super().method()`
  from a `D` instance dispatches to `C`, which is `B`'s *sibling*.
- **`super()` outside a method, or with a mismatched instance, is an error.** `super(C, obj)`
  requires `isinstance(obj, C)`.

The canonical demonstration:

```python
class A:
    def go(self): print("A")

class B(A):
    def go(self): print("B"); super().go()

class C(A):
    def go(self): print("C"); super().go()

class D(B, C):
    def go(self): print("D"); super().go()

D().go()        # D B C A
```

`B.go`'s `super()` reached `C`. `B` does not inherit from `C`. It works because the MRO of
`D` is `[D, B, C, A, object]` and `super()` walks *that*.

### 2.4 Cooperative multiple inheritance: the contract

The above only works if every participant obeys a contract:

1. **Every class in the hierarchy calls `super()`** for the cooperative method, including
   the ones that "don't need to". A class that omits the call truncates the chain, and the
   classes after it silently do nothing.
2. **The root of the chain must not call `super()`** — or must call it into something that
   absorbs it. Usually the chain ends at `object`, whose `__init__` accepts no arguments.
3. **Signatures must be compatible.** Since a method may be followed by an arbitrary class,
   the safest convention is that cooperative methods accept and forward `**kwargs`, each
   consuming what it understands:

```python
class Base:
    def __init__(self, **kw): super().__init__(**kw)     # ends at object.__init__()

class Timestamped(Base):
    def __init__(self, *, created_at=None, **kw):
        super().__init__(**kw)
        self.created_at = created_at

class Named(Base):
    def __init__(self, *, name, **kw):
        super().__init__(**kw)
        self.name = name

class Doc(Timestamped, Named):
    pass

Doc(name="x", created_at=1)
```

Every class strips its own keywords and forwards the rest; `object.__init__()` receives an
empty dict and is happy. If any class forgets `**kw`, the chain breaks with a confusing
`TypeError` naming a class that is not the culprit.

4. **`__init_subclass__` and `__set_name__` chains obey the same rule.** Always
   `super().__init_subclass__(**kw)`.

This contract is fragile. It is also, in practice, how mixins work everywhere in the
Python ecosystem — Django's class-based views, `unittest` mixins, ABC hierarchies. Learn to
write it correctly; then learn to prefer composition (SE-521 L03) so that you rarely have
to.

### 2.5 Mixins done properly

A well-behaved mixin:

- Inherits from `object` (or a documented cooperative base), never from the concrete class
  it augments.
- Is listed **before** the concrete base: `class MyView(LoginRequiredMixin, DetailView)`.
  This is not style; the MRO puts earlier bases first, so a mixin listed last cannot
  intercept anything.
- Declares `__slots__ = ()` so it does not force a `__dict__` onto slotted classes.
- Does not define `__init__` unless it participates cooperatively with `**kwargs`.
- Names itself `...Mixin` so that reviewers can see the intent.

The ordering rule catches people constantly. `class V(DetailView, LoginRequiredMixin)`
type-checks, imports, and does nothing, because `DetailView.get` is found first and never
calls into the mixin.

### 2.6 Abstract base classes

```python
import abc

class Repository(abc.ABC):
    @abc.abstractmethod
    def get(self, key: str) -> bytes: ...

    @abc.abstractmethod
    def put(self, key: str, value: bytes) -> None: ...

    def get_or_default(self, key: str, default: bytes) -> bytes:
        try:
            return self.get(key)
        except KeyError:
            return default
```

Mechanism: `abc.abstractmethod` sets `__isabstractmethod__ = True` on the function;
`ABCMeta.__new__` collects those names into `cls.__abstractmethods__`; and
`object.__new__` refuses to instantiate a class with a non-empty `__abstractmethods__`.

Two things this buys you: a *template method* (`get_or_default`) shared by all
implementations, and an instantiation-time error rather than a call-time `AttributeError`.

Two things it costs: nominal coupling (implementers must import and inherit from your ABC)
and a metaclass (`ABCMeta`), with the composition problems of L02 §3.

`typing.Protocol` is the structural alternative — no inheritance required, checked
statically. PY-502 L02 treats the choice properly. The short version: **use a Protocol to
describe what you consume; use an ABC when you want to share implementation.**

### 2.7 Composition and delegation

The alternative to inheritance is holding a reference:

```python
class CachedRepository:
    def __init__(self, inner: Repository, cache: MutableMapping[str, bytes]) -> None:
        self._inner, self._cache = inner, cache

    def get(self, key: str) -> bytes:
        try:
            return self._cache[key]
        except KeyError:
            value = self._cache[key] = self._inner.get(key)
            return value

    def put(self, key: str, value: bytes) -> None:
        self._inner.put(key, value)
        self._cache.pop(key, None)
```

Longer. Also: explicit about what is forwarded, testable in isolation, composable with
other decorators in any order, and it cannot be broken by a change to the base class's
internal call structure. The classic failure of inheritance-for-reuse — a base class method
being rewritten to stop calling an overridable hook, breaking every subclass — cannot
happen here.

The decision rule that survives contact with real code:

> Inherit to be *substitutable*. Compose to *reuse*.

If you would not pass your subclass to a function expecting the base (Liskov), you are not
specializing; you are reusing, and you should compose.

## 3. Construction: a diagnosable MRO

Build a tool that explains an MRO rather than just printing it.

**Version 1** — print it:

```python
def mro(cls: type) -> list[str]:
    return [c.__name__ for c in cls.__mro__]
```

Useless when something goes wrong, because the interesting question is *why* a particular
class won, and where a `super()` chain stopped.

**Version 2** — resolve one attribute and show the chain:

```python
def resolve(cls: type, name: str) -> list[tuple[str, str]]:
    """Every class in the MRO that defines `name`, in order."""
    out = []
    for c in cls.__mro__:
        if name in c.__dict__:
            out.append((c.__name__, type(c.__dict__[name]).__name__))
    return out
```

`resolve(D, "go")` → `[('D','function'), ('B','function'), ('C','function'), ('A','function')]`.
Now you can see the whole cooperative chain, and you can see when a class is missing from
it.

**Version 3** — detect broken cooperation. For a given method name, statically check
whether every class that defines it also calls `super()`:

```python
import inspect, ast

def calls_super(cls: type, name: str) -> bool:
    fn = cls.__dict__.get(name)
    if fn is None or not callable(fn):
        return False
    try:
        tree = ast.parse(inspect.getsource(fn).lstrip())
    except (OSError, SyntaxError):
        return False
    return any(
        isinstance(n, ast.Call) and isinstance(n.func, ast.Attribute)
        and isinstance(n.func.value, ast.Call)
        and getattr(n.func.value.func, "id", None) == "super"
        for n in ast.walk(tree)
    )
```

Crude — it misses `super(C, self)`, aliased supers, and dynamic dispatch — but it catches
the common truncated-chain bug. Making it robust is exercise C3, and the honest conclusion
you should reach is that *this cannot be done reliably by static analysis in Python*, which
is itself an argument about cooperative inheritance as a design.

## 4. Failure modes

- **"`super()` calls my parent."** It calls the next class in `type(self)`'s MRO.
- **Truncated chain.** One class in the middle forgets `super()`; classes after it never
  run. Symptom: an attribute is mysteriously unset, and the class that fails to set it is
  not the one you edited.
- **Mixin listed last.** §2.5. Silent no-op.
- **Incompatible `__init__` signatures.** Fix with `**kwargs` forwarding, or avoid multiple
  inheritance for constructors entirely.
- **Calling `super().__init__()` with positional args in a diamond.** Position-based
  forwarding cannot work when the next class is unknown. Keyword-only, always.
- **Deep hierarchies.** Beyond three levels, an override's effect becomes unpredictable by
  reading. Ousterhout's *A Philosophy of Software Design* calls this out; treat depth > 3
  as a defect requiring justification.
- **Inheriting to reuse.** §2.7.
- **Assuming `__mro__` is stable.** It is fixed at creation, but `cls.__bases__` can be
  assigned in exotic cases and the MRO recomputed. Do not.

## 5. Exercises

### Warm-up (30 min)

**W1.** Compute by hand the MRO of `Z` in this hierarchy, then verify:

```python
class O: pass
class A(O): pass
class B(O): pass
class C(O): pass
class D(A, B): pass
class E(B, C): pass
class Z(D, E): pass
```

**W2.** Construct two more hierarchies with no valid linearization, structurally different
from §2.1's.

**W3.** Write a class where `super().m()` reaches a class that is not among the defining
class's own bases. Explain in one sentence why it works.

### Core (2.5 h)

**C1 — Implement C3.** Write `c3(cls_graph)` operating on a plain description of a
hierarchy (names and base lists), producing the linearization or raising a descriptive
error naming the conflicting constraint. Validate against `type.__mro__` for at least 30
randomly generated hierarchies. Then use `hypothesis` (after SE-511 L04) to generate
hierarchies and assert your result matches CPython's — including that both fail on the same
inputs.

**C2 — Cooperative `__init__` under stress.** Build a five-class diamond where each class
needs its own constructor arguments. Make it work. Then deliberately break it four
different ways (missing `super()`, positional forwarding, wrong mixin order, a class that
does not accept `**kwargs`) and write down, for each, the exact error message and how you
would diagnose it from that message alone. This is the exercise that pays off in
production.

**C3 — Cooperation checker.** Improve `calls_super` to handle `super(C, self)`, methods
defined via decorators, and methods inherited from a class whose source is unavailable
(C extensions). Document precisely what your checker cannot detect, and give a program that
defeats it. Conclude with 300 words: is cooperative multiple inheritance statically
checkable in Python? Defend your answer.

**C4 — ABC vs Protocol.** Take a real interface from your work (a repository, a client, a
serializer). Express it three ways: an ABC, a `typing.Protocol`, and a plain duck-typed
convention with no declaration. Write the same test suite against all three. Compare:
what errors are caught, when, with what message, and what each costs the implementer. 500
words.

### Challenge

**X1.** Read Michele Simionato's "The Python 2.3 Method Resolution Order" essay and
implement the *pathological* examples it discusses. Then find a hierarchy where C3 gives a
correct-but-surprising order and write up why the surprise is a property of the hierarchy
rather than the algorithm.

**X2.** Build a linter rule (a `ruff`-style AST check or an `ast.NodeVisitor`) that flags:
mixins listed after concrete bases, cooperative methods missing `super()`, and inheritance
depth > 3. Run it against a real open-source codebase and report the false-positive rate.
A high false-positive rate is a finding, not a failure — explain what causes it.

## 6. Self-check

1. State precisely what `super()` dispatches to.
2. Give the three properties C3 guarantees.
3. Run the merge algorithm aloud on `L[D] = D + merge(L[B], L[C], [B, C])`.
4. Why must cooperative methods forward `**kwargs` rather than positional arguments?
5. Why must a mixin be listed before the concrete base?
6. What does `abstractmethod` actually do, and what enforces it?
7. State the inherit-vs-compose decision rule and justify it with Liskov.
8. Give a concrete way a base class change silently breaks subclasses that composition
   would have prevented.

## 7. Primary sources

- Simionato, "The Python 2.3 Method Resolution Order" (python.org). The definitive account.
- Barrett et al., "A Monotonic Superclass Linearization for Dylan" (OOPSLA 1996) — the
  original C3 paper.
- Liskov & Wing, "A Behavioral Notion of Subtyping" (TOPLAS 1994). Read §1–3.
- Hettinger, "Python's `super()` Considered Super!" (2011 essay and PyCon talk).
- Ousterhout, *A Philosophy of Software Design*, ch. on "Different Layer, Different
  Abstraction" and the inheritance critique.

---

**Previous:** [L03](L03-attribute-lookup-and-descriptors.md) · **Next:**
[L05 — Identity, Equality, Hashing, and Ordering](L05-identity-equality-hashing-ordering.md)
