# Placement Self-Assessment

**Purpose.** Find out which Term 1 material you can skim and which you must study. This is
not a quiz to feel good about. Answer from memory, in writing, without a REPL. Then check
against the answer key and score yourself severely: partial credit is a lie you tell
yourself.

**Time:** 90 minutes. **No interpreter, no search.**

---

## Section A — Python semantics (10 questions)

**A1.** Explain the difference between a *name*, a *reference*, and an *object* in Python.
Then explain precisely what `x = y` does, and what it does not do.

**A2.** What does this print, and why?

```python
def f(a, items=[]):
    items.append(a)
    return items

print(f(1)); print(f(2)); print(f(3))
```

**A3.** Given `class A: pass` and `a = A()`, describe the full lookup sequence Python
performs for `a.x`, including every hook it may call and in what order.

**A4.** What is the difference between `__getattr__` and `__getattribute__`? When is each
invoked?

**A5.** Explain why this is not equivalent to a `for` loop over `range(3)`:

```python
fs = [lambda: i for i in range(3)]
print([f() for f in fs])
```

What does it print, and what would you change to get `[0, 1, 2]`?

**A6.** Define the method resolution order. State the three constraints C3 linearization
satisfies. Give a class hierarchy for which no valid MRO exists.

**A7.** What is the difference between `is` and `==`? Under what circumstances does
`a is b` hold for two separately constructed small integers or short strings, and why is
relying on that a bug?

**A8.** Explain what a generator's `yield` does to the frame. What happens to the frame
between `next()` calls? What is the lifetime of local variables in a suspended generator?

**A9.** Describe the semantics of `try/finally` when the `try` block contains a `return`.
What if the `finally` block also contains a `return`?

**A10.** Explain reference counting and why CPython additionally needs a cycle collector.
Give a two-object cycle that reference counting alone would leak.

## Section B — Software construction (8 questions)

**B1.** Define, precisely, the difference between a *stub*, a *mock*, a *fake*, and a
*spy*. Give a case where using a mock makes a test worse.

**B2.** What is the difference between coverage and adequacy? Give a function with 100%
line coverage and an obvious untested bug.

**B3.** State the property-based testing idea in one paragraph. Write (in prose) three
properties you would assert about a `sort` function that together nearly pin down its
behaviour.

**B4.** What does `mypy --strict` actually check that a normal run does not? Name three
specific behaviours.

**B5.** Explain covariance and contravariance using a Python container and a callable.
Why is `list[Derived]` not a subtype of `list[Base]`?

**B6.** Describe what a lockfile solves. Why is `pip install -r requirements.txt` with
pinned versions still not fully reproducible?

**B7.** What is the difference between a package's *runtime* dependencies and its *build*
dependencies, and where is each declared in a modern `pyproject.toml`?

**B8.** Why do flaky tests cause more damage than failing tests?

## Section C — Design and architecture (6 questions)

**C1.** State Parnas's information-hiding criterion for modular decomposition. How does it
differ from decomposing by processing step?

**C2.** Define coupling and cohesion. Give an example of code with low coupling and low
cohesion — and explain why that combination is still bad.

**C3.** What problem does dependency inversion solve? Give a case where applying it makes
a system worse.

**C4.** Explain the difference between an interface being *narrow* and being *small*.

**C5.** What is an architectural decision record and what makes a bad one?

**C6.** Sketch the trade-off between a shared library and a service for code reuse across
teams. Name at least four dimensions on which they differ.

## Section D — Theory readiness (6 questions)

**D1.** State the formal definition of `O(f(n))`. Then state `Θ` and `Ω`. Prove or
disprove: `n log n = O(n^1.01)`.

**D2.** Give the recurrence for merge sort and solve it. State the Master Theorem.

**D3.** What does it mean for a problem to be NP-complete? Name the two things you must
show.

**D4.** Explain amortized analysis using the dynamic array (list append) as the example.

**D5.** What is a graph's topological order and when does one exist?

**D6.** Explain, informally but correctly, why the halting problem is undecidable.

## Section E — Systems and distribution (6 questions)

**E1.** Explain why a wall-clock timestamp cannot be used to order events across two
machines, even with NTP.

**E2.** Define linearizability. Define eventual consistency. Is a linearizable system
necessarily available under partition?

**E3.** What does a CPU cache line have to do with the performance of a Python program?

**E4.** Explain the difference between concurrency and parallelism with an example of each
that the other does not cover.

**E5.** Describe what happens, at the OS level, from `socket.recv()` being called to data
arriving in your Python `bytes` object.

**E6.** Two services each with 99.9% availability are called in sequence by a third. What
is the composite availability, and what assumption did you just make?

---

## Scoring

Award yourself:

- **2** — answered correctly and completely, would survive a follow-up "why?"
- **1** — got the gist, missed detail or precision
- **0** — didn't know, or produced something confidently wrong

