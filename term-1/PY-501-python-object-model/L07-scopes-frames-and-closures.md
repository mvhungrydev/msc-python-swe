# PY-501 · Lesson 07 — Scopes, Frames, and Closures

**Estimated study time:** 3.5 hours
**Prerequisites:** L01, L02

---

## 1. Orientation

```python
x = 1
def f():
    print(x)
    x = 2
f()          # UnboundLocalError: cannot access local variable 'x' where it is not associated with a value
```

The `print` fails — on a line that executes *before* the assignment. Nothing about `x` at
that moment is different from a program without line 4. Yet the presence of a later
assignment changes the meaning of an earlier read.

That is because **scope in Python is decided at compile time, for the whole function body,
before any of it runs.** This lesson makes that precise. It also explains closures,
`nonlocal`, why class bodies do not participate in closures, and why the comprehension
late-binding bug of L01 §4.4 happens.

## 2. Theory

### 2.1 The four scopes: LEGB

Name resolution searches, in order:

1. **Local** — the current function's namespace.
2. **Enclosing** — the local scopes of lexically enclosing functions, innermost first.
3. **Global** — the module's `__dict__`.
4. **Builtin** — the `builtins` module.

But "searches" overstates it. For local and cell/free variables the compiler resolves the
*category* statically and emits a different opcode for each. There is no runtime search.

### 2.2 Static categorization

When the compiler processes a function body it categorizes every name:

- **Local** if the name is *bound anywhere* in the body (assignment, `def`, `class`,
  `import`, `for` target, `with ... as`, `except ... as`, walrus, augmented assignment,
  parameter, match capture) and is not declared `global` or `nonlocal`.
- **Cell** if it is local to *this* function and referenced by a nested function.
- **Free** if it is referenced here and local to an enclosing function.
- **Global** if declared `global`, or not bound here and not free.
- **Builtin** — resolved at runtime as a fallback from global lookup.

This explains the opening puzzle exactly: `x = 2` anywhere in `f` makes `x` local
throughout `f`, so `print(x)` compiles to `LOAD_FAST x`, and the slot is empty. The error
is not "name not found" but "local variable referenced before assignment".

Inspect the categorization directly:

```python
def f():
    print(x)
    x = 2

f.__code__.co_varnames     # ('x',)   -> local
f.__code__.co_names        # ('print',) -> global/builtin
f.__code__.co_freevars     # ()
f.__code__.co_cellvars     # ()
```

Or, far better, use `symtable`:

```python
import symtable
st = symtable.symtable("def f():\n print(x)\n x=2\n", "<s>", "exec")
fn = st.get_children()[0]
for s in fn.get_symbols():
    print(s.get_name(), s.is_local(), s.is_global(), s.is_free(), s.is_assigned())
```

`symtable` is the compiler's own answer, and it is the tool for exercise X1 of L01.

### 2.3 `global` and `nonlocal`

`global x` in a function body declares that all bindings of `x` in that body target the
module namespace. `nonlocal x` declares that they target the nearest enclosing *function*
scope that binds `x` — and it is a compile-time error if no such scope exists.

Two asymmetries worth memorizing:

- `global x` works even if `x` does not exist yet (it creates the module global on first
  assignment). `nonlocal x` requires an existing binding in an enclosing function.
- `nonlocal` cannot reach module scope. `global` cannot reach an enclosing function.

There is no way to bind into an arbitrary enclosing scope. That is a deliberate design
limit and it keeps the categorization decidable at compile time.

### 2.4 Cells and closures

A closure is a function plus captured bindings. The captured bindings are **cells** — one
level of indirection, so that all closures over the same variable see the same, mutable
binding:

```python
def counter():
    n = 0
    def inc():
        nonlocal n
        n += 1
        return n
    return inc

c = counter()
c(), c(), c()          # 1, 2, 3
inc = c
inc.__closure__        # (<cell at 0x...: int object at 0x...>,)
inc.__closure__[0].cell_contents   # 3
inc.__code__.co_freevars           # ('n',)
```

The function object holds `__closure__`, a tuple of cells parallel to `co_freevars`. The
enclosing frame is *not* retained — only the cells are. That matters for memory: a closure
over one variable of a function with a huge local does not keep the huge local alive.

**Cells capture variables, not values.** This is the whole of L01 §4.4:

```python
fs = [lambda: i for i in range(3)]     # all three share one cell
[f() for f in fs]                       # [2, 2, 2]
```

Fixes, in order of preference:

```python
fs = [lambda i=i: i for i in range(3)]          # bind at definition time via default
fs = [functools.partial(lambda i: i, i) for i in range(3)]
def make(i): return lambda: i                    # fresh scope per call
fs = [make(i) for i in range(3)]
```

The default-argument fix works because defaults are evaluated at `def`/`lambda` execution
(L01 §2.6), which happens once per loop iteration.

### 2.5 Comprehension scope

Since Python 3, comprehensions and generator expressions execute in **their own function
scope**. The loop variable does not leak:

```python
[i for i in range(3)]
i                       # NameError
```

