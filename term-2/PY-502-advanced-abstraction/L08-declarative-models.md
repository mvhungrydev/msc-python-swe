# PY-502 · Lesson 08 — Declarative Models: dataclasses, attrs, pydantic

**Estimated study time:** 4 hours
**Prerequisites:** L02, L03, L04; SE-511 L06

---

## 1. Orientation

You now know every mechanism these libraries use: descriptors, `__set_name__`,
`__init_subclass__`, class decorators, code generation via `exec`, and
`@dataclass_transform`. This lesson is about the *choice* between them, which is one of the
most consequential per-project decisions in modern Python and is usually made by habit.

The question is not "which is best". It is: **what is this class for, and what does it need
to guarantee?**

## 2. Theory

### 2.1 The four jobs, and why they are different

A "model class" is asked to do up to four separate things:

1. **Reduce boilerplate** — generate `__init__`, `__repr__`, `__eq__`, `__hash__`.
2. **Enforce invariants at construction** — validation.
3. **Convert between representations** — parse from JSON/dict/form data, serialize back.
4. **Describe itself** — generate a schema (OpenAPI, JSON Schema), a form, a table.

These are genuinely distinct, and a class that needs only (1) should not pay for (3).
The most common architectural error in this area is using one library for everything —
typically a validating parser used as the domain model — which couples the domain to the
wire format and makes every schema change a domain change.

### 2.2 `dataclasses`

Standard library. Generates methods from annotations via `exec`'d source (PY-501 L07 §2.8).

```python
from dataclasses import dataclass, field

@dataclass(frozen=True, slots=True, kw_only=True, order=False)
class Order:
    id: OrderId
    lines: tuple[Line, ...]
    created_at: datetime
    note: str | None = None
    _cache: dict[str, str] = field(default_factory=dict, compare=False, repr=False)

    def __post_init__(self) -> None:
        if not self.lines:
            raise ValueError("order must have at least one line")
```

The options that matter:

| Option | Effect |
|---|---|
| `frozen=True` | generates a raising `__setattr__`; combined with `eq` gives `__hash__` |
| `slots=True` | builds a **new class** with `__slots__`; watch decorator stacking |
| `kw_only=True` | keyword-only `__init__`; makes field additions non-breaking |
| `order=True` | generates comparisons from the field tuple — rarely what you want |
| `field(default_factory=...)` | the fix for mutable defaults (PY-501 L01 §4.1) |
| `field(compare=False)`, `hash=False`, `repr=False` | per-field control |
| `InitVar[T]` | a constructor parameter that is not a field |

**Use it when:** the data comes from your own code, you control every construction site, and
you need (1) plus perhaps a light `__post_init__` check for (2).

**Strengths:** zero dependencies, zero import cost, universally understood, works with
`match`, plays well with type checkers, and produces plain Python objects with no runtime
machinery in the hot path.

**Weaknesses:** no validation to speak of (annotations are not enforced —
`Order(id="oops")` runs fine); no conversion; `__post_init__` is a blunt instrument for
per-field rules; no schema.

`kw_only=True` deserves special mention: it makes adding a field a non-breaking change and
eliminates the two-arguments-of-the-same-type-swapped bug. Make it your default.

### 2.3 `attrs`

The library `dataclasses` was derived from, and still ahead of it.

```python
from attrs import define, field, validators as v

@define(frozen=True, kw_only=True)
class Order:
    id: OrderId
    lines: tuple[Line, ...] = field(validator=v.min_len(1))
    note: str | None = field(default=None, validator=v.optional(v.max_len(500)))
    created_at: datetime = field(factory=lambda: datetime.now(UTC))

    @lines.validator
    def _same_currency(self, attribute, value) -> None:
        if len({l.currency for l in value}) > 1:
            raise ValueError("mixed currencies")
```

What it adds over `dataclasses`:

- **Per-field validators and converters**, composable, with good error messages.
- `__attrs_post_init__`, `on_setattr` hooks, and field aliases.
- `attrs.evolve()` for functional updates of frozen instances (`dataclasses.replace` is the
  equivalent and is fine).
