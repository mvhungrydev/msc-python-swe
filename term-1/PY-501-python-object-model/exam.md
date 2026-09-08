# PY-501 — Written Examination

**Time allowed: 3 hours. Closed book. No interpreter, no notes, no search.**
**Answer FOUR questions from Section A and ONE from Section B.**
Section A questions are worth 15 marks each; Section B is worth 40.

Sit this at a desk with a timer. Mark it one week later against the rubric in
`00-program/assessment-and-rubrics.md` §4. Write your answers by hand or in a plain editor with no
tooling — the point is to find out what you can produce without a machine helping.

Where a question asks you to trace or compute, show every step. A correct answer with the working
omitted scores half marks, because the working is what is being examined.

---

## Section A — answer FOUR

### Names, objects, and binding (L01)

**A1.** State the three attributes every Python object has and say which of them can change. Then
explain why `x = v` can never raise an exception attributable to `x`'s previous value, while
`x.attr = v` can, and what that tells you about the difference between a name and an attribute.
*(15)*

**A2.** Give the four steps of augmented assignment in order. Use them to explain precisely why
`t = ([1],); t[0] += [2]` both raises `TypeError` **and** mutates the list. Then state the general
rule this illustrates about `__iadd__` and give one further case where it produces a surprise. *(15)*

**A3.** Explain when default argument expressions are evaluated and where the result is stored.
Give the classic mutable-default bug, its standard fix, and one case where a mutable default is
actually correct. Then explain why `except E as e` unbinds `e` at the end of the block, and what
breaks if you need the exception afterwards. *(15)*

### Types, instances, and metatypes (L02)

**A4.** State the five steps of executing a `class` statement, in order. Explain what determines the
metaclass when none is given explicitly, and what error is raised when that determination fails.
*(15)*

**A5.** Give the exact condition under which `__init__` is *not* called after `__new__`. Then state
when a type should customize `__new__` rather than `__init__`, giving two cases where `__new__` is
required, and explain the argument-passing obligation each imposes. *(15)*

**A6.** Explain, with reference to the object model, why subclassing `dict` to intercept writes is
unreliable. Name three operations that bypass a subclass's `__setitem__`, give the standard-library
alternative, and state the general design principle this illustrates. Then explain what
`abc.ABC.register` does and, more importantly, what it does not. *(15)*

### Attribute lookup and descriptors (L03)

**A7.** State the algorithm `object.__getattribute__` implements, in its six steps. Then, for each
of the following, say which step resolves it and what is returned:
(a) `p.x` where `x` is a `property` and `p.__dict__["x"]` exists;
(b) `c.f` where `f` is a plain function in the class body;
(c) `s.y` where the class defines `__slots__ = ("y",)` and `y` was never assigned;
(d) `o.z` where the class defines both `__getattr__` and a class attribute `z`. *(15)*

**A8.** Define data descriptor and non-data descriptor and state the priority rule between them and
the instance `__dict__`. Explain why `functools.cached_property` is deliberately *not* a data
descriptor and what would break if it were. Then give the recursion trap in a hand-written
`__getattribute__` and its fix. *(15)*

**A9.** Trace, step by step, the evaluation of `obj.method(arg)` for an ordinary instance of an
ordinary class. Name every mechanism involved, in order, from the `LOAD_ATTR` instruction to the
function body beginning execution. State two points at which a class could intervene and what each
intervention would look like. Then explain why assigning a function to an *instance* does not create
a bound method. *(15)*

### Inheritance, `super`, and the MRO (L04)

**A10.** Define C3 linearization and its three guarantees. Compute, showing every merge step, the
MRO of `Z` given `class A(O)`, `class B(O)`, `class C(O)`, `class D(A,B)`, `class E(B,C)`,
`class Z(D,E)`. Then construct a hierarchy with no valid linearization and explain which guarantee
cannot be satisfied. *(15)*

**A11.** "`super()` calls the parent class." Explain why this is wrong, state what `super()` actually
does, and give a worked example in which `super()` inside a class `B` dispatches to a class that is
not among `B`'s bases. Then state the three obligations a class has if it participates in
cooperative multiple inheritance, and describe the observable symptom of violating each. *(15)*

**A12.** Explain why cooperative methods must forward `**kwargs` rather than positional arguments,
and why a mixin must be listed before the concrete base. Then state the inherit-versus-compose
decision rule, justify it with reference to Liskov substitution, and give a concrete way in which a
base class change silently breaks subclasses where composition would not have. *(15)*

### Identity, equality, hashing, ordering (L05)