Consequences:

- The iterable of the *outermost* `for` is evaluated in the enclosing scope and passed as
  an argument; everything else runs inside.
- A comprehension can read enclosing locals (they become free variables) but assigning to
  them requires the walrus, which by explicit design binds in the *enclosing* scope:
  `[y := f(x) for x in xs]` leaves `y` bound outside.
- Inside a **class body**, a comprehension cannot see class-level names, because class
  bodies do not create a closure scope (§2.6):

```python
class C:
    xs = [1, 2, 3]
    ys = [x * 2 for x in xs]        # works — xs is the outermost iterable, evaluated outside
    zs = [x * n for x in xs]        # NameError if n is a class attribute
```

This trips people constantly. `xs` works because the outermost iterable is special-cased;
any *other* class-level name is invisible.

*(Version note: CPython 3.12 inlined comprehensions for speed — PEP 709 — which changes
the frame structure and some tracebacks but preserves the scoping semantics above. Check
`dis` output on your version; if the comprehension no longer produces a separate code
object, you are on 3.12+.)*

### 2.6 Class bodies are not closures

A class body executes in its own namespace, but that namespace **does not participate in
lexical scoping for nested functions**:

```python
class C:
    n = 10
    def m(self):
        return n          # NameError at call time — not 10
```

Methods do not see class attributes as free variables. They must go through `self.n` or
`C.n`. The reason is that a class namespace is a mapping built at class-creation time and
discarded into `cls.__dict__`; making it a closure scope would require it to behave like a
function scope, which would break `__prepare__`, dynamic class bodies, and attribute
semantics.

The exception is the implicit `__class__` cell (L04 §2.3), which the compiler inserts
specifically so that zero-argument `super()` works.

### 2.7 Frames

Every call creates a **frame object** holding: the code object, the local variable storage,
the value stack, the instruction pointer, a reference to the globals dict, and a link to
the caller (`f_back`). Frames form the call stack; a traceback is a chain of frames.

```python
import sys
def g():
    fr = sys._getframe()
    print(fr.f_code.co_name, fr.f_back.f_code.co_name, fr.f_lineno)
```

Frames are ordinary Python objects, which has three important consequences:

1. **Generators keep their frame alive.** A suspended generator holds its frame with all
   locals intact — that is what makes resumption possible, and it means locals in a
   generator that is never exhausted are never freed.
2. **Tracebacks keep frames alive.** An exception's `__traceback__` references frames,
   which reference locals, which may reference large objects. Storing an exception in a
   long-lived structure retains everything on the stack at raise time. This is the most
   common source of "why is my memory not going down" in Python services (L09).
