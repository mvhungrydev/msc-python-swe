# SE-511 · Lesson 06 — Gradual Typing: Theory and Practice

**Estimated study time:** 4 hours
**Prerequisites:** PY-501 L02, L05

---

## 1. Orientation

```python
def f(x: int) -> str:
    return x          # mypy: error

def g(x: int) -> str:
    y: Any = x
    return y          # no error
```

The second function returns an `int` from a function annotated `-> str`, and the type
checker is *correct* to allow it. That is not a bug in mypy; it is the defining property of
a **gradual** type system, and understanding exactly why is the difference between using
type annotations as decoration and using them as a tool.

Python's type system is also **not enforced at runtime**. `f("hello")` runs fine. The
annotations are metadata; the guarantee is entirely at check time and entirely dependent on
having checked everything.

## 2. Theory

### 2.1 What gradual typing is

Siek & Taha (2006) formalized the idea: a type system with a distinguished type — written
`?`, and spelled `Any` in Python — that is **compatible with every type in both
directions**.

The key relation is **consistency** (`~`), which replaces subtyping at the boundary:

- `int ~ int`
- `Any ~ int` and `int ~ Any`
- `list[int] ~ list[Any]` and `list[Any] ~ list[int]`

Consistency is **reflexive and symmetric but NOT transitive**. `int ~ Any` and
`Any ~ str`, but `int ~ str` is false. This non-transitivity is what makes gradual typing
work at all: it lets typed and untyped regions interoperate without the untyped region
poisoning everything.

`Any` is therefore not "the top type". `object` is the top type: every value is an
`object`, and you can do almost nothing with an `object` without narrowing it. `Any` is
*unknown*: you can do anything with it, and the checker will not complain. They are opposites
in the only way that matters.

**Practical rule:** when you want "any value", write `object` and narrow. Write `Any` only
when you mean "I am switching the checker off here", and treat each occurrence as a debt
with a comment.

### 2.2 Soundness, and Python's deliberate unsoundness

A type system is **sound** if a well-typed program cannot go wrong at runtime in the ways
the types rule out. Python's is not sound, on purpose, in at least these ways:

- **`Any` propagation.** §2.1's example.
- **No runtime enforcement.** Nothing checks arguments at call time.
- **Unchecked boundaries.** `json.loads` returns `Any`. Every piece of data entering your
  program from JSON, YAML, a database driver, or an untyped library arrives as `Any` and
  flows anywhere.
- **`cast()`.** An unchecked assertion by the programmer.
- **Deliberate unsoundness for usability.** `mypy` treats a `list[int]` parameter as
  invariant (sound) but permits some unsound conveniences, e.g. the default handling of
  `**kwargs` and certain descriptor cases; checkers differ on how much of this to allow.

The consequence to internalize: **a green type check is evidence, not proof.** Its strength
is proportional to the fraction of your code and dependencies that are actually typed. In a
codebase where 40% of expressions are `Any`, a green check means very little — which is why
`--disallow-any-expr` and mypy's `--any-exprs-report` exist, and why measuring your `Any`
surface is more informative than measuring "percent annotated".

### 2.3 What annotations *do* buy

Given they prove nothing at runtime, why bother? Four concrete returns, ranked by what the
evidence and experience support:

1. **Refactoring.** Renaming a field, changing a signature, or splitting a class becomes a
   mechanical, checkable operation instead of a search-and-hope. This is the largest single
   benefit and grows superlinearly with codebase size.
2. **Documentation that cannot rot.** A signature is checked; a docstring is not.
3. **Editor tooling.** Completion, go-to-definition, and inline errors. This changes the
   experience of working in unfamiliar code more than anything else on this list.
4. **A specific class of defect.** `None` handling above all — `Optional` with strict
   checking eliminates the single most common Python runtime error. Also: wrong argument
   order for same-arity calls, misspelled attributes, unhandled union members.

What they do **not** buy: correctness of logic, protection at trust boundaries, or
performance (CPython ignores them entirely; `typing` is not `mypyc`).

