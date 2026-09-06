# PY-501 — Written Examination

**Time allowed: 3 hours. Closed book. No interpreter, no notes, no search.**
**Answer FOUR questions from Section A and ONE from Section B.**
Section A questions are worth 15 marks each; Section B is worth 40.

Sit this at a desk with a timer. Mark it one week later against the rubric in
`00-program/assessment-and-rubrics.md` §4. Write your answers by hand or in a plain editor
with no tooling — the point is to find out what you can produce without a machine helping.

---

## Section A — answer FOUR

**A1.** Trace, step by step, the evaluation of `obj.method(arg)` for an ordinary instance
of an ordinary class. Name every mechanism involved, in order, from the `LOAD_ATTR`
instruction to the function body beginning execution. State two points at which a class
could intervene, and what each intervention would look like. *(15)*

**A2.** State the algorithm `object.__getattribute__` implements. Then, for each of the
following, say which step of the algorithm resolves it and what is returned:
(a) `p.x` where `x` is a `property` and `p.__dict__["x"]` exists;
(b) `c.f` where `f` is a plain function in the class body;
(c) `s.y` where the class defines `__slots__ = ("y",)` and `y` was never assigned;
(d) `o.z` where the class defines both `__getattr__` and a class attribute `z`.
*(15)*

**A3.** Define C3 linearization and its three guarantees. Compute, showing every merge
step, the MRO of `Z` given `class A(O)`, `class B(O)`, `class C(O)`, `class D(A,B)`,
`class E(B,C)`, `class Z(D,E)`. Then construct a hierarchy with no valid linearization and
explain which guarantee cannot be satisfied. *(15)*

**A4.** "`super()` calls the parent class." Explain why this is wrong, state what `super()`
actually does, and give a worked example in which `super()` inside a class `B` dispatches
to a class that is not among `B`'s bases. Then state the three obligations a class has if
it participates in cooperative multiple inheritance, and describe the observable symptom of
violating each. *(15)*

**A5.** State the hash invariant. Explain why defining `__eq__` sets `__hash__` to `None`,
and why that is the right default. Then describe, precisely, what goes wrong when an object
is mutated while a member of a `set` — include what `x in s` returns, what `list(s)`
contains, and why the two disagree. *(15)*

**A6.** Explain the role of `NotImplemented` in binary operator dispatch. Give the full
dispatch sequence for `a - b`, including the subclass-priority rule. Then explain why
returning `False` from `__eq__` instead of `NotImplemented` produces asymmetric equality,
with an example. *(15)*

**A7.** Distinguish an iterable from an iterator. Give a function that behaves correctly
when passed a `list` and incorrectly when passed a generator, explain why, and give two
different fixes — one at the implementation and one at the type-annotation level. Then
explain the `__getitem__` iteration fallback and when it still applies. *(15)*

**A8.** Explain why `x = 1` at module level followed by a function that prints `x` and then
assigns to `x` raises `UnboundLocalError`. Your answer must refer to compile-time
categorization and to the specific opcode emitted. Then explain why a comprehension in a
class body can see the outermost iterable but not other class attributes. *(15)*

**A9.** Describe CPython's cycle collector: what it tracks, the algorithm it uses to
identify garbage, and the generational structure. Then explain the operational consequence
for a long-running service with a large in-memory cache, and describe two mitigations with
their costs. *(15)*

**A10.** Give the precise semantics of `finally` when the `try` block executes a `return`.
Explain what happens when `finally` also returns, and when `finally` raises. Then state
what `__context__`, `__cause__`, and `__suppress_context__` mean, and give a rule for when
to use `raise ... from e` versus `raise ... from None`. *(15)*

**A11.** Explain the specializing adaptive interpreter (PEP 659): what quickening is, what
a guard does, and what causes deoptimization. Then state three consequences for how a
performance-sensitive Python program should be written or benchmarked. *(15)*

**A12.** Explain, with reference to the object model, why subclassing `dict` to intercept
writes is unreliable. Name three operations that bypass a subclass's `__setitem__`, give
the standard-library alternative, and state the general design principle this illustrates.
*(15)*

---

## Section B — answer ONE

**B1. (40)** You are reviewing a colleague's ORM-style library. Its central class is:

```python
class Model:
    def __init__(self, **kwargs):
        for k, v in kwargs.items():
            setattr(self, k, v)

    def __eq__(self, other):
        return self.__dict__ == other.__dict__

    def __getattr__(self, name):
        return self._data.get(name)

    class Meta:
        pass

class Field:
    def __init__(self, name, type_):
        self.name, self.type = name, type_
    def __get__(self, obj, objtype=None):
        return obj._data[self.name]
    def __set__(self, obj, value):
        if not isinstance(value, self.type):
            raise TypeError
        obj._data[self.name] = value
```

Identify **at least seven** distinct defects, spanning the object model, equality/hashing,
descriptors, attribute lookup, and lifetime. For each: state the defect precisely, give a
program that exhibits it, and give the fix. Then propose a redesign in outline and state
what your redesign gives up.

Marks are for precision and coverage, not for length. A defect stated vaguely earns
nothing; a defect stated exactly with a two-line reproduction earns full credit.

**B2. (40)** A service written by your team has these symptoms: RSS grows steadily over
five days from 400 MB to 3 GB and never falls; p999 latency shows spikes of 200–800 ms
uncorrelated with request rate; and after a restart both symptoms reset. No profiler has
been run.

Write an investigation plan. It must include: your ranked hypotheses with the reasoning for
each ranking; the specific measurement you would take to discriminate between them, in
order, and what result would eliminate which hypothesis; the tools you would use and what
each can and cannot see; and the changes you would make for each outcome, with their costs.

Then, for the two most likely root causes, write the code change you would make and the
test that would prevent regression.

Marks are for the *discrimination* — a plan that would distinguish the hypotheses in three
measurements beats one that gathers everything.

**B3. (40)** Argue for or against the following proposition, with reference to specific
mechanisms covered in this course:

> "Python's object model is too dynamic. The features that make `property`, descriptors,
> metaclasses, and `__getattr__` possible cost more in comprehensibility and performance
> than they return in expressiveness, and a language designed today would not include them."

Your answer must: state the strongest version of the position you are opposing; give at
least four specific mechanisms as evidence, with concrete costs and benefits for each;
address the performance argument with reference to specialization and inline caches; and
conclude with a *falsifiable* claim about what evidence would change your mind.

Even-handedness is assessed. An answer that only attacks the opposing view scores below one
that states it fairly and then answers it.

---

## Marking guidance

For each Section A question, marks are allocated roughly:

- 6 marks — the standard correct answer, complete.
- 4 marks — precision: exact terminology, correct edge cases, no hand-waving.
- 3 marks — an example not drawn from the lessons.
- 2 marks — a stated limitation of your own answer, or a connection to another lesson.

For Section B, see §4 of the assessment rubric. A "correct" answer is a 24/40; the marks
above that are for judgement, coverage, and honesty about what you do not know.
