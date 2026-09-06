# PY-501 · Lesson 01 — Names, Objects, and Binding

**Estimated study time:** 3 hours
**Prerequisites:** none
**You will need:** a REPL, and `python -X int_max_str_digits=0` is not required but a
scratch file is

---

## 1. Orientation

Here is a piece of code that has confused every Python programmer at least once:

```python
t = ([1, 2], 3)
t[0] += [4]
```

This raises `TypeError: 'tuple' object does not support item assignment` — **and it also
appends 4 to the list**. Both. The exception fires *after* the mutation succeeds.

If you cannot explain that in one paragraph, you do not yet have a precise model of what
assignment means in Python, and everything downstream in this course depends on having one.
This lesson builds that model. By the end you will be able to derive the behaviour above
from first principles rather than remember it as trivia.

The deeper point: most languages let you get away with an informal model of assignment
("a variable holds a value"). Python does not, because Python's variables do not hold
values. They are entries in a mapping, pointing at objects that exist independently of
them. Almost every "surprising" behaviour in the language falls out of that one difference.

## 2. Theory

### 2.1 Objects

The Language Reference is unusually direct here:

> Objects are Python's abstraction for data. All data in a Python program is represented by
> objects or by relations between objects.

Every object has exactly three things:

- **Identity** — never changes for the lifetime of the object. `id(x)` returns it. In
  CPython it is the object's address in memory, but that is an implementation detail; the
  only guaranteed property is that two simultaneously-live objects have distinct ids.
- **A type** — `type(x)`. Determines what operations the object supports and what values
  it can hold. An object's type is fixed for practical purposes; you can assign to
  `x.__class__` in restricted cases, but do not.
- **A value** — the data. Whether it can change partitions all objects into **mutable**
  and **immutable**.

Note what is *not* in this list: a name. Objects do not know their names. An object may
have zero names, one, or a thousand. `[1, 2, 3]` evaluated at the REPL is an object with
no name at all; it exists, briefly, and is destroyed.

### 2.2 Names and namespaces

A **name** is a binding in a **namespace**. A namespace is a mapping from names to object
references. Module globals are a real `dict` (`module.__dict__`); class bodies build a
mapping during execution; function locals are, in CPython, an array of slots indexed at
compile time (which is why they are fast and why `locals()` in a function returns a
snapshot rather than a live view).

The critical claim:

> **A name is not a box that contains an object. A name is a label attached to an object.**

Assignment attaches a label. It never copies, never constructs, never calls a method on
the object being bound.

```python
a = [1, 2, 3]
b = a
b.append(4)
print(a)          # [1, 2, 3, 4]
print(a is b)     # True
```

There is one list. There are two labels. `b.append(4)` reaches through the label and
mutates the object, and both labels see it because there is nothing else to see.

Contrast:

```python
a = [1, 2, 3]
b = a
b = b + [4]
print(a)          # [1, 2, 3]
print(a is b)     # False
```

`b + [4]` constructs a *new* list and `b =` rebinds the label `b` to it. The object `a`
names is untouched. Nothing about `b`'s previous binding constrains this: rebinding is
always allowed, regardless of the type of the object previously bound.

Ramalho's formulation is worth memorizing: *variables are labels, not boxes.* If you have
been carrying the box model since a C or Java background, deliberately replace it now — the
box model will mispredict Python's behaviour in at least a dozen places you will meet in
this course.

### 2.3 The binding operations

Assignment with `=` is the obvious one, but Python has many binding constructs, and they
all do the same thing: attach a name in some namespace to an object. Knowing the full list
matters, because "where did this name come from?" is a question you will ask often.

