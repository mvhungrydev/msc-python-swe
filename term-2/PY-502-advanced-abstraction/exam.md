# PY-502 — Written Examination

**Time allowed: 3 hours. Closed book.**
**Answer FOUR from Section A and ONE from Section B.**
Section A: 15 marks each. Section B: 40 marks.

---

## Section A — answer FOUR

### Protocols and interface design (L01)

**A1.** Give the two properties of a good interface and explain the tension between them. Then name
four things duck typing gives up, and say what recovers each. *(15)*

**A2.** Explain why a `Protocol` lets the *consumer* own the interface, and why that matters for
dependency direction. Then give four criteria that select an ABC over a Protocol, and state what
`@runtime_checkable` actually checks. *(15)*

**A3.** State the expression problem and say which form of polymorphism suits which axis of change.
Explain why raising `NotImplementedError` for an optional capability is a poor design and what to do
instead, and name something Python's type system cannot express about an interface and what fills
the gap. *(15)*

### Descriptors as a design tool (L02)

**A4.** Give the six-step decision ladder for choosing an attribute mechanism, and the specific
question that selects a descriptor over the rungs below it. *(15)*

**A5.** Name the four descriptor storage strategies and give one fatal flaw of each. Explain when
`__set_name__` is *not* called and what breaks as a result. *(15)*

**A6.** Explain why descriptors compose by wrapping rather than by inheritance. State what bypasses
`__set__` validation and what you do about it, and explain what the `__get__` overloads tell a type
checker and what happens without them. *(15)*

### Class construction and metaclasses (L03)

**A7.** Give the six steps of class creation in order, stating exactly where `__set_name__` and
`__init_subclass__` fire relative to the others. *(15)*

**A8.** Name the four things only a metaclass can do. Explain why metaclasses do not compose, what
that costs a library's users, and why `len(MyClass)` must be defined on the metaclass. *(15)*

**A9.** State the difference between `__init_subclass__` and a class decorator with respect to
inheritance, and when each is correct. Give the failure mode of implicit registration and three
fixes. Then quote Tim Peters's rule about metaclasses and say when it is wrong. *(15)*

### Decorators (L04)

**A10.** State exactly what `functools.wraps` copies and what `__wrapped__` is for. Explain why a
decorator applied to a method receives `self` as an ordinary argument, and why `classmethod` must be
outermost. *(15)*

**A11.** Give three decorator pairs where the order of application changes the semantics, and say
what each order means. Then name the three places decorator state can live and the lifetime of each.
*(15)*

**A12.** Explain what `ParamSpec` fixes and what `Concatenate` adds. Then explain why a synchronous
wrapper silently breaks an async function, how you detect one at decoration time, and give the
three-part rule for when a decorator is the right tool. *(15)*

### Generators and lazy evaluation (L05)

**A13.** What happens when you *call* a generator function — what runs and what does not? Explain
where argument validation must go as a result, and why. *(15)*

**A14.** Give the four generator protocol operations and what each does. Enumerate the four cases in
which a `with` block inside a generator exits, and say which of them is the dangerous one. *(15)*

**A15.** Name five costs of laziness. Explain why `groupby` requires sorted input and what the
failure mode looks like, when `tee` defeats the purpose of a lazy pipeline, and what is meant by
"the stage that decides the memory profile". *(15)*

### Coroutines and delegation (L06)

**A16.** Give the four things `yield from` does that a `for` loop does not. Explain what priming is
and why it is required. *(15)*

**A17.** Give the three type parameters of `Generator` in order and what each means. Explain how
`@contextmanager`'s `__exit__` is implemented in terms of the generator protocol, and why the
`try/finally` in the generator body is mandatory rather than good practice. *(15)*

**A18.** Relate `await` to `yield from` at the interpreter level. Explain `CancelledError` in terms
of the generator protocol, state what sans-I/O design is and what it buys, and distinguish an async
generator from a coroutine in both return type and protocol. *(15)*

### Context managers and resources (L07)

**A19.** State the two rules of the context manager protocol. Distinguish reusable from reentrant
with an example of each, and give a context manager that is one and not the other. *(15)*

**A20.** Explain the problem `ExitStack` solves that nested `with` statements cannot. Describe
`pop_all()` and the transfer-of-ownership pattern, and give a case where it is the only correct
answer. *(15)*

**A21.** Give the three-layer resource discipline and the role of each layer. Answer both cleanup
questions — should a cleanup failure mask the original exception, and should one failure stop the
remaining cleanups — and justify each. Then explain why `ContextVar.reset(token)` exists rather than
simply setting the old value back. *(15)*

### Declarative models (L08)

**A22.** Name the four jobs a model class may be asked to do and explain why conflating them is a
design error rather than a convenience. *(15)*

**A23.** Give three things `attrs` provides that `dataclasses` does not. State what `pydantic` is
actually for and identify its most dangerous default, with the failure it produces. *(15)*

**A24.** State the two-layer rule and the cost it imposes. Give three evolutions that are cheap
under it and expensive without it, explain the class of security bug the split prevents, and say why
`kw_only=True` should be the default. *(15)*

### Internal DSLs (L09)

**A25.** Give the three properties that justify building a DSL. Name six DSL construction techniques
and one hazard of each. *(15)*

**A26.** Explain why `and` and `or` cannot be overloaded and what follows for expression-building
APIs. Explain what breaks when `__eq__` returns a non-boolean, and give the three validation timings
in order of preference with the reason for the ordering. *(15)*