- `attrs.fields()` introspection that is richer than `dataclasses.fields()`.
- Slots by default in `@define`, with correct inheritance handling.
- `cattrs` as a *separate* library for structuring/unstructuring — which is exactly the
  right separation: validation and serialization are different jobs (§2.1).

**Use it when:** you want (1) and (2) with real per-field rules, but you do not want a
parsing/serialization framework welded to your domain model.

**Weaknesses:** a dependency; two APIs (`attr.s`/`attr.ib` legacy and `attrs.define`/`field`
modern) that make search results confusing; no schema generation.

### 2.4 `pydantic`

A **parsing and validation** library, and the distinction from "a model library" is the
whole point.

```python
from pydantic import BaseModel, Field, field_validator, ConfigDict

class CreateOrder(BaseModel):
    model_config = ConfigDict(frozen=True, extra="forbid", strict=False)

    customer_id: int
    lines: list[LineIn] = Field(min_length=1)
    note: str | None = Field(default=None, max_length=500)

    @field_validator("lines")
    @classmethod
    def _same_currency(cls, v: list[LineIn]) -> list[LineIn]:
        if len({l.currency for l in v}) > 1:
            raise ValueError("mixed currencies")
        return v

order = CreateOrder.model_validate(request.json())   # parses AND validates AND coerces
```

What it adds:

- **Coercion.** `"3"` → `3` by default. This is `pydantic`'s most useful and most dangerous
  feature: excellent at a wire boundary, alarming inside a domain. `strict=True` disables
  it per-model or per-field, and on a domain model you almost certainly want it.
- **Serialization** — `model_dump()`, `model_dump_json()`, with field aliases, exclusion
  sets, and custom serializers.
- **JSON Schema generation** — which is what makes FastAPI's automatic OpenAPI docs work.
- **Rust core** (v2, `pydantic-core`) — validation is fast, often faster than hand-written
  Python checks.
- Rich, structured error reports (`ValidationError.errors()` gives a list of
  `{loc, msg, type}` suitable for returning to an API caller directly).

**Use it when:** data crosses a trust boundary — HTTP bodies, config files, message
payloads, third-party API responses, CLI arguments. This is the anti-corruption layer
(SE-511 L06 §2.7).

**Weaknesses:** a heavy dependency with a compiled core; `BaseModel` inheritance imposes
itself on your class hierarchy; the coercion defaults surprise people; import time is
non-trivial for CLIs; and — the important one — **it invites you to make your wire schema
your domain model.**

`msgspec` is the lighter alternative: faster still, no coercion by default, works with
`dataclasses` and `attrs` types as well as its own `Struct`, no schema-generation ecosystem.
Worth considering when you need speed and not the ecosystem.

### 2.5 The decision

| Need | Choose |
|---|---|
| A record used only inside your own code | `dataclass(frozen=True, kw_only=True, slots=True)` |
| Domain entity with real invariants | `attrs` (or `dataclass` + `__post_init__` if simple) |
| Anything parsed from outside | `pydantic` (or `msgspec`) at the boundary |
| Config from files/env | `pydantic-settings` or equivalent |
| Hot-path DTO, millions per second | `msgspec.Struct` or a `slots` dataclass — measure |
| A value object with behaviour | a plain class; none of the above generate behaviour |

**The architectural rule** that follows, and which is the real content of this lesson:

> **Two model layers, not one.** A `pydantic` (or `msgspec`) model at each boundary, and
> plain `dataclass`/`attrs` domain objects inside, with an explicit translation between
> them.

Costs: duplication (two shapes for the same concept) and translation code. Benefits: the
domain does not change when the wire format does; the domain has no dependency on a
validation library; the translation is where you handle version skew, defaults, and renames;
and the domain model can have invariants the wire format cannot express.

When to skip it: small applications, internal tools, and prototypes, where the wire format
*is* the domain and will change together. Say so explicitly rather than sliding into it. The
signal that you should have split them is your first "we can't rename this field because
it's in the API".

### 2.6 Typing and `@dataclass_transform`

All three tell type checkers what they generate via PEP 681:

```python
from typing import dataclass_transform

@dataclass_transform(kw_only_default=True, field_specifiers=(field,))
class ModelBase: ...
```

Without it, a checker sees a class with class-level annotations and no `__init__`, and
either infers the wrong signature or gives up. If you build a declarative framework
(Problem Set 1), this is not optional.

