# PY-502 — Problem Sets

Marked against the five-criterion rubric in `00-program/assessment-and-rubrics.md`.

**A note on evidence.** Every claim about an abstraction's cost must be measured, not asserted.
"Descriptors are slower" is a sentence; "here is the benchmark, here is the noise floor, here is the
2.3× difference and here is what it becomes once the specialising interpreter warms up" is an
answer.

**A note on the constructions.** Each set builds on the lessons' §3 constructions. Where a part
names lesson stages, do those first.

**A note on subtraction.** Several parts ask you to remove an abstraction, or to justify one against
the simpler mechanism below it on the ladder. Those are the assessed parts. A submission that only
adds machinery has demonstrated half the skill.

---

## Problem Set 1 — A Declarative Framework, Built Four Ways
**Covers L01–L04 · Budget: 14–18 hours**
*Builds on: L01 §3 stages 1–6, L02 §3 (all stages), L03 §3 stages 1–7, L04 §3 (all stages)*

Build the *same* small declarative framework four times, using a different mechanism each
time, then compare.

**The requirement.** A `Record` base such that:

```python
class Account(Record):
    name  = Str(min_len=1, max_len=80)
    email = Str(pattern=EMAIL_RE)
    limit = Int(min_value=0, default=1000)
    tier  = Enum(Tier, default=Tier.BASIC)

a = Account(name="Mike", email="m@example.com")   # keyword-only, validated
a.limit = -5                                       # ValueError
Account.fields()                                   # introspectable
Account.to_json_schema()                           # derived
```

Implementations:

1. **Descriptors + `__init_subclass__`** (no metaclass).
2. **Descriptors + a metaclass** — including `__prepare__` to reject duplicate field names
   and to preserve declaration order explicitly.
3. **A class decorator** reading annotations, generating `__init__` by `exec`.
4. **`dataclasses` + a validation layer**, i.e. as little custom machinery as possible.

Each must: type-check under `mypy --strict` *and* `pyright`, with
`@dataclass_transform` where applicable; support inheritance (subclass adds a field);
survive `copy`, `deepcopy`, and `pickle` with validation reapplied; and produce good error
messages naming the field and the rule violated.

**Comparison table** (the assessed artifact): for each implementation — lines of machinery,
what a type checker knows at a call site, IDE autocomplete behaviour, `grep`-ability of a
field name, class-creation cost, instantiation cost, import cost, composability with
`abc.ABC` and with another library's base class, and the traceback quality on a validation
failure.

**Design note (1,200–1,500 words).** Which would you ship, for a library and for an
application (different answers are allowed and should be defended)? What did the four-way
comparison teach you that building one would not have? And: what does your framework do that
`attrs` does not — answer honestly.

### Marking emphasis

Subtraction. Four implementations are required; the marks are in the comparison and in identifying
which one you would actually ship, with the cost of the others named.

---

## Problem Set 2 — A Streaming Pipeline with Correct Resource Semantics
**Covers L05–L07 · Budget: 12–16 hours**
*Builds on: L05 §3 (all stages), L06 §3 stages 1–7, L07 §3 (all stages)*

Build a pipeline library and use it on a real workload of ≥1 GB.

**Requirements**

- Stages compose; each stage takes an iterator and returns an iterator; no stage performs
  I/O acquisition of its own.
- An explicit **error policy**: per-record failures are routed, counted, and sampled, with a
  configurable abort threshold.
- **Correct cleanup in all four termination cases** (L05 §2.4): full consumption, early
  break, abandonment, interpreter exit. Verified by test, with
  `-W error::ResourceWarning` enabled.
- A **resource-owning driver** built with `ExitStack` + `pop_all` (L07 §2.3), tested with
  fault injection at every acquisition point.
- **Aggregated cleanup failures** as an `ExceptionGroup`, never masking an original
  exception.
- A **sans-I/O parsing stage** (L06 §3) driven by `send`, testable with byte chunks and
  property-tested over random chunkings including one byte at a time.
- **Observability**: a passthrough progress stage that does not consume the iterator, and
  per-stage timing.
- An **async variant** of the driver, with a documented analysis of what happens when
  cleanup is cancelled.

**Measurements to report:** peak memory for eager vs lazy on the same input; throughput;
the memory profile of the one stage that decides it (L05 §2.7), measured both exactly and
approximately.

**Design note (1,000–1,400 words).** The yield/send rhythm you chose for the parser and why.
The error policy's threshold and its justification. What your cleanup story still does not
guarantee, and why the correct answer to that gap is idempotence rather than more cleanup
code.

### Marking emphasis

Correctness under failure. A pipeline that works on the happy path is the starting point, not the
deliverable. The resource semantics under early exit, exception, and abandonment are the assessed
part.

---

## Problem Set 3 — An Internal DSL and Its Critique
**Covers L08–L10 · Budget: 12–16 hours**
*Builds on: L08 §3 stages 1–6, L09 §3 (all stages), L10 §3 stages 1–5*

**Part A — The ten-uses test.** Before building anything: write ten realistic uses of your
proposed DSL and the same ten in plain Python (L09 §2.7). Include this in the deliverable
regardless of what you conclude. If the plain version wins, say so and build the DSL anyway
for the exercise — but your design note must own the finding.

**Part B — Build it.** A validation-rule, query, or workflow DSL. Required:

- Immutable fluent chaining.
- At least one operator-overloading construct, with `__bool__` raising to catch the
  precedence trap.
- Source-location capture, so every error names the user's line and shows it.
- Construction-time validation for at least six distinct misuses, each with a message
  saying what to write instead.
- Introspection: `describe()` and a schema/documentation generator.
- A visible, non-injectable escape hatch.
- A stated position on what the type checker does and does not know.

**Part C — Two model layers.** Wire your DSL into a boundary: a strict `pydantic` (or
`msgspec`) request model, plain `attrs`/`dataclass` domain objects, explicit translation,
and a separate response model (L08 §3). Include the evolution table: cost of renaming a wire
field, adding a domain-only field, and adding a status, in the one-layer and two-layer
designs.

**Part D — The audit.** Run the L10 §3 abstraction audit on the thing you just built, plus
one real codebase. Inline one abstraction and report the measurements.

**Design note (1,400–1,800 words).** Ship or don't ship, argued. What a new team member must
learn. What the tooling loses. The two abstractions you decided *not* to remove and why.
And the one thing you built during this course that you now think should not exist.

### Marking emphasis

Judgement. The critique of your own DSL carries more marks than the DSL. Applying the ten-uses
test honestly to your own work, and reporting a negative result, scores in the upper band.

---

## Course position paper (1,500 words)

**Driving question: when is metaprogramming a better answer than repetition?**

Claim, grounds, rebuttal, limits. Draw your grounds from the four-way comparison in PS1, the
tooling losses in PS3, and the audit measurements in PS3 Part D. The rebuttal section must
state fairly the strongest case *for* heavier metaprogramming than you recommend — the case
that frameworks like SQLAlchemy and pytest are enormously valuable precisely because they
were willing to pay these costs — and then answer it.

---

## Submission checklist (per set)

- [ ] Code in `courses/py502/psN/`, runnable from a clean clone.
- [ ] Tests, including the failure paths — most bugs in this course live on paths a
      happy-path test never executes.
- [ ] `mypy --strict` and `pyright` both clean, or every exception documented with a reason.
- [ ] Measurements with machine, OS, and Python build recorded; medians and spread, never a
      single run.
- [ ] `NOTE.md` — the design note.
- [ ] `MARK.md` — self-marked, one line of justification per criterion.
- [ ] `log/failures.md` updated.