| Construct | Binds |
|---|---|
| `x = expr` | `x` |
| `x: T = expr` | `x` (the annotation is stored separately, see §2.7) |
| `def f(...)` | `f` |
| `class C(...)` | `C` |
| `lambda` parameters, function parameters | each parameter name, in the new frame |
| `for x in ...` | `x`, once per iteration |
| `with expr as x` | `x` |
| `except E as e` | `e` — **and unbinds it at the end of the block** (§4.3) |
| `import m`, `from m import n`, `import m as p` | `m`, `n`, `p` |
| `global x`, `nonlocal x` | change *which namespace* subsequent bindings of `x` target |
| `x := expr` (walrus) | `x`, in the enclosing scope |
| `match ...: case Point(x=a)` | `a` — capture patterns bind |
| `del x` | *un*binds `x` |

`del x` is worth dwelling on. It does not delete the object. It removes the name from the
namespace. If that was the last reference, the object becomes unreachable and CPython
reclaims it immediately (see L09); if not, nothing observable happens to the object at all.

### 2.4 Assignment targets: the two kinds

Every assignment statement has a *target*, and targets come in two families that behave
completely differently:

**Simple targets** — a bare name. `x = v` binds `x` in the current namespace. No method on
any object is called. The compiler decides at compile time which namespace `x` lives in
(L07 covers the rules).

**Compound targets** — `obj.attr = v`, `obj[k] = v`, `obj[i:j] = v`. These are not
bindings at all. They are **method calls on `obj`**:

- `obj.attr = v` calls `type(obj).__setattr__(obj, 'attr', v)`
- `obj[k] = v` calls `type(obj).__setitem__(obj, k, v)`
- `del obj[k]` calls `type(obj).__delitem__(obj, k)`

This distinction is the single most useful thing in this lesson. It explains why `x = v`
can never fail on account of `x`'s previous value, while `obj.attr = v` can raise anything
at all, and it is the door through which descriptors, properties, and `__slots__` enter in
L03.

Tuple and list targets unpack: `a, b = b, a` evaluates the right-hand side into a tuple
first, then binds left to right. Starred targets (`a, *rest = xs`) bind `rest` to a *new
list*, always — even if `xs` was a tuple.

### 2.5 Augmented assignment: the resolution of the opening puzzle

`x += y` is *not* sugar for `x = x + y`. Its actual semantics:

1. Evaluate the target enough to obtain the current object. For `t[0] += v`, this means
   evaluating `t` and then `t[0]`, i.e. calling `__getitem__`.
2. Attempt `type(obj).__iadd__(obj, y)`. If the type defines `__iadd__`, it is called; it
   is expected to mutate in place and return `self` (though it may return anything).
3. If there is no `__iadd__`, fall back to `obj = obj + y` — i.e. `__add__`/`__radd__`.
4. **Store the result back into the target**, using the target's normal store operation.

Step 4 is the one everyone forgets. Augmented assignment *always* performs a store, even
when `__iadd__` mutated in place and returned the same object.

Now the opening puzzle resolves cleanly:

```python
t = ([1, 2], 3)
t[0] += [4]
```

1. `t[0]` → the list `[1, 2]`.
2. `list.__iadd__([1,2], [4])` → mutates the list to `[1, 2, 4]`, returns the same list.
3. Store back: `t[0] = <that list>` → `tuple.__setitem__` does not exist → `TypeError`.

The mutation in step 2 already happened. The exception in step 3 cannot undo it. Hence:
error *and* mutation.

Confirm the store is unconditional:

```python
import dis
dis.dis("t[0] += x")
```

You will see `BINARY_OP` with the in-place variant followed by `STORE_SUBSCR`. The store
is compiled in; there is no branch that skips it.

The same asymmetry explains:

```python
a = [1, 2]; b = a
a += [3]        # __iadd__ mutates; b sees [1, 2, 3]
a = a + [4]     # new object; b still [1, 2, 3]
```

For immutable types there is no `__iadd__`, so `x += y` is exactly `x = x + y` and no
aliasing surprise is possible. This is precisely why `str` accumulation in a loop is
quadratic (L02 problem set) — every `+=` builds a new string.

### 2.6 Default arguments