Note the limits: `@dataclass_transform` describes *`__init__` synthesis from fields*. It
cannot describe arbitrary generated methods. Anything else your framework generates is
invisible unless you ship stubs or a plugin.

### 2.7 Performance

Rough shape, to be verified with your own measurements (PY-602 L01 — and *do* verify, these
change between versions):

- **Instantiation**: `msgspec.Struct` ≲ slotted `dataclass` ≈ `attrs` slots < plain
  `dataclass` < `pydantic` (which validates).
- **Attribute access**: slotted variants faster than `__dict__` variants; `pydantic` v2
  models are close to plain since validation happens at construction.
- **Memory**: slots saves ~40–60% per small instance.
- **Import time**: `dataclasses` ≈ 0; `attrs` small; `pydantic` significant (tens of ms) —
  which matters for CLI startup and for Lambda cold starts (CA-731 L04).
- **Validation**: `pydantic` v2's Rust core often beats hand-written Python validation, which
  surprises people who assume "a library must be slower".

The rule: for the 99% of classes that are not in a hot loop, choose on semantics, not speed.
For the 1%, measure — and be prepared for the measurement to contradict your intuition.

## 3. Construction: one concept, three layers

Take an `Order` and build it properly across a boundary.

**Step 1 — the wire model.**

```python
class OrderRequest(BaseModel):
    model_config = ConfigDict(extra="forbid", frozen=True)
    customer_id: int
    lines: list[LineRequest] = Field(min_length=1)
    idempotency_key: str = Field(pattern=r"^[A-Za-z0-9_-]{8,64}$")
```

`extra="forbid"` is a deliberate choice: reject unknown fields rather than ignore them. It
catches client typos immediately and it means adding a field to the API is a visible
decision. The alternative — tolerant readers — is the right choice for
*consuming* messages you do not control (DI-721 L09 argues both sides). Decide per boundary
and write down why.

**Step 2 — the domain model.**

```python
@define(frozen=True, kw_only=True)
class Order:
    id: OrderId
    customer: CustomerId
    lines: tuple[OrderLine, ...] = field(validator=v.min_len(1))
    status: OrderStatus = OrderStatus.PENDING

    @property
    def total(self) -> Money:
        return sum((l.total for l in self.lines), start=Money.zero(self.currency))

    def cancel(self) -> "Order":
        if self.status is not OrderStatus.PENDING:
            raise IllegalTransition(self.status, OrderStatus.CANCELLED)
        return evolve(self, status=OrderStatus.CANCELLED)
```

Note what the domain model has that the wire model does not: an identity, a status with
transition rules, `Money` (not `float`), and *behaviour*. And what the wire model has that
the domain does not: an idempotency key, which is a transport concern.

**Step 3 — translation, explicitly.**

```python
def to_domain(req: OrderRequest, id: OrderId, catalog: Catalog) -> Order:
    return Order(
        id=id,
        customer=CustomerId(req.customer_id),
        lines=tuple(OrderLine(sku=l.sku, qty=l.qty, price=catalog.price(l.sku))
                    for l in req.lines),
    )
```

Notice this function needs the catalog — the price is *not* client-supplied. That security
property is expressible only because the two models are separate; a single model with a
`price` field trusts the client with pricing, which is a real and recurring vulnerability
class.

**Step 4 — the response model,** distinct again, because what you return is not what you
store:

```python
class OrderResponse(BaseModel):
    id: str
    status: Literal["pending", "paid", "cancelled"]
    total: str            # a decimal string, not a float — JSON floats lose precision
    lines: list[LineResponse]
```

**Step 5 — measure the cost you just paid.** Count the lines of translation code. Then
count the changes required, in each design, for three realistic evolutions: rename a wire
field; add a domain-only field; add a status. Report the table. That is the honest
cost/benefit of the two-layer rule and it will make the argument concrete for you in a way
no essay can.

## 4. Failure modes

- **One model for everything.** The wire schema becomes the domain, and vice versa.
- **Trusting client-supplied values that should be server-derived** (prices, ids,
  timestamps, roles). The two-layer split prevents it structurally.
