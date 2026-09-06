# PY-501 — Problem Sets

Three sets, each assessed against the five-criterion rubric in
`00-program/assessment-and-rubrics.md`. Each is a real deliverable: code plus a written
design note. The written note is not optional and is where most of the marks live.

---

## Problem Set 1 — The Attribute Access Tracer
**Covers L01–L04 · Budget: 8–12 hours**

### Deliverable

A package `pytrace` providing:

```python
trace_lookup(obj, name) -> LookupReport
trace_mro(cls) -> MroReport
explain(obj, name) -> str        # human-readable narrative
```

`LookupReport` must record, for a single attribute access:

- Which class in `type(obj).__mro__` provided the attribute (or that none did).
- Whether the provider was a data descriptor, a non-data descriptor, or a plain value.
- Whether the instance `__dict__` was consulted, and whether it won.
- Whether `__getattr__` fired.
- Whether `__getattribute__` was overridden anywhere in the MRO (and if so, that your
  report describes the *default* algorithm and may not reflect reality — say so explicitly
  rather than silently lying).

`MroReport` must render the C3 linearization and, for a given method name, the full
cooperative chain: every class defining it, in MRO order, with a flag for whether each
appears to call `super()`.

### Test corpus (all must be covered)

1. A plain class attribute.
2. An instance attribute shadowing a class attribute.
3. A `property`.
4. A `property` shadowed by an instance-dict entry (show that the property wins).
5. A method (demonstrate the non-data descriptor path and the fresh bound method).
6. `staticmethod` and `classmethod`.
7. `functools.cached_property`, before and after first access.
8. A `__slots__` class.
9. `__getattr__` fallback.
10. A custom `__getattribute__`.
11. A diamond hierarchy with a cooperative `__init__`.
12. A hierarchy with no valid MRO (your tool must report the failure usefully).

### Design note (800–1,200 words)

Answer:

- Where does your tool's model of attribute lookup diverge from what CPython actually does,
  and how did you establish that?
- What can your tool *not* determine without executing the access, and why?
- You had to choose between reimplementing the algorithm and instrumenting the real one.
  Which did you choose and what did it cost?
- Give one bug from your own experience (or a plausible one) that this tool would have
  diagnosed in minutes rather than hours.

### Marking emphasis

Justification and Precision of language. A tool that works but whose note is vague scores
below one that is slightly incomplete but whose note is exact.

---

## Problem Set 2 — A Numeric Type Under the Full Data Model
**Covers L05–L06 · Budget: 10–14 hours**

### Deliverable

A `Quantity` type: a number with a physical unit. Requirements:

**Value semantics.** Immutable, hashable, correct `__eq__` (including `NotImplemented` for
foreign types), correct `__hash__` consistent with equality, `__repr__` that round-trips
through `eval`, and `__format__` supporting standard numeric format specs plus at least one
custom spec of your design.

**Arithmetic.** `+` and `-` between compatible units (with conversion), `*` and `/`
producing derived units, `**` with integer exponents, unary `-`, `abs()`. Correct
`NotImplemented` returns so that `2 * q` works via `__rmul__` and so that a future type can
interoperate. `sum()` must work.

**Ordering.** Decide whether `Quantity` is totally or partially ordered, implement
accordingly, and defend the choice in the note. If you implement `__lt__`, state what
`sorted()` does with incompatible units and why that is acceptable.

**Containers.** Usable as a dict key and set member. Demonstrate that
`Quantity(1, "km") == Quantity(1000, "m")` and that their hashes agree — or argue that
they should *not* be equal, and defend that instead. (Both positions are defensible. The
mark is for the argument, not the position.)

**Numeric tower.** Decide whether to register with `numbers.Real`. Determine empirically
what breaks if you do and what breaks if you do not.

**Serialization.** `copy`, `deepcopy`, and `pickle` round-trip with invariants re-checked.

### Tests

