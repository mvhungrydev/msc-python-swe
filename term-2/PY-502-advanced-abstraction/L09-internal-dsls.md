# PY-502 · Lesson 09 — Internal DSLs and Fluent Interfaces

**Estimated study time:** 3.5 hours
**Prerequisites:** L01–L04, L08

---

## 1. Orientation

```python
q = (select(User)
     .where(User.age > 21, User.country.in_(["US", "CA"]))
     .order_by(User.created_at.desc())
     .limit(100))
```

That reads like SQL and is Python. `User.age > 21` does not evaluate to `True` or `False`;
it builds an expression tree, using the operator dispatch of PY-501 L06.

Internal DSLs — domain-specific languages hosted inside the host language's syntax — are the
most visible payoff of Python's metaprogramming facilities. They are also the fastest way to
build something only their author can maintain. This lesson teaches the techniques and, more
importantly, the criteria.

## 2. Theory

### 2.1 What a DSL buys

A DSL is worth building when **the notation itself carries meaning that ordinary code
obscures.** Three properties to look for:

1. **A real domain vocabulary** exists, and domain experts use it. Query languages, build
   specifications, test scenarios, schedules, validation rules, state machines, permission
   grammars.
2. **The structure is the value.** The point of the query above is that its *shape* mirrors
   the query's shape. A list of `add_filter(...)` calls encodes the same thing and reads
   worse.
3. **It is written far more often than the machinery is changed.** The cost of building and
   learning the DSL amortizes over many uses.

Fail any of these and you have an abstraction that costs more than it saves. Fowler's
distinction is useful: a DSL has *limited expressiveness* (it cannot express everything) and
a *language nature* (it composes). Something that is merely an API with method chaining is
not a DSL, and calling it one does not make it good.

### 2.2 The techniques

**Method chaining (fluent interface).** Each method returns something chainable.

```python
class Query:
    def where(self, *conds: Cond) -> "Query":
        return replace(self, conds=(*self.conds, *conds))     # IMMUTABLE — returns new
```

The critical decision is **immutable or mutable**. Returning `self` after mutation is
easier and produces aliasing bugs: `base = query.where(a)` then `q1 = base.where(b)` and
`q2 = base.where(c)` gives `q1 is q2 is base` with all three conditions. Return a new object.
`dataclasses.replace` / `attrs.evolve` makes this one line.

Fluent interfaces have a real cost: they are awkward to debug (one expression, one line in
the traceback), awkward to build conditionally (`if x: q = q.where(...)` breaks the chain),
and they encourage very long expressions.

**Operator overloading.** `__gt__`, `__and__`, `__or__`, `__invert__`, `__getitem__`,
`__call__` building an expression tree rather than computing.

```python
class Column:
    def __gt__(self, other: object) -> "Cond": return Cond(self, ">", other)
    def __and__(self, other: "Cond") -> "Cond": return And(self, other)
```

Powerful and dangerous. Specific hazards:

- **`and`/`or`/`not` cannot be overloaded.** They short-circuit on truthiness and there is
  no protocol. Every Python DSL therefore uses `&`, `|`, `~` — and those have *higher
  precedence than comparison*, so `a == 1 & b == 2` parses as `a == (1 & b) == 2`.
  Users must write `(a == 1) & (b == 2)`, and they will forget. Some libraries make
  `__bool__` raise to catch it, which is a good idea.
- **`__eq__` returning a non-bool** breaks `==`, `in`, `dict` keys, `assert x == y`, and
  every tool that compares objects. SQLAlchemy accepts this cost knowingly; you should
  decide consciously.
- **Type checkers do not understand it.** `User.age > 21` has type `bool` as far as the
  checker is concerned unless you annotate `__gt__ -> Cond`, and then `if x > y:` in
  ordinary code on that type stops type-checking sensibly.

**Context managers for scoping.**