- **`pydantic` coercion inside the domain.** `strict=True`, or do not use it there.
- **Mutable defaults.** `field(default_factory=list)` — `dataclasses` will actually raise
  for known-mutable defaults, but not for your own mutable classes.
- **`@dataclass` without `frozen`** on a value object, then used as a dict key.
- **`slots=True` under another decorator that captured the original class.** It returns a
  new class.
- **`order=True` by reflex.** Generates comparisons over all fields, usually meaningless.
- **Positional `__init__`.** Field additions become breaking; use `kw_only=True`.
- **No `@dataclass_transform` on a custom framework.** Users get no typing.
- **JSON floats for money.** Use a decimal string on the wire.
- **`extra="ignore"` at an inbound boundary you own.** Client typos silently ignored.
- **Optimizing model choice before measuring.** §2.7.

## 5. Exercises

### Warm-up (25 min)

**W1.** Show that a `dataclass` does not validate annotations: construct one with wrong
types for every field and demonstrate it runs.

**W2.** Demonstrate `pydantic`'s default coercion on five inputs where it does something you
would not want in a domain model. Then show `strict=True` rejecting them.

**W3.** Show that `dataclass(slots=True)` returns a new class, and construct a program where
it matters.

### Core (2.5 h)

**C1 — Three layers.** Complete §3, all five steps, for a real concept from your work.
Deliverable: the three models, the translations, the evolution table from step 5, and a
600-word note taking a position on whether the split was worth it *for this case*.

**C2 — Library comparison, empirically.** Implement the same 8-field model in
`dataclass(slots)`, `attrs`, `pydantic` (strict and non-strict), and `msgspec`. Measure:
instantiation rate, attribute access, memory per instance, validation throughput on a
realistic mixed-validity input, JSON round-trip throughput, and import time. Report a table
with medians and spread and the machine/version details. Then state which you would choose
for four different scenarios.

**C3 — Boundary hardening.** Take an existing endpoint whose handler accepts a raw dict.
Introduce a strict request model, an explicit domain model, and a response model. Then write
fifteen malicious or malformed inputs (extra fields, wrong types, huge strings, deeply
nested JSON, unicode edge cases, negative quantities, client-supplied price) and verify each
is rejected with a useful message. Report the ones your original code accepted.

**C4 — Error messages.** Take `pydantic`'s `ValidationError.errors()` output for five bad
inputs and turn it into an API error response a client developer could act on. Then do the
same for `attrs` validators and for hand-written checks. Compare the effort and the quality.
Error-message quality is a real product feature and this exercise makes that concrete.

### Challenge

**X1.** Build a translation layer that is *checked*: a mechanism that fails at import time
if a domain field has no mapping from the wire model, or vice versa, so that adding a field
cannot silently produce a partially-populated object. Consider: `dataclass_transform`,
`__init_subclass__` verification, or a `mypy` plugin. Report which you chose and what it
cannot catch.

**X2.** Read `dataclasses.py` in full, and the `attrs` `_make.py`. Write 1,200 words
comparing their code-generation strategies: how each builds `__init__`, how each handles
inheritance and slots, how each interacts with descriptors, and what `attrs` does better.
Identify one thing `dataclasses` does better.

## 6. Self-check

1. Name the four jobs a model class may be asked to do, and why conflating them is a
   problem.
2. Give three things `attrs` provides that `dataclasses` does not.
3. What is `pydantic` actually for, and what is its most dangerous default?
4. State the two-layer rule and the cost you pay for it.
5. Give three evolutions that are cheap under the two-layer rule and expensive without it.
6. Why does the two-layer split prevent a whole class of security bug?
7. What does `@dataclass_transform` express, and what can it not express?
8. Why should `kw_only=True` be your default?

## 7. Primary sources

- PEP 557 (`dataclasses`), PEP 681 (`@dataclass_transform`).
- `attrs` documentation, "Why not…?" page — an unusually honest comparison.
- `pydantic` v2 documentation, "Conversion Table" and "Strict Mode".
- Evans, *Domain-Driven Design*, ch. 5 (entities, value objects) — the vocabulary behind
  §2.5's rule.

---

**Previous:** [L07](L07-context-managers-and-resources.md) · **Next:**
[L09 — Internal DSLs and Fluent Interfaces](L09-internal-dsls.md)