### 2.4 The type system's building blocks

```python
from collections.abc import Sequence, Mapping, Callable, Iterator
from typing import Any, TypeAlias, Literal, Final, TypedDict, NewType, cast

Age = NewType("Age", int)              # distinct at check time, int at runtime
Handler: TypeAlias = Callable[[str], None]

MAX: Final = 100                       # cannot be reassigned; also narrows to Literal[100]

Mode = Literal["r", "w", "a"]          # a finite set of values

class User(TypedDict):                 # a dict with a known key/value shape
    id: int
    name: str
    email: str | None
```

Notes that matter:

- **`X | None`, not `Optional[X]`.** Since PEP 604 the pipe syntax works at runtime on 3.10+
  and is shorter. `Optional[X]` means exactly `X | None` and misleads people into thinking
  it means "optional argument".
- **Use `collections.abc` for parameters, concrete types for returns.** Accept
  `Sequence[int]`; return `list[int]`. The general principle (Postel-ish): be liberal in
  what you accept, precise in what you promise. `Iterable` as a *parameter* is a claim you
  will iterate once — see L06's iterator/iterable distinction in PY-501.
- **`NewType` is free at runtime** and gives you a distinct type. `Age(30)` is just `30`, but
  a function taking `Age` will reject a bare `int`. Excellent for `UserId` vs `OrderId` —
  the two-int-arguments-swapped bug is real and this eliminates it.
- **`TypedDict`** for JSON-shaped data you cannot change. If you *can* change it, use a
  dataclass or `pydantic` model instead; `TypedDict` has no runtime validation and no
  methods.
- **`Final`** for constants — and note it doubles as a `Literal` narrowing, which enables
  exhaustiveness checks.

### 2.5 Narrowing

The checker tracks types through control flow:

```python
def f(x: int | str | None) -> str:
    if x is None:
        return ""
    if isinstance(x, int):
        return str(x)      # x: int here
    return x.upper()       # x: str here — narrowed by elimination
```

Narrowing constructs the checker understands: `isinstance`, `issubclass`, `is None`/
`is not None`, `==`/`!=` against `Literal`s, truthiness (partially), `assert`,
`type(x) is C`, `in` against a literal tuple, and `match` statements.

**Exhaustiveness checking** is the highest-value trick in the whole type system:

```python
from typing import assert_never

def area(shape: Circle | Square | Triangle) -> float:
    match shape:
        case Circle(r):   return math.pi * r * r
        case Square(s):   return s * s
        case Triangle(b, h): return b * h / 2
        case _:
            assert_never(shape)     # error if any union member is unhandled
```

Add a fourth shape and every `assert_never` in the codebase becomes a type error pointing
at the place that must be updated. This converts "find all the places that switch on shape"
from a grep into a compiler task, and it is the single strongest argument for typed unions
over class hierarchies with virtual methods when the *set of operations* changes more often
than the set of types. (The reverse — types change more often than operations — favours
polymorphism. This is the *expression problem*, and CS-641 L06 treats it properly.)

For custom predicates, `TypeGuard` (PEP 647) and `TypeIs` (PEP 742) let you teach the
checker:

```python
from typing import TypeIs

def is_str_list(xs: list[object]) -> TypeIs[list[str]]:
    return all(isinstance(x, str) for x in xs)
```

`TypeIs` narrows in both branches (the `else` gets the negation); `TypeGuard` narrows only
the positive branch. Prefer `TypeIs` where applicable — it is what you almost always mean.

### 2.6 Strictness, incrementally

`mypy --strict` is a bundle. The individually important flags:

| Flag | What it catches |
|---|---|
| `disallow_untyped_defs` | functions with no annotations (which are silently skipped otherwise) |
| `disallow_incomplete_defs` | half-annotated functions |
| `check_untyped_defs` | analyses bodies of unannotated functions |
| `no_implicit_optional` | `def f(x: int = None)` — now an error, as it should be |
| `warn_return_any` | returning `Any` from a function declared to return something specific |
| `disallow_any_generics` | bare `list`, `dict` instead of `list[int]` |
| `warn_unused_ignores` | `# type: ignore` comments that no longer suppress anything |
| `strict_equality` | `x == y` where the types cannot overlap — finds real bugs |
| `warn_unreachable` | code the checker proves cannot run — finds real bugs |