Default parameter values are evaluated **once**, when the `def` statement executes, and
stored on the function object:

```python
def f(a, items=[]):
    items.append(a)
    return items

f.__defaults__        # ([],) — the actual list, shared by every call
```

This follows directly from §2.3: `def` is a binding operation that constructs a function
object, and constructing it requires evaluating the default expressions. There is nowhere
else for them to be evaluated.

The idiom:

```python
def f(a, items: list[int] | None = None) -> list[int]:
    if items is None:
        items = []
    items.append(a)
    return items
```

`None` as sentinel works because `None` is a singleton, so `is None` is exact and cheap. If
`None` is a legitimate value for the parameter, define your own sentinel:

```python
_MISSING = object()

def f(a, items=_MISSING):
    if items is _MISSING:
        items = []
```

A module-level `object()` instance is the cheapest unforgeable sentinel Python offers.
`enum` with a single member is the more self-documenting version, and typing-friendly.

### 2.7 Annotations do not bind

```python
x: int          # binds nothing at all
print(x)        # NameError
```

An annotation without a value records the annotation and does not create a binding. At
module and class scope the annotation is recorded in `__annotations__`; in a function body
it is not evaluated at runtime at all (only registered by the compiler as making the name
local — which matters for L07's scoping rules).

This is not a curiosity: `x: int` in a class body creates an *annotation* but no class
attribute, which is exactly what `dataclasses` and `pydantic` read to build their fields.
PY-502 L08 builds a library on this mechanism.

### 2.8 Identity, interning, and why `is` is not `==`

`a is b` asks whether the two expressions denote the same object. `a == b` asks whether
they are equal, by calling `__eq__` (L05).

CPython caches small integers (`-5` through `256`) and interns some strings, so identity
sometimes coincides with equality:

```python
a = 256; b = 256; a is b     # True on CPython
a = 257; b = 257; a is b     # True at module level (constant folding), often False if built at runtime
int("257") is int("257")     # False
```

The second line's behaviour depends on the compiler folding both constants in the same code
object into one. This is an *implementation detail that changes between versions*. Never
write code whose correctness depends on it; if a linter warns about `is` with a literal,
the linter is right.

Legitimate uses of `is`: comparison against singletons (`None`, `True`, `False`,
`NotImplemented`, `Ellipsis`, your own sentinels), and identity checks in caches, cycle
detection, and graph traversal.

### 2.9 Copying

Because assignment never copies, copying is explicit:

```python
import copy
shallow = copy.copy(obj)       # new outer object, same inner references
deep    = copy.deepcopy(obj)   # recursively new, memoized on id() to handle cycles
```

`copy.copy` on a list is equivalent to `list(xs)` or `xs[:]`. All three produce a new list
whose elements are the *same objects*. That is usually what you want and occasionally
catastrophic:

```python
grid = [[0] * 3] * 3     # three references to ONE row
grid[0][0] = 1
grid                      # [[1,0,0],[1,0,0],[1,0,0]]
```

`[expr] * n` evaluates `expr` once. The correct form is a comprehension, which evaluates
per iteration: `[[0] * 3 for _ in range(3)]`.

`deepcopy` is correct but slow and full of sharp edges: it copies things you may not want
copied (open files, locks, database connections), and its behaviour on your own classes is
customizable via `__copy__`, `__deepcopy__`, and the pickle protocol (L05 problem set).
Treat a call to `deepcopy` in production code as a design smell to be justified, not a
default.

## 3. Construction: an aliasing detector

Build a small tool that answers "which of these names refer to the same object?" — useful
in its own right and good practice at thinking in terms of objects rather than names.

**Version 1, naive.** Group names by value:

```python
def groups(ns: dict[str, object]) -> dict[object, list[str]]:
    out: dict[object, list[str]] = {}
    for name, obj in ns.items():
        out.setdefault(obj, []).append(name)   # BUG
    return out
```

Run it on `{"a": [1], "b": [1]}`. It raises `TypeError: unhashable type: 'list'`, and even
for hashable values it would merge *equal* objects, not *identical* ones — the opposite of
what we asked for. The bug is that we used the object as a dict key, which means `__hash__`
and `__eq__`, which means value semantics.

**Version 2.** Key on identity:

```python
from collections import defaultdict

def alias_groups(ns: dict[str, object]) -> list[list[str]]:
    by_id: defaultdict[int, list[str]] = defaultdict(list)
    for name, obj in ns.items():
        by_id[id(obj)].append(name)
    return [names for names in by_id.values() if len(names) > 1]
```

Now `alias_groups({"a": x, "b": x, "c": [1]})` returns `[["a", "b"]]`. Correct — but there
is a latent bug. `id()` values are only unique among *live* objects. If we stored ids and
the objects died, a later object could reuse the id. Here we hold `ns` for the duration so
we are safe, but the general lesson matters: **an `id()` is not a durable object handle.**

**Version 3.** If you need to hold identities across time, hold the objects too — or hold
weak references and accept that entries vanish:

```python
import weakref

class IdentityMap:
    """Maps objects to labels by identity, without keeping them alive."""
    def __init__(self) -> None:
        self._d: weakref.WeakValueDictionary[int, object] = weakref.WeakValueDictionary()
        self._labels: dict[int, str] = {}

    def add(self, obj: object, label: str) -> None:
        self._d[id(obj)] = obj          # keeps id valid only while obj lives
        self._labels[id(obj)] = label
```

This is still not quite right — `_labels` leaks — and fixing it properly needs
`weakref.finalize`, which is L09 material. Leave it broken and note the debt; we will
return to it.

## 4. Failure modes

### 4.1 Mutable default arguments

Covered in §2.6. Ubiquitous. Every linter catches the `[]` and `{}` cases; almost none
catch `def f(t=datetime.now())` or `def f(conn=make_connection())`, which are the same bug
with worse consequences.

### 4.2 Shared mutable class attributes

```python
class Node:
    children = []          # ONE list, shared by every instance
```

Same mechanism as 4.1: the class body executes once. `self.children.append(x)` mutates the
shared list; `self.children = [x]` creates an instance attribute that shadows it, so the
bug appears intermittently depending on which methods run first. Use `__init__` or a
dataclass `field(default_factory=list)`.

### 4.3 The `except ... as e` unbinding

```python
try:
    ...
except ValueError as e:
    pass
print(e)     # NameError: name 'e' is not defined
```

Python deletes the name at the end of the `except` block, compiling it into an implicit
`finally: del e`. The reason is L09 material — the exception holds a traceback which holds
frames which hold the exception, a reference cycle — but the consequence is here: if you
need the exception afterwards, bind it to another name first.

Worse: the deletion happens even if `e` was bound before the `try`. The name is gone.

### 4.4 Late binding in closures

```python
fs = [lambda: i for i in range(3)]
[f() for f in fs]        # [2, 2, 2]
```

The lambdas close over the *cell* holding `i`, not over its value at creation time. All
three share one cell, and after the comprehension finishes the cell holds `2`. Fixes:
bind at definition time with a default argument (`lambda i=i: i`), or use a factory
function that creates a fresh scope. L07 makes this precise.

### 4.5 Assuming `==` implies `is` or vice versa

`is` implies `==` for well-behaved types but not universally: `float('nan') == float('nan')`
is `False`, and with `x = float('nan')`, `x == x` is also `False` while `x is x` is `True`.
This means a list containing NaN behaves oddly: `x in [x]` is `True` (because `in` checks
identity first as a fast path) while `x in [float('nan')]` is `False`.

## 5. Exercises

### Warm-up (20 minutes)

**W1.** Without running it, predict the output. Then run it.

```python
a = [1, 2]
b = [a, a]
c = list(b)
a.append(3)
print(b, c, b is c, b[0] is c[0])
```

**W2.** Explain why `x = x + [1]` and `x += [1]` can produce different observable results,
and give the smallest program that demonstrates the difference.

**W3.** For each of these, say whether it binds a name and, if so, in which namespace:
`import os.path`, `for _ in range(3): pass`, `x: int`, `(y := 5)`,
`with open(f) as fh: pass`, `class C: pass`, `del z`.

### Core (2 hours)

**C1 — The tuple puzzle, generalized.** Construct three more expressions with the same
"error and side effect" property as `t[0] += [4]`, using different types and different
operators. Then construct one where the *store* succeeds but a later operation fails,
leaving a partially-updated structure. Write up in 300 words what invariant a caller can
and cannot rely on after an exception from an augmented assignment.

**C2 — A defensible sentinel.** Design a sentinel value for "argument not supplied" that:
(a) is not equal to any user value, (b) has a useful `repr`, (c) is a singleton across
module reloads, (d) type-checks cleanly under `mypy --strict` when used as a default for a
parameter typed `int | None`. Compare at least three implementations (`object()`, an
`enum.Enum` member, a custom class with `__bool__`) and state the trade-offs. Look at how
the standard library solves this (`dataclasses.MISSING`, `inspect.Parameter.empty`) and
say whether you agree with their choices.

**C3 — Finish the identity map.** Complete `IdentityMap` from §3 so that entries disappear
when their objects die, with no leak in `_labels`. Use `weakref.finalize`. Write tests that
prove the leak is gone — which means finding a way to observe collection deterministically.
Note in your write-up which parts of your test depend on CPython's reference counting and
would not hold on PyPy.

**C4 — Copy semantics.** Implement a `Config` class holding nested dicts and lists.
Give it correct `__copy__` and `__deepcopy__`. Write a property test (you may return to
this after SE-511 L04) asserting that for all configs `c`, mutating `copy.deepcopy(c)`
never changes `c`, while mutating `copy.copy(c)` may change `c` only through shared
mutable children.

### Challenge (open-ended)

**X1.** Write `explain_binding(src: str) -> str` that takes a snippet of Python source and
reports, for every name in it, which namespace the compiler assigned it to (local, cell,
free, global, builtin) and where it is bound. Use the `symtable` module. Test it against
the trickiest cases you can construct: comprehensions, `nonlocal` in nested functions,
class bodies (which do *not* participate in closures — find out why and explain), and
walrus inside a comprehension.

**X2.** Read the Language Reference §7.2 (assignment statements) in full and produce a
table of *every* target form with the exact protocol method it invokes. Then find at least
one statement in the reference that is ambiguous or that does not match observed CPython
behaviour, and write up the discrepancy. (There are some. Finding one is a distinction-level
answer.)

## 6. Self-check

Answer aloud, from memory:

1. State the three attributes every Python object has, and which of them can change.
2. Why can `x = v` never raise an exception attributable to `x`'s previous value, while
   `x.a = v` can?
3. Give the four steps of augmented assignment in order.
4. When are default argument expressions evaluated? Where is the result stored?
5. Why does `except E as e` unbind `e`?
6. What is the difference between `[[0]*3]*3` and `[[0]*3 for _ in range(3)]`, and why?
7. Name three legitimate uses of `is` and one illegitimate one.
8. What does `del x` do to the object formerly bound to `x`?

## 7. Primary sources

- Python Language Reference §3.1 (Objects, values and types) and §7.2 (Assignment
  statements). Short, and the actual specification.
- Ramalho, *Fluent Python* 2e, ch. 6 ("Object References, Mutability, and Recycling").
- PEP 572 (assignment expressions) — read the *Rationale* and *Rejected ideas* sections;
  they are an unusually good record of how a binding construct gets designed.

---

**Next:** [L02 — Types, Instances, and Metatypes](L02-types-instances-and-metatypes.md)