A property-based suite (you may do this after SE-511 L04, or use `hypothesis` now with the
docs) asserting at minimum:

- `a + b == b + a`, `(a + b) + c == a + (b + c)` within a tolerance you justify.
- `a == b` implies `hash(a) == hash(b)`, over generated pairs.
- Unit algebra: `(a * b) / b == a` for compatible dimensions.
- Round-trip: `eval(repr(q)) == q`; `pickle.loads(pickle.dumps(q)) == q`.

### Design note (1,000–1,500 words)

- Your representation choice (float vs Decimal vs Fraction vs integer base units) and its
  consequences for equality, hashing, and the associativity property above. Be specific
  about what your tolerance-based property test does *not* prove.
- Your equality-across-units decision.
- Three protocols you deliberately did not implement, and why.
- A comparison with `pint` or `astropy.units`: name two things they do that you did not,
  and say whether they were right to.

---

## Problem Set 3 — Semantics at the Bytecode Level
**Covers L07–L10 · Budget: 10–14 hours**

### Part A — The evaluation-order reference

Produce a reference document establishing, empirically and from bytecode, the exact
evaluation order for at least fifteen constructs. Minimum set:

`f(a(), b(), c=d(), *e(), **g())`, `{a(): b(), c(): d()}`, `[a(), b()]`,
`x[a()] = b()`, `x.attr = a()`, `a() if b() else c()`, `a() and b() or c()`,
`[a(x) for x in b()]`, `{a(x): b(x) for x in c()}`, `f"{a()}{b()}"`,
`a() + b() * c()`, chained assignment `p = q[a()] = b()`,
chained comparison `a() < b() < c()`, `with a() as x, b() as y:`,
`try/finally` with a `return` in each.

For each: the bytecode, a side-effect experiment confirming it, and a statement of whether
the Language Reference **guarantees** the order or whether it is CPython implementation
detail. That last column is the assessed part; most people assume everything is guaranteed
and most of it is not.

### Part B — The scope and lifetime analyzer

A tool that, given a module's source, reports:

- Every name's scope category (from `symtable`).
- Closures over loop variables (the late-binding bug).
- Names read before assignment in a scope where a later assignment makes them local.
- Class-body names referenced from methods without `self.`.
- Suspected retention hazards: `lru_cache` on methods, module-level mutable containers,
  exceptions stored in containers, generators assigned to long-lived attributes.

Run it on a real project of at least 5,000 lines. Report findings **and** your
false-positive rate, established by manually reviewing every finding.

### Part C — A cleanup-correctness test harness

A pytest plugin or fixture set that, for a given resource-holding class, automatically
verifies cleanup under: normal exit, exception, `return` inside `with`, `break` inside
`with` in a loop, generator abandonment (`GeneratorExit`), `KeyboardInterrupt` during the
body, and interpreter shutdown with the object still referenced. Apply it to three classes
from the standard library and three of your own.

### Design note (1,200–1,800 words)

- Which of Part A's fifteen orders are guaranteed and which are not? What would you do
  differently in code you ship, knowing this?
- Part B's false-positive rate: what causes it, and what would it take to reduce? Argue
  whether a linter for these patterns is worth having at that rate.
- Part C: which of your six scenarios do real standard-library classes fail? Is that a bug
  or a documented limitation?
- Overall: name the single most useful thing you learned in PY-501 and the single thing you
  are least confident about. The second is worth more marks than the first.

---

## Submission checklist

For each set:

- [ ] Code in `courses/py501/psN/`, importable, with a README that runs from a clean clone.
- [ ] Tests passing, with a stated coverage figure *and* a sentence on what coverage does
      not tell you here.
- [ ] `mypy --strict` clean, or a documented list of the ignores and why each is necessary.
- [ ] Design note as `NOTE.md`.
- [ ] Self-marked against the rubric, with the marks and one line of justification each,
      committed as `MARK.md`.
- [ ] `log/failures.md` updated with anything you got wrong on the way.
