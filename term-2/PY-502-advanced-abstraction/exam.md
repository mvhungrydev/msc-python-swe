# PY-502 — Written Examination

**Time allowed: 3 hours. Closed book.**
**Answer FOUR from Section A and ONE from Section B.**
Section A: 15 marks each. Section B: 40 marks.

---

## Section A — answer FOUR

**A1.** Give the rule for choosing between duck typing, `typing.Protocol`, and an abstract
base class, with four criteria that select the ABC. Then explain why a Protocol lets the
consumer own the interface, and what that changes about the direction of a dependency.
Finally: name the thing about an interface that neither mechanism can express, and what
fills the gap. *(15)*

**A2.** Give the six-step decision ladder before writing a descriptor, and the question that
selects one. Then describe the four storage strategies for per-instance descriptor state,
with one fatal flaw of each, and say which is the default and why. *(15)*

**A3.** List the six steps of executing a `class` statement in order, including where
`__set_name__` and `__init_subclass__` fire. Then explain two consequences of that ordering
that would surprise someone who had not thought about it. *(15)*

**A4.** Name the four things only a metaclass can do. For each, give a real example from the
standard library or a well-known framework, and say what would be lost if you tried to do it
with `__init_subclass__` instead. Then explain why metaclasses do not compose and what that
costs a library's users. *(15)*

**A5.** Explain what `functools.wraps` copies and why `__wrapped__` matters. Then give three
pairs of decorators where stacking order changes the semantics, explaining each. Finally,
describe how you would enforce a required ordering at import time and what your enforcement
cannot detect. *(15)*

**A6.** Explain why `ParamSpec` exists, what the situation was before it, and what
`Concatenate` adds. Then write (in outline) the type signature of an
optionally-parameterized decorator, and say why overloads are required. *(15)*

**A7.** Enumerate the four cases determining when a `with` block *inside* a generator exits.
Then give three consequences for how generators that own resources should be written, and
state which of the four cases is not guaranteed on a non-refcounting implementation. *(15)*

**A8.** Give the four things `yield from` does that `for x in sub: yield x` does not. Then
relate `await` to `yield from` at the interpreter level, and explain
`asyncio.CancelledError` in terms of the generator protocol. *(15)*

**A9.** State the two rules of the context-manager protocol, and explain why one of them is
the most dangerous line in the protocol. Then explain what `ExitStack.pop_all()` is for,
with the transfer-of-ownership pattern, and say what problem it solves that nested `with`
cannot. *(15)*

**A10.** Name the four jobs a "model class" may be asked to do and explain why conflating
them is an architectural error. Then state the two-layer rule, the costs it imposes, and
three concrete evolutions that are cheap under it and expensive without it. *(15)*

**A11.** Give the three properties that justify building an internal DSL. Then name four
construction techniques with one hazard each, explain why `and`/`or` cannot be overloaded
and what follows, and identify the single investment that most distinguishes a good internal
DSL from a bad one. *(15)*

**A12.** Name the six costs of abstraction. Explain Ousterhout's deep/shallow distinction
with an example of each. Then explain why duplication is cheaper than the wrong abstraction,
being precise about the asymmetry in the costs. *(15)*

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

## Marking guidance

Section A per question: 6 for the standard correct answer; 4 for precision; 3 for an example
not from the lessons; 2 for a stated limitation of your own answer or a connection to
PY-501/SE-511.

Section B: a correct, complete answer is 24/40. The rest is judgement, fairness to the
opposing case, and honesty about uncertainty.