**The migration strategy for an existing codebase**, which is the situation you will
actually be in:

1. Turn on mypy with almost everything off. Get to green. Commit.
2. Enable `warn_unused_ignores` and `strict_equality` — cheap, immediate value.
3. Add strict settings **per module**, starting at the leaves (no internal imports) and
   working inward:

```toml
[[tool.mypy.overrides]]
module = ["mypkg.domain.*", "mypkg.util.*"]
disallow_untyped_defs = true
disallow_any_generics = true
```

4. Ratchet: CI fails if the number of `type: ignore`s or non-strict modules **increases**.
   A ratchet is the only migration mechanism that survives contact with a busy team.

Never do a big-bang annotation of an entire codebase. Annotations written without running
the checker are wrong at a rate of roughly one in five, and wrong annotations are worse
than none — they are confidently misleading.

### 2.7 Runtime validation is a separate problem

The most common serious misunderstanding: *annotations do not validate input.* At every
trust boundary — HTTP request bodies, config files, message payloads, database rows,
CLI arguments — you need actual runtime validation.

```python
from pydantic import BaseModel

class CreateOrder(BaseModel):
    customer_id: int
    items: list[Item]
    coupon: str | None = None

order = CreateOrder.model_validate(request.json())   # raises on bad input
```

The pattern: **validate once at the boundary, then trust the types inside.** The validated
model is where `Any` stops. This is the *anti-corruption layer* of SE-521 L06, and typing
gives you a mechanical way to see whether you have one: if `Any` appears deep in your
domain, data is entering unvalidated.

Alternatives: `pydantic` (fast, ubiquitous, opinionated, brings a dependency and a
metaclass), `attrs` + `cattrs` (lighter, more composable), `msgspec` (fastest, less
featureful), hand-written parsers (always an option, and correct for small surfaces).
Choose deliberately; the choice affects your error messages, which affects your API's
usability more than most people expect.

## 3. Construction: typing a real module

Take a 200–400-line module of your own with no annotations.

**Step 1 — turn the checker on with nothing strict.** Fix whatever it finds. There will be
some; this alone finds bugs.

**Step 2 — annotate signatures only, leaves first.** Not bodies, not local variables — the
checker infers those. Run mypy after *each function*, not at the end. The whole value is in
the feedback loop.

**Step 3 — notice where you cannot express the type.** This is the interesting part. Every
place where the type is hard to write is a place where the design is loose:

- A function returning `dict | list | None` depending on a flag argument → it is three
  functions, or it needs `@overload`.
- A parameter typed `Any` because it might be a path, a string, or a file object → define
  the union explicitly, or a `Protocol`, and discover that one of the three was never
  actually supported.
- A `**kwargs: Any` passed through four layers → use `TypedDict` with
  `Unpack` (PEP 692), or a dataclass, and find the two keys nobody handles.

Record each. This list is the module's design review, produced for free.

**Step 4 — enable strict, per-module.** Then handle the fallout, distinguishing three cases:

- **Real bug** → fix it. (You will find some. `None` handling is the usual one.)
- **The checker is wrong / the code is dynamic** → `# type: ignore[specific-code]` with a
  comment explaining why. Never bare `# type: ignore`; always the error code.
- **The design is wrong** → fix the design. This is the most valuable outcome and the one
  people skip.

**Step 5 — measure your `Any` surface.**

```bash
mypy --any-exprs-report reports/ mypkg
```

Report the percentage of expressions with `Any` type. This number, not "percent of
functions annotated", is the honest measure of how much your green check is worth.

## 4. Failure modes