**A27.** State what an escape hatch needs to be safe, and describe the classic escape-hatch
vulnerability. Give the ten-uses test, and name the single investment that most distinguishes a good
internal DSL from a bad one. *(15)*

### The cost of abstraction (L10)

**A28.** Name the six costs of abstraction. State Ousterhout's deep/shallow distinction and give one
example of each from code you have actually read. *(15)*

**A29.** Explain why duplication is cheaper than the wrong abstraction, being specific about why the
two costs are asymmetric rather than merely different. Then give the mechanism ladder in order and
the question that moves you up a rung. *(15)*

**A30.** Explain what a boolean behaviour flag on a function indicates about its design. Give six
signals that an abstraction should be removed, describe "inline, then re-extract" and why in that
order, and state the correct criterion for YAGNI — which is not likelihood. *(15)*

---

## Section B — answer ONE

**B1. (40)** You are asked to review this proposal from a colleague:

> "Our fifty API handlers all repeat the same boilerplate: parse the body, validate,
> authorize, log, handle errors, serialize the response. I propose a `Handler` base class
> with a metaclass that reads type annotations on the `handle` method to generate parsing
> and validation, registers the handler by its `path` class attribute, wraps `handle` with
> authorization based on a `permissions` class attribute, and generates the OpenAPI schema.
> Each handler then becomes six lines."

Write the review. It must include:

- What is genuinely good about the proposal, stated fairly and first.
- Each mechanism it proposes, placed on the ladder of L10 §2.4, with what the next mechanism
  to the left would cost and buy.
- The specific tooling losses, named precisely (type checking, autocomplete, go-to-
  definition, grep, debugger, traceback quality).
- At least three failure modes from this course that the proposal walks into.
- A counter-proposal achieving most of the benefit with less machinery, in outline.
- The conditions under which you would approve the original as proposed — there are some,
  and identifying them fairly is worth marks.

**B2. (40)** Design a plugin system for a data-processing application. Third parties write
plugins; plugins declare inputs and outputs with types; the host validates compatibility
before running a pipeline; plugins may hold resources; plugins may be sync or async.

Your answer must cover:

- The declaration mechanism, chosen from the ladder, with justification against the two
  mechanisms adjacent to your choice.
- How plugins are discovered, and why *not* import side effects.
- The resource lifecycle, including acquisition failure partway through a pipeline of
  plugins, cleanup failure, and cancellation in the async case.
- What the type checker knows about a plugin, and for the plugin author and the host
  separately.
- The error messages a plugin author sees for the five most likely mistakes.
- The escape hatch.
- What you deliberately did not build, and why.

**B3. (40)** Argue for or against:

> "Python's metaprogramming facilities are a net negative for the ecosystem. The frameworks
> they enable — ORMs, dependency injectors, declarative validators — are individually
> impressive and collectively responsible for most of the difficulty new engineers have
> reading production Python. A language without descriptors and metaclasses would have a
> healthier ecosystem."

Required: state the opposing position at its strongest; give at least four specific
mechanisms as evidence with concrete costs and benefits; address the *typed* Python argument
separately (the costs have changed since 2015 and this is where the interesting argument
is); and conclude with a falsifiable claim about what evidence would change your mind.

Even-handedness is assessed. An answer that does not state the opposing case fairly cannot
score above 24.

---

**B4. (40)** A library in your organisation has grown a configuration layer with the following
features, added over three years: a metaclass that registers subclasses; a class decorator that adds
validation; `__init_subclass__` hooks in two base classes; descriptors for typed fields; a
`__getattr__` fallback for legacy key names; and a `pydantic` model wrapping the whole thing for
external input. Users report that the error messages are incomprehensible, that subclassing it in a
second metaclass hierarchy is impossible, and that nobody can tell where a given attribute's value
comes from.

Write the assessment and the redesign. Your answer must: identify which mechanism is responsible for
each of the three reported symptoms, precisely; explain the metaclass composition failure in terms
of what Python actually requires; work down the mechanism ladder and state, for each feature, the
simplest mechanism that would deliver it; identify which features should not exist at all and why;
present the redesign with an explicit statement of what it gives up; and state the migration path,
given that the library has external users who cannot all be changed at once.

Marks are for the *subtraction*. A redesign that keeps every feature and merely reorganises it has
not engaged with the question.

**B5. (40)** You are designing the public API for a data-validation library. The core operation is:
given a schema and a value, either produce a validated typed object or a structured error. You want
it to feel declarative, to give good errors, to be fully typed, and to be extensible by users.

Design it. Your answer must: state which abstraction mechanisms you would use for the schema
definition and why — considering at minimum plain classes, dataclasses, descriptors, a metaclass,
`__init_subclass__`, and a builder-style DSL; explain how a user extends it with a custom validator,
and what that extension point costs you in future flexibility; explain how the types work — what a
type checker can infer about the output of a schema and what mechanism makes that possible; address
error reporting specifically, since it is the thing every such library gets wrong, and say what
structural decision makes good errors possible; state the three validation timings and which you
chose; and identify the abstraction in your design that you are least confident about, with the
signal that would tell you it was wrong.

Then apply the ten-uses test to your own design and report the result honestly.

---

## Marking guidance

Section A per question: 6 for the standard correct answer; 4 for precision; 3 for an example
not from the lessons; 2 for a stated limitation of your own answer or a connection to
PY-501/SE-511.

Section B: a correct, complete answer is 24/40. The rest is judgement, fairness to the
opposing case, and honesty about uncertainty.
