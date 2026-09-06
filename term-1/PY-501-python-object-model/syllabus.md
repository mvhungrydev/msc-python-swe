# PY-501 — The Python Object Model & Execution Semantics

**Term:** 1 · **Credits:** 15 · **Nominal hours:** 125
**Prerequisites:** professional Python experience; no formal CS required
**Co-requisite:** SE-511

---

## Driving question

> What actually happens when Python evaluates `a.b(c)`?

You can write Python for a decade without being able to answer this. The gap shows up as a
class of bugs you can only fix by trial and error — a descriptor that fires at the wrong
time, a metaclass conflict, a memory profile that will not come down, a subclass whose
`super()` call reaches somewhere you did not expect. This course closes the gap.

## Learning outcomes

On completion you will be able to:

1. **Explain** Python's object model precisely: the relationship between objects, types,
   and metatypes, and the invariants each maintains.
2. **Trace** attribute access through `__getattribute__`, the MRO, the descriptor
   protocol, and `__getattr__`, and predict the result for arbitrary class hierarchies.
3. **Derive** the C3 linearization of a hierarchy by hand and identify hierarchies for
   which no linearization exists.
4. **Implement** types that behave correctly under equality, hashing, ordering, copying,
   pickling, and inheritance — and argue why each choice was made.
5. **Read** CPython bytecode fluently and use it to answer questions about semantics that
   the documentation does not settle.
6. **Reason** about object lifetime: reference counting, cycles, the generational
   collector, weak references, and finalization order.
7. **Analyse** the scoping and closure rules well enough to explain any binding-related
   surprise in unfamiliar code.
8. **Critique** an implementation on the basis of the object model — e.g. explain why a
   given caching decorator breaks on methods, or why a `__slots__` class still grows.

## Lessons

| # | Title | Hours |
|---|---|---|
| L01 | Names, Objects, and Binding | 3 |
| L02 | Types, Instances, and Metatypes | 3.5 |
| L03 | Attribute Lookup and the Descriptor Protocol | 4 |
| L04 | Inheritance, `super()`, and the MRO | 4 |
| L05 | Identity, Equality, Hashing, and Ordering | 3.5 |
| L06 | The Data Model: Protocols and Operator Dispatch | 4 |
| L07 | Scopes, Frames, and Closures | 3.5 |
| L08 | Bytecode and the Evaluation Loop | 4 |
| L09 | Memory: Reference Counting, Cycles, and Finalization | 4 |
| L10 | Exceptions, Control Flow, and Cleanup Semantics | 3.5 |

Plus problem sets (~50h), primary reading (~15h), exam preparation and exam (~15h).

## Assessment

| Component | Weight |
|---|---|
| Problem set 1 (L01–L04): a working attribute-access tracer | 20% |
| Problem set 2 (L05–L06): a numeric type that is correct under the full data model | 20% |
| Problem set 3 (L07–L10): a bytecode-level analysis tool | 20% |
| Written exam | 25% |
| Course position paper (1,500 words) | 15% |

## Required reading

- Ramalho, *Fluent Python*, 2nd ed., chapters 1, 6, 11–14, 16, 22–24.
- Shaw, *CPython Internals*, chapters on objects, the evaluation loop, and memory.
- The Python Language Reference, sections 3 (Data model) and 4 (Execution model). Read
  these twice, in full. They are the actual specification and almost nobody reads them.
- PEPs 8, 20, 3119, 3141, 557, 634.

## Recommended

- CPython source: `Objects/object.c`, `Objects/typeobject.c` (especially
  `type_new` and `_PyObject_GenericGetAttrWithDict`), `Python/ceval.c`.
- Hettinger, "Python's Class Development Toolkit" and "Super Considered Super!" (PyCon
  talks). The second is the clearest available explanation of cooperative multiple
  inheritance.

## A note on version sensitivity

This course touches implementation detail. Everything is written against **CPython 3.12+**
and marked where behaviour differs. Bytecode in particular changes between minor versions:
when a lesson shows a disassembly, *run it yourself* and expect differences. Noticing and
explaining the difference is part of the exercise.