- **Treating annotations as validation.** §2.7. The most consequential error on this list.
- **`Any` where `object` was meant.** §2.1.
- **Bare `# type: ignore`.** Suppresses future, unrelated errors on that line forever.
- **Annotating without running the checker.** Produces confidently wrong documentation.
- **`--strict` on an untyped codebase.** 4,000 errors, everyone gives up, the tool is
  uninstalled.
- **Using `cast` to silence an error you did not understand.** `cast` is an assertion you
  are making; if it is wrong, the checker will now *propagate* your error.
- **Trusting third-party stubs blindly.** Stubs from `typeshed` are community-maintained and
  occasionally wrong. When behaviour disagrees with the stub, the stub is wrong.
- **Ignoring `Any` from untyped dependencies.** `disallow_untyped_calls` and
  `disallow_any_unimported` are how you find them.
- **`isinstance` on a `Protocol` without `runtime_checkable`** — and even then, it only
  checks method *presence*, not signatures.

## 5. Exercises

### Warm-up (25 min)

**W1.** Write a program that type-checks cleanly and crashes at runtime, using only `Any`
propagation (no `cast`, no `ignore`). Then do the same using `cast`.

**W2.** Show that consistency is not transitive, with a concrete three-type example and the
mypy output.

**W3.** Write an exhaustiveness check with `assert_never`, then add a union member and show
the error.

### Core (2.5 h)

**C1 — Type a module.** Complete §3 on a real module. Deliverable: the diff, the list from
step 3 (places where the type was hard to write and what that revealed), the bugs found in
step 4, and the `Any`-surface percentage before and after.

**C2 — Boundary validation.** Take an API endpoint or message consumer with no validation.
Add it with `pydantic` (or `attrs`+`cattrs`). Then write tests for ten malformed inputs and
check the error messages a client would receive. Rewrite any message that would not help a
caller fix their request. Report the before/after.

**C3 — `NewType` for identifiers.** Find a codebase with several bare-`int` or bare-`str`
identifiers (`user_id`, `order_id`, `sku`). Introduce `NewType` for each. Report every
error the checker finds — there will be at least one place where the wrong id was being
passed, and it will have been there for a long time.

**C4 — Strictness ratchet.** Implement a CI check that fails when the count of
`# type: ignore` comments or the number of non-strict modules increases. Include a report
that shows the trend. Then write 300 words on why a ratchet works where a target does not.

### Challenge

**X1.** Read Siek & Taha, "Gradual Typing for Functional Languages" (2006) and PEP 483
("The Theory of Type Hints"). Write 1,000 words explaining consistency versus subtyping,
why the gradual guarantee matters, and where Python's implementation departs from the
theory — naming at least three specific departures and the pragmatic reason for each.

**X2.** Take a module with 100% `--strict` compliance and find three runtime errors it
permits. Categorize each by which unsoundness (§2.2) enabled it. Then propose, for each, a
technique that *would* catch it (runtime validation, property test, assertion, formal
method) and estimate the cost.

## 6. Self-check

1. Define consistency and explain why it is not transitive.
2. Distinguish `Any` from `object`, and say when to use each.
3. Give four sources of unsoundness in Python's type system.
4. What do annotations buy, and what do they not?
5. Why `Sequence` for parameters and `list` for returns?
6. What does `assert_never` do and what problem does it solve?
7. Give the migration strategy for an untyped codebase, and say why big-bang fails.
8. Where must runtime validation live, and what does it have to do with `Any` appearing
   deep in a codebase?

## 7. Primary sources

- PEP 483 ("The Theory of Type Hints") and PEP 484. Read 483 first.
- Siek & Taha, "Gradual Typing for Functional Languages" (Scheme Workshop, 2006).
- PEP 604 (union syntax), 647 (`TypeGuard`), 742 (`TypeIs`), 692 (`Unpack` for kwargs),
  695 (type parameter syntax).
- mypy documentation: "Common issues", "Dynamic typing", and the configuration reference.

---

**Previous:** [L05](L05-test-architecture.md) · **Next:**
[L07 — Generics, Variance, and Protocols](L07-generics-variance-protocols.md)