```python
with pipeline("etl") as p:
    with p.stage("extract"):
        p.task(fetch, retries=3)
    with p.stage("load"):
        p.task(write)
```

Uses nesting to express structure. Clean, but it relies on implicit "current scope" state
(usually a `ContextVar` or a stack), which is invisible and makes the functions non-pure.

**Decorators for declaration.**

```python
@app.route("/orders/{id}", methods=["GET"])
@requires("orders:read")
def get_order(id: OrderId) -> Response: ...
```

The dominant Python DSL form (Flask, FastAPI, `click`, `pytest`). Good: the declaration sits
with the thing declared. Bad: registration depends on import side effects (L03 §3).

**Class bodies as declarations.**

```python
class UserTable(Table):
    id = Integer(primary_key=True)
    email = String(unique=True)
```

Uses descriptors + `__set_name__` + `__init_subclass__` (L02, L03). Reads well, is
introspectable, and is what every ORM does.

**Callable objects and partial application** for building small combinator languages —
parser combinators, validation chains, retry policies. Often the cleanest option and the
most overlooked:

```python
policy = retry_on(TimeoutError) & max_attempts(5) & backoff(exponential(0.1))
```

### 2.3 The costs, stated honestly

A DSL is a language, and every language has these costs:

- **It must be learned.** A new team member cannot read your DSL by knowing Python. Every
  hour saved writing is paid back in onboarding.
- **Tooling degrades.** `go to definition` on `User.age` lands on a descriptor, not a
  column. Autocomplete may or may not work. The debugger shows expression-tree
  construction, not your intent. `grep` for a field name may find nothing.
- **Error messages are terrible by default.** A user error in a DSL surfaces as an exception
  inside your machinery, with a traceback through your internals. Making DSL errors point at
  the user's line requires deliberate work — capturing source locations at construction, and
  wrapping internal errors.
- **Type checking is limited.** §2.2. This is a bigger cost in 2026 than it was in 2016.
- **Debuggability.** You cannot set a breakpoint "inside" a declarative rule.
- **Escape hatches are required.** Every DSL eventually meets a case it cannot express.
  If there is no way to drop to ordinary code, users will fight it or abandon it. Design
  the escape hatch on day one.

### 2.4 Error messages: the differentiator

This is what separates a good internal DSL from a bad one, and it is almost always skipped.

Compare:

```
TypeError: unsupported operand type(s) for &: 'bool' and 'Column'
```

with

```
QueryError: in the filter at orders.py:42
    .where(User.age > 21 & User.active)
                     ^^^^^^^^^^^^^^^^^
  `&` binds tighter than `>`. Write (User.age > 21) & User.active
```

The second requires: raising your own exception type from `__and__` when it receives a
`bool`; capturing the caller's frame (`sys._getframe`, `traceback.extract_stack`) at
construction; and formatting the source line. Perhaps forty lines of work, and it converts
your DSL from hostile to teachable.

**Rule: for every way a user can misuse your DSL, produce an error naming their line and
saying what to write instead.** If you cannot afford that for all of them, you cannot afford
the DSL.

### 2.5 Validation timing

Three points at which a DSL can reject something, in order of preference:

1. **Static (type checker).** Best, and mostly unavailable for operator-overloading DSLs.
   `@overload`, `Literal`, and `TypeGuard` can get you some of it.
2. **Construction time.** When `select(...).where(...)` is *built* — usually at import for
   declarative forms. Good: errors appear at startup, not at request time.
3. **Execution time.** When the query runs. Worst, but sometimes unavoidable (a column that
   does not exist in the target table).

Push validation as early as you can. A DSL whose errors only appear when a rarely-taken
branch executes is worse than the code it replaced.

### 2.6 The escape hatch

Every DSL needs a documented way out:

```python
q = select(User).where(text("age > 21 AND weird_pg_function(x)"))   # raw SQL
```

Two design points:

- **Make it visible.** `text(...)`, `raw(...)`, `unsafe_(...)` — a name that shows up in
  review. Do not let escaping be invisible.
- **Do not make it a trap.** The most famous escape hatch, string interpolation into SQL, is
  an injection vulnerability. Provide parameters even in the escape hatch.

A DSL with no escape hatch forces users to abandon it entirely for the 5% case, which means
maintaining two mechanisms.

### 2.7 When not to build one

Reach for the alternatives first:

- **A configuration format** (TOML/YAML) plus a validated model (L08). If the DSL is
  declarative data with no logic, it does not need to be code. It also becomes editable by
  non-programmers and diffable.
- **Plain functions with keyword arguments.** Often 90% as readable with none of the cost.
- **An existing DSL.** SQL, Jinja, CEL, JSONLogic, `jq`. Using one is nearly always better
  than inventing one.
- **A builder object with ordinary methods.** No operators, no magic, full type checking.

The test: *write ten realistic uses of the proposed DSL, and the same ten in plain Python.*
If the plain version is not clearly worse, stop. This exercise takes an hour and has
prevented many bad libraries.

## 3. Construction: a validation-rule DSL

Build one, then critique it.

**Step 1 — the ten uses, both ways.** Write ten realistic validation rules as the DSL you
intend, and as plain functions. For example:

```python
# DSL
rules = [
    field("email").required().matches(EMAIL).max_len(254),
    field("age").optional().integer().between(0, 150),
    when(field("country") == "US", then=field("state").required()),
]

# plain
def validate(d: dict) -> list[Error]:
    errs = []
    if not d.get("email"): errs.append(Error("email", "required"))
    elif not EMAIL.match(d["email"]): errs.append(Error("email", "invalid"))
    ...
```

Judge honestly. For ten rules the plain version is competitive. For two hundred rules,
edited by non-authors, with the rule set needing to be introspected to generate a form and
documentation — the DSL wins. **State which situation you are in.** That is the assessed
part.

**Step 2 — the expression tree.** Rules build objects; nothing evaluates during
construction:

```python
@dataclass(frozen=True)
class Rule:
    field: str
    checks: tuple[Check, ...]
    def __call__(self, data: Mapping[str, object]) -> Iterator[Error]: ...
```

Immutable chaining via `replace`. Note that making `Rule` callable means it composes with
ordinary Python — a small but important escape hatch: any callable
`(Mapping) -> Iterator[Error]` is a valid rule.

**Step 3 — source locations.** At construction, capture the caller's frame:

```python
@dataclass(frozen=True)
class Origin:
    file: str
    line: int
    text: str

def _origin(depth: int = 2) -> Origin:
    f = sys._getframe(depth)
    return Origin(f.f_code.co_filename, f.f_lineno,
                  linecache.getline(f.f_code.co_filename, f.f_lineno).strip())
```

Attach it to every node. Now every error message can point at the rule's definition site,
which is the difference between a usable DSL and an unusable one.

**Step 4 — errors for misuse.** Enumerate the ways a user can get it wrong: chaining
`required()` after `optional()`; a `between` with reversed bounds; referring to a field that
does not exist in the schema; using `and` instead of `&`. For each, raise at *construction*
with the origin and a suggested fix. Make `__bool__` raise on your condition type so that
`if field("x") == "y":` fails loudly.

**Step 5 — introspection and the escape hatch.** Add `Rule.describe()` producing
human-readable documentation and `Rule.to_json_schema()`. Add `custom(fn)` for arbitrary
predicates, with the origin captured so its errors are attributable too.

**Step 6 — the critique.** Write it. What does a new team member need to learn? What does
the type checker know? What happens in the debugger? How does `grep "email"` behave? What is
the import-time cost? Would you ship this? A well-argued "no" is a full-credit answer, and
the more common correct one.

## 4. Failure modes