3. **`locals()` in a function returns a snapshot**, because locals live in an array, not a
   dict. Mutating the returned dict does not change the variables.
   *(Version note: PEP 667, landing in 3.13, changes `locals()` in optimized scopes to
   return an independent snapshot with well-defined write semantics through
   `FrameType.f_locals`. Check your version's behaviour before relying on either.)*

### 2.8 `eval`, `exec`, and dynamic scope

`eval(expr, globals, locals)` and `exec(stmt, globals, locals)` take explicit namespaces.
Code executed by them is compiled *without* knowledge of the enclosing function's
categorization, so it cannot see enclosing-function locals as free variables:

```python
def f():
    y = 1
    return eval("y")        # works — reads from locals() snapshot
def g():
    y = 1
    exec("y = 2")
    return y                # 1 — the exec wrote to a snapshot
```

`eval` reads a snapshot (so it happens to work); `exec` writes to the snapshot and the
write is lost. This asymmetry is why dynamic code generation in Python uses explicit dicts
and returns results rather than assigning into the caller's scope — which is exactly what
`dataclasses` does when it generates `__init__` via `exec` into a fresh namespace.

## 3. Construction: a scope explainer

**Version 1 — read the code object.**

```python
def describe(fn) -> dict[str, list[str]]:
    c = fn.__code__
    return {
        "args": list(c.co_varnames[: c.co_argcount]),
        "locals": list(c.co_varnames[c.co_argcount :]),
        "cells": list(c.co_cellvars),
        "free": list(c.co_freevars),
        "globals/attrs": list(c.co_names),
    }
```

Useful, and immediately limited: `co_names` mixes globals with attribute names, and nested
functions are separate code objects that this does not descend into.

**Version 2 — use `symtable`,** which is what the compiler used:

```python
import symtable

def explain(src: str, name: str = "<src>") -> str:
    lines: list[str] = []
    def walk(tbl, depth=0):
        pad = "  " * depth
        lines.append(f"{pad}{tbl.get_type()} {tbl.get_name()!r}")
        for s in sorted(tbl.get_symbols(), key=lambda s: s.get_name()):
            kinds = [
                k for k, ok in [
                    ("param", s.is_parameter()), ("local", s.is_local()),
                    ("global", s.is_global()), ("free", s.is_free()),
                    ("cell", s.is_assigned() and s.is_local() and not s.is_parameter()),
                    ("imported", s.is_imported()), ("assigned", s.is_assigned()),
                ] if ok
            ]
            lines.append(f"{pad}  {s.get_name():<16} {', '.join(kinds)}")
        for child in tbl.get_children():
            walk(child, depth + 1)
    walk(symtable.symtable(src, name, "exec"))
    return "\n".join(lines)
```

Run it on the opening puzzle, on a closure, on a comprehension in a class body, and on a
walrus inside a nested comprehension. Every surprise in this lesson becomes visible.

**Version 3 — add the diagnosis.** Detect and report the three classic bugs: (a) a name
read before assignment where the assignment makes it local; (b) a closure over a loop
variable; (c) a class-body name referenced from a method. Each is a pattern in the symbol
table plus a little AST. This is a genuinely useful lint that no mainstream tool fully
implements.

## 4. Failure modes

- **`UnboundLocalError` from a later assignment.** §2.2.
- **Loop-variable capture.** §2.4.
- **Expecting a comprehension in a class body to see class attributes.** §2.5.
- **Expecting a method to see class-level names without `self.`.** §2.6.
- **Mutating `locals()`.** No effect in a function.
- **`exec("x = ...")` expecting to bind in the caller.** §2.8.
- **Retaining exceptions.** §2.7 (2). If you must store one, store
  `traceback.format_exc()` (a string), or explicitly `e.__traceback__ = None`.
- **Retaining generators.** A partially-consumed generator held in a cache holds its frame
  and everything in it.
- **`global` as a substitute for state design.** Beyond scoping mechanics: a module global
  mutated from multiple functions is a shared mutable singleton with no lifecycle and no
  thread safety (PY-601 L04). It works until it doesn't.

## 5. Exercises

### Warm-up (25 min)

**W1.** Predict, then verify, the output of six variants of the opening puzzle: with
`global x`, with `x` only read, with `del x`, with `x` bound in a nested `if False:` block,
with `x` as a `for` target, with `x` bound only in an `except ... as x`.

**W2.** Show that all closures from a loop share one cell, by printing
`f.__closure__[0]` identity for each.

**W3.** Write a function whose `co_cellvars` is non-empty, and one whose `co_freevars` is
non-empty, and explain the difference.

### Core (2 h)

**C1 — The scope explainer.** Complete version 3 of §3. It must handle: nested functions
three deep, comprehensions (both pre- and post-3.12 inlining — say which you tested),
class bodies, `global`/`nonlocal`, walrus in comprehensions, and lambdas. Test it against
20 snippets you write specifically to break it.

**C2 — Closure memory.** Construct a function with a 100 MB local and a nested closure over
one small variable. Show that returning the closure does *not* retain the 100 MB. Then
construct a case where it *does* (hint: capture something that transitively references it,
or use a generator instead of a closure). Measure with `tracemalloc`. Write 300 words on
what a reviewer should look for.

**C3 — The exception-retention leak.** Build a small service loop that catches exceptions
and appends them to a list for later reporting. Show the memory growth. Then show the three
fixes (format immediately, clear `__traceback__`, use a bounded structure) and measure
each. Explain why `except ... as e` deletes `e` (L01 §4.3) in light of what you found.

**C4 — Generated code.** Read `dataclasses._create_fn` in the standard library. Explain why
it builds source text and `exec`s it rather than composing a function object directly, what
namespaces it passes, and how it makes the generated function's closure work. Then write
your own miniature version that generates an `__init__` from a field list, and compare
generated-code readability in tracebacks.

### Challenge

**X1.** Implement `bind_into_caller(name, value)` — a function that binds a name in its
*caller's* scope. Do it with `sys._getframe`. Then demonstrate why it works for module-level
callers and fails for function-level callers, and connect this to §2.7's array-vs-dict
storage. Finally, write 400 words on whether PEP 667 changes your answer.

**X2.** Read PEP 709 (inlined comprehensions) and measure the difference on your version:
construct a benchmark where inlining is visible, and a case where the change to traceback
structure is observable. Report both.

## 6. Self-check

1. Why does an assignment later in a function change the meaning of an earlier read?
2. List the binding constructs that make a name local.
3. What is a cell, and why is a closure over a variable rather than a value?
4. Give three fixes for loop-variable capture and say which you prefer and why.
5. Why can a comprehension in a class body see the outermost iterable but no other class
   attribute?
6. Why do methods not see class-level names lexically?
7. Name two things that keep frames alive after the call returns.
8. Why does `exec("x = 1")` inside a function not bind `x`?

## 7. Primary sources

- Language Reference §4.2 (Naming and binding). Two pages; the definitive statement.
- PEP 227 (statically nested scopes), PEP 3104 (`nonlocal`), PEP 572 (walrus — the scoping
  section), PEP 709 (inlined comprehensions), PEP 667 (consistent `locals()`).
- `symtable` module documentation and source.

---

**Previous:** [L06](L06-data-model-and-operator-dispatch.md) · **Next:**
[L08 — Bytecode and the Evaluation Loop](L08-bytecode-and-the-evaluation-loop.md)