**A13.** State the hash invariant. Explain why defining `__eq__` sets `__hash__` to `None`, and why
that is the right default. Then describe precisely what goes wrong when an object is mutated while a
member of a `set` — including what `x in s` returns, what `list(s)` contains, and why the two
disagree. *(15)*

**A14.** Give the four-step dispatch order for `a == b`, including the subclass-priority rule.
Explain why `__eq__` should return `NotImplemented` rather than `False` for an unrecognised type,
and demonstrate with an example the asymmetric equality that results from returning `False`. *(15)*

**A15.** Explain why `hash("abc")` differs between interpreter runs, what attack this defends
against, and what you must therefore never do. Then give the dataclass `__hash__` generation table
(the combinations of `eq` and `frozen` and what each produces), and explain why a tolerance-based
`__eq__` is dangerous — being specific about which invariant it breaks. *(15)*

### The data model and operator dispatch (L06)

**A16.** Explain why implicit special-method calls bypass the instance dictionary — that is, why
`len(c)` ignores `c.__dict__["__len__"]`. Give the mechanism, and state two consequences for how
you would implement a proxy or wrapper object. *(15)*

**A17.** Explain the role of `NotImplemented` in binary operator dispatch. Give the full five-step
dispatch sequence for `a - b`, including the subclass-priority rule. Then explain what happens when
both operands return `NotImplemented`, and what error the user sees. *(15)*

**A18.** Distinguish an iterable from an iterator. Give a function that behaves correctly when
passed a `list` and incorrectly when passed a generator, explain why, and give two different fixes —
one at the implementation level and one at the type-annotation level. Then explain the `__getitem__`
iteration fallback and when it still applies. *(15)*

**A19.** State the two rules governing `__enter__`'s return value and `__exit__`'s return value, and
give the bug that results from misunderstanding each. Then explain why `pickle` does not call
`__init__`, what you must do about it, and what `__match_args__` is for and what sets it
automatically. *(15)*

### Scopes, frames, and closures (L07)

**A20.** Explain why `x = 1` at module level, followed by a function that prints `x` and then
assigns to `x`, raises `UnboundLocalError`. Your answer must refer to compile-time categorisation
and to the specific opcode emitted. Then list the binding constructs that make a name local. *(15)*

**A21.** What is a cell, and why is a closure over a *variable* rather than a value? Demonstrate
with the loop-variable capture bug, then give three fixes and say which you prefer and why. *(15)*

**A22.** Explain why a comprehension in a class body can see the outermost iterable but not other
class attributes. Then explain why methods do not see class-level names lexically, and why
`exec("x = 1")` inside a function does not bind `x`. All three answers should follow from the same
underlying facts — say what they are. *(15)*

### Bytecode and the evaluation loop (L08)

**A23.** State what is in a code object that is not in a function object, and vice versa. Then
explain what the bytecode for `a + b * c` proves about evaluation order, and name three things the
compiler does before the evaluation loop ever runs. *(15)*

**A24.** Explain the specialising adaptive interpreter (PEP 659): what quickening is, what a guard
does, and what causes deoptimisation. Then state three consequences for how a performance-sensitive
Python program should be written or benchmarked, including why a microbenchmark must warm up. *(15)*

**A25.** Explain zero-cost exceptions: what changed, what the cost profile now is, and one
consequence for coding style. Then explain why `assert` is unsafe for input validation, and why
allocation usually dominates instruction count in real Python programs. *(15)*

### Memory, reference counting, and finalisation (L09)

**A26.** Explain why reference counting cannot collect cycles. Describe the subtractive-count
algorithm the cycle collector uses to identify garbage, and explain the generational structure and
what triggers a collection at each generation. *(15)*

**A27.** Describe what a generation-2 collection costs and why that matters for a long-running
service with a large in-memory cache. Give two mitigations with their costs, explain what
`gc.freeze()` does and when you would call it, and list five things that commonly keep objects alive
unintentionally. *(15)*

**A28.** Give three reasons not to use `__del__` for cleanup and state the correct alternative.
Explain why a `weakref.finalize` callback must not reference the object it finalises. Then explain
why freeing 95% of a large allocation often does not reduce RSS. *(15)*

### Exceptions and cleanup (L10)

**A29.** Give the precise semantics of `finally` when the `try` block executes a `return`. Explain
what happens when `finally` also returns, and when `finally` raises. Then state the pending-action
rule in general terms, and explain why `except Exception` does not catch `KeyboardInterrupt`. *(15)*