| Section | Max | Interpretation |
|---|---|---|
| A — Python semantics | 20 | ≥16: skim PY-501 L01–L04, study L05–L10. <10: study all of PY-501 carefully. |
| B — Construction | 16 | ≥13: skim SE-511 L01–L03. <8: SE-511 is your highest priority course. |
| C — Design | 12 | ≥10: you may run SE-521 concurrently with Term 1. <6: do SE-521 in sequence. |
| D — Theory | 12 | ≥9: CS-621 can be accelerated. <5: budget 1.5× the nominal hours for Term 4. |
| E — Systems | 12 | ≥9: consider pulling DS-701 forward one term. <5: do not skip PY-602. |

**Total ≥ 58/72:** You are ready for graduate-level work; treat Term 1 as review and
consolidation, but do not skip its problem sets — they build the habits the later terms
assume.

**Total 35–57:** The intended starting point. Follow the sequence as written.

**Total < 35:** Also fine, and more common than you'd think for experienced engineers who
learned on the job. Follow the sequence, take the nominal hours seriously, and do not read
ahead. You will close the gap in Term 1 and Term 2.

---

## Answer key — brief

Full worked answers live in the lessons they belong to; here is enough to grade yourself.

**A1.** A name is a binding in a namespace (a dict, or a fast-locals array); it refers to
an object. Objects have identity, type, and value. `x = y` binds the name `x` to the same
object `y` refers to; it copies a reference, not the object, and it does not call any
method on the object (unlike `x.attr = y`, which does).

**A2.** `[1]`, `[1, 2]`, `[1, 2, 3]`. Default arguments are evaluated once, at function
definition time, and stored on the function object (`f.__defaults__`).

**A3.** `type(a).__getattribute__(a, 'x')` runs. It looks up `'x'` in `type(a).__mro__`;
if found and it is a *data descriptor* (defines `__set__` or `__delete__`), its `__get__`
is called and wins. Otherwise `a.__dict__['x']` is consulted. Otherwise the class-level
value is used, and if it is a *non-data descriptor* its `__get__` is called. If nothing is
found, `AttributeError` is raised — and only then is `type(a).__getattr__` tried.

**A4.** `__getattribute__` is called unconditionally for every attribute access;
`__getattr__` is called only as a fallback after normal lookup raises `AttributeError`.

**A5.** Prints `[2, 2, 2]`. The lambdas close over the *variable* `i`, not its value, and
the comprehension's `i` has a single binding cell reused for each iteration. Fix with a
default argument (`lambda i=i: i`) or a factory function.

**A6.** MRO is the linear order in which base classes are searched. C3 preserves: local
precedence order (a class's bases appear in the order written), monotonicity (a subclass's
MRO is consistent with its parents' MROs), and the class itself precedes its bases. No MRO
exists for e.g. `class A: pass; class B: pass; class X(A,B): pass; class Y(B,A): pass;
class Z(X,Y): pass`.

**A7.** `is` compares identity, `==` compares value via `__eq__`. CPython interns small
ints (−5..256) and some strings, so identity may coincidentally hold; this is an
implementation detail and varies by version and context (compile-time constant folding).

**A8.** `yield` suspends the frame, saving the instruction pointer and value stack; the
frame object persists, referenced by the generator, so locals stay alive until the
generator is exhausted or collected.

**A9.** `finally` runs before the function actually returns; the return value is computed
first, then `finally` executes. A `return` in `finally` overrides the pending return *and*
swallows any in-flight exception.

**A10.** Refcounts are incremented/decremented on each new/removed reference; at zero the
object is freed. Cycles keep counts non-zero: `a = []; b = [a]; a.append(b)` leaks without
the generational cycle collector.

**B1.** Stub returns canned answers; fake is a working lightweight implementation; mock
asserts on interactions; spy records calls while delegating. Mocks make tests worse when
they pin down *how* rather than *what*, coupling the test to the implementation.

**B2.** Coverage says a line ran; adequacy says the behaviour was checked. `def f(x):
return x / y` with a test that never passes `y == 0` has full line coverage.

**B4.** `--strict` turns on (among others) `--disallow-untyped-defs`,
`--disallow-any-generics`, `--warn-return-any`, `--no-implicit-optional`, and strict
equality checks.

**B5.** A mutable container must be invariant: if `list[Derived] <: list[Base]`, you could
append a `Base` through the `Base`-typed alias and break the `Derived` invariant.
Callables are contravariant in parameters, covariant in return.

**D1.** `f = O(g)` iff ∃c>0, n₀ such that ∀n≥n₀, `f(n) ≤ c·g(n)`. Yes,
`n log n = O(n^1.01)` since `log n = O(n^0.01)`.

**E6.** 99.8% — assuming independence of failures, which is almost never true (shared
dependencies, correlated deploys, shared network).