- **A DSL for something used five times.** The machinery outweighs the uses.
- **Mutable fluent interfaces.** Aliasing bugs.
- **`&`/`|` precedence.** Users write `a == 1 & b == 2` and get nonsense. Make `__bool__`
  raise.
- **`__eq__` returning non-bool** without accounting for everything it breaks.
- **No source locations.** Errors point at your internals.
- **Validation only at execution.** Errors in production, not at startup.
- **No escape hatch**, or an escape hatch that is an injection vector.
- **Invisible state** (a "current scope" `ContextVar`) that breaks under concurrency or
  nesting.
- **Reimplementing SQL/regex/CEL badly.** Use the existing one.
- **A DSL that could be a TOML file.**
- **Type checking abandoned silently.** Users lose it and do not know why.

## 5. Exercises

### Warm-up (25 min)

**W1.** Demonstrate the `&`-precedence trap. Then implement a `__bool__` that raises with a
message telling the user exactly what to write.

**W2.** Build a mutable fluent interface and demonstrate the aliasing bug. Then fix it with
`replace`.

**W3.** Capture a caller's source line with `sys._getframe` + `linecache` and include it in
an exception.

### Core (2.5 h)

**C1 — The validation DSL.** Complete §3, all six steps. Deliverable: the ten-uses
comparison, the implementation, the misuse errors with source locations, the introspection,
and the written critique with a ship/don't-ship recommendation.

**C2 — Dissect a real DSL.** Choose SQLAlchemy Core's expression language, `polars`
expressions, `click`, or `pytest`'s fixtures/marks. Identify: every technique from §2.2 it
uses, what its escape hatch is, how good its errors are (test five deliberate misuses), and
what the type checker knows. Write 800 words. Then name one thing you would change.

**C3 — DSL versus config.** Take a DSL (yours or a real one) whose content is essentially
declarative. Express the same content as TOML plus a `pydantic` model. Compare: expressive
power lost, tooling gained, who can now edit it, diff quality, and what happens when a rule
needs a computation. Recommend one with reasons.

**C4 — Error message rewrite.** Take a library whose DSL errors are bad. For five realistic
mistakes, write the error message you wish it produced. Then implement the improvement for
at least two, as a patch or a wrapper. Measure the work.

### Challenge

**X1.** Build a small typed query DSL where the type checker *does* follow the types:
`select(User).where(...)` returns a `Query[User]`, `.select(User.name)` narrows to
`Query[str]`, and a column comparison against the wrong type is a static error. Use generics,
`Literal`, and overloads. Report exactly where the type system runs out, and what a language
with higher-kinded types would let you express (return to this after CS-641 L05).

**X2.** Read Fowler's *Domain-Specific Languages*, Part I. Write 1,200 words applying his
internal/external distinction and his "language workbench" argument to a DSL in your
ecosystem, and take a position on whether Python's metaprogramming makes internal DSLs too
easy to build — that is, whether the low cost of construction leads to systematically bad
cost/benefit decisions.

## 6. Self-check

1. Give the three properties that justify a DSL.
2. Name six DSL construction techniques and one hazard of each.
3. Why can `and`/`or` not be overloaded, and what follows?
4. What breaks when `__eq__` returns a non-bool?
5. Give the three validation timings in order of preference and why.
6. What does an escape hatch need, and what is the classic escape-hatch vulnerability?
7. What is the ten-uses test?
8. What single investment most distinguishes a good internal DSL from a bad one?

## 7. Primary sources

- Fowler, *Domain-Specific Languages* (2010), Part I.
- SQLAlchemy Core documentation and source — the most sophisticated internal DSL in Python.
- `click` and `pytest` sources for the decorator-declaration form.
- Hudak, "Building Domain-Specific Embedded Languages" (Computing Surveys, 1996) — the
  original statement of the embedded-DSL idea.

---

**Previous:** [L08](L08-declarative-models.md) · **Next:**
[L10 — The Cost of Abstraction](L10-cost-of-abstraction.md)