**A30.** State what `__context__`, `__cause__` and `__suppress_context__` mean, and give a rule for
when to use `raise ... from e` versus `raise ... from None`. Then give the five cleanup mechanisms
in order of strength, say what defeats each, and explain what the `else` clause of `try` is for and
which bug it prevents. *(15)*

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

Identify **at least seven** distinct defects, spanning the object model, equality and hashing,
descriptors, attribute lookup, and lifetime. For each: state the defect precisely, give a program
that exhibits it, and give the fix. Then propose a redesign in outline and state what your redesign
gives up.

Marks are for precision and coverage, not for length. A defect stated vaguely earns nothing; a
defect stated exactly with a two-line reproduction earns full credit.

**B2. (40)** A service written by your team has these symptoms: RSS grows steadily over five days
from 400 MB to 3 GB and never falls; p999 latency shows spikes of 200–800 ms uncorrelated with
request rate; and after a restart both symptoms reset. No profiler has been run.

Write an investigation plan. It must include: your ranked hypotheses with the reasoning for each
ranking; the specific measurement you would take to discriminate between them, in order, and what
result would eliminate which hypothesis; the tools you would use and what each can and cannot see;
and the changes you would make for each outcome, with their costs.

Then, for the two most likely root causes, write the code change you would make and the test that
would prevent regression.

Marks are for the *discrimination* — a plan that would distinguish the hypotheses in three
measurements beats one that gathers everything.

**B3. (40)** Argue for or against the following proposition, with reference to specific mechanisms
covered in this course:

> "Python's object model is too dynamic. The features that make `property`, descriptors,
> metaclasses, and `__getattr__` possible cost more in comprehensibility and performance than they
> return in expressiveness, and a language designed today would not include them."

Your answer must: state the strongest version of the position you are opposing; give at least four
specific mechanisms as evidence, with concrete costs and benefits for each; address the performance
argument with reference to specialisation and inline caches; and conclude with a *falsifiable*
claim about what evidence would change your mind.

Even-handedness is assessed. An answer that only attacks the opposing view scores below one that
states it fairly and then answers it.

**B4. (40)** You are asked to design a configuration system for a large application. Requirements:
values come from defaults, a file, and the environment, with that precedence; every access must be
type-checked; unknown keys must be an error at load time, not at access time; the object must be
usable as `config.database.pool_size`; and reading a value must be fast, because it happens in a
request path.

Design it. Your answer must: state which object-model mechanisms you would use and why —
considering at minimum `__getattr__`, descriptors, `__slots__`, metaclasses, and plain dictionaries;
explain how each requirement is met by a specific mechanism; state the mechanisms you *rejected*
and why; address what `__getattr__` costs on the fast path and how you would avoid paying it
repeatedly; explain how you would make an unknown key fail at load time given that `__getattr__` is
inherently late; and state what your design does when a value is mutated at runtime, and whether it
should be permitted at all.

Then give the three tests that would most effectively catch a regression in this design, and say
what each one is protecting.

**B5. (40)** A junior engineer on your team writes:

> "I've read that `__slots__` makes classes faster and smaller, so I'm going to add it to every
> class in our codebase. I've also read that we should avoid `is` for comparisons and always use
> `==`, and that we should define `__hash__` on everything so our objects can go in sets."

Write the response you would give in a code review — not a lecture, but a technical assessment.

Your answer must: state precisely what `__slots__` does and does not do, with the actual mechanism
and the actual savings, and name at least four things it breaks or complicates; state the correct
rule for `is` versus `==`, including the three legitimate uses of `is` and why the "always use `==`"
advice is wrong in a specific case; explain what defining `__hash__` on a mutable object actually
causes, with the failure exhibited concretely; and identify the pattern of reasoning behind all
three pieces of advice and say what a better heuristic would be.

Marks are for correctness and for the judgement about *which* of the three is most wrong and why.
An answer that treats all three as equally mistaken has not weighed them.

---

## Marking guidance

For each Section A question, marks are allocated roughly:

- 6 marks — the standard correct answer, complete.
- 4 marks — precision: exact terminology, correct edge cases, no hand-waving.
- 3 marks — an example not drawn from the lessons.
- 2 marks — a stated limitation of your own answer, or a connection to another lesson.

For Section B, see §4 of the assessment rubric. A "correct" answer is a 24/40; the marks above that
are for judgement, coverage, and honesty about what you do not know.

**A note on choosing.** Section A spans all ten lessons at three questions each. If you find
yourself always able to pick four from the same two lessons, you have a coverage problem that this
exam has just diagnosed — note which lessons you avoided and revisit them.
