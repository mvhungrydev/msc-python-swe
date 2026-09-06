# SE-521 · Lesson 05 — Ports, Adapters, and the Dependency Rule

**Estimated study time:** 4 hours
**Prerequisites:** L01–L04

---

## 1. Orientation

Hexagonal architecture (Cockburn, 2005), onion architecture (Palermo, 2008), and clean
architecture (Martin, 2012) are the same idea with different diagrams:

> **Dependencies point inward. The inside knows nothing about the outside.**

The idea is sound and the popular practice around it is often bad — four layers of DTOs, a
mapper between each, and an `IUserRepository` with one implementation. This lesson teaches
the principle, the honest cost, and how to decide how much of it you need.

## 2. Theory

### 2.1 The dependency rule

```
        ┌──────────────────────────────────────────┐
        │  drivers / entrypoints                   │   HTTP handlers, CLI, consumers, cron
        │  ┌────────────────────────────────────┐  │
        │  │  application (use cases)           │  │   orchestration, transactions
        │  │  ┌──────────────────────────────┐  │  │
        │  │  │  domain                      │  │  │   entities, value objects, rules
        │  │  └──────────────────────────────┘  │  │
        │  └────────────────────────────────────┘  │
        │  driven adapters                         │   DB, queue, HTTP clients, filesystem
        └──────────────────────────────────────────┘

        imports point INWARD only
```

The rule, stated as something you can enforce: **no module in an inner layer may import
from an outer layer.** The domain imports nothing but the standard library and other domain
modules. Not the ORM, not the web framework, not `requests`, not the settings module.

`import-linter` enforces exactly this (SE-511 L08 §2.4), and *the rule is worth nothing
unenforced*. Every layered architecture decays; the only ones that do not have a build step
that fails.

### 2.2 Ports: driving and driven

A **port** is an interface at the boundary. Two kinds, and the distinction matters because
they are owned differently:

- **Driving (primary) ports** — how the outside invokes the application. Your use-case
  interfaces: `PlaceOrder`, `CancelOrder`. Drivers (an HTTP handler, a CLI command, a queue
  consumer) call them. In Python these are usually just functions or a service class, and a
  formal Protocol is often unnecessary because there is exactly one implementation and many
  callers.
- **Driven (secondary) ports** — what the application needs from the outside:
  `OrderRepository`, `PaymentGateway`, `Clock`, `EventPublisher`. These *must* be Protocols
  declared in the inner layer (L03 §2.1), because the inner layer is the one imposing the
  requirement.

The asymmetry is the useful insight: **driving ports rarely need an interface; driven ports
always do.** Most codebases get this backwards, defining an interface for the service (one
implementation, many callers — no polymorphism needed) and calling the database directly.

### 2.3 Adapters

An **adapter** implements a port against a specific technology.

```
Driving adapters:  FastAPI route → PlaceOrder use case
                   Click command → PlaceOrder use case
                   Kafka consumer → PlaceOrder use case

Driven adapters:   PostgresOrderRepository  → OrderRepository port
                   InMemoryOrderRepository  → OrderRepository port  (the fake)
                   StripeGateway            → PaymentGateway port
                   FakeGateway              → PaymentGateway port
```

Two properties define a good adapter:

- **Thin.** It translates and nothing else. Business rules in an adapter are the defect this
  architecture exists to prevent, and they are also the most common violation.
- **The only place its technology appears.** `psycopg` should be importable from exactly one
  package. Enforce it with a `banned-api` rule (SE-511 L08 §2.4) outside that package.

The test-double payoff is structural, not incidental: because ports are narrow and driven
adapters are thin, the in-memory fake for each port is 20–40 lines, and the contract test
(SE-511 L02 §2.6) proves it behaves like the real one. Then the *entire application layer*
is testable in-process, in milliseconds, with no mocks — SE-511 L05's "small size, wide
scope" tier.

### 2.4 What lives in each layer

**Domain** — entities, value objects, domain services, domain events, and the rules. Pure.
No I/O, no framework, no ORM, no `datetime.now()` (time comes in as a parameter), no
randomness, no config. The test: *could you run this layer in a REPL with no dependencies
installed?*

**Application (use cases)** — orchestration. A use case: loads via ports, calls domain
logic, persists via ports, publishes events, and owns the transaction boundary (L07). It
contains no business rules — if a use case has an `if` about the domain, that rule belongs
in the domain.

**Adapters** — translation. SQL, HTTP, serialization, framework-specific code.

**Entrypoints / composition root** — wiring, configuration, process lifecycle.

The two boundary questions that decide most real designs:

**Where does validation go?** Both places, differently. *Format* validation ("is this a
well-formed email string?") is at the boundary, in the request model (PY-502 L08). *Domain*
validation ("can this order be cancelled in its current state?") is in the domain, as an
invariant that cannot be violated by construction. Splitting them this way means the domain
never has to defend against malformed input and can express real rules cleanly.

**Where do transactions go?** The application layer, via a Unit of Work port (L07). Not the
domain (which knows nothing of persistence) and not the adapter (which cannot see the scope
of a use case).

### 2.5 The honest costs

Say these out loud whenever you propose this architecture:

- **More files.** A feature touches a domain object, a port, an adapter, a use case, a
  request model, a response model, and a route. Seven files for something that could be one
  function.
- **Mapping code.** Domain ↔ persistence and domain ↔ wire. Real, boring, and a place bugs
  hide.
- **You will fight your ORM.** Persistence-ignorant domain objects mean either a classical
  mapper (SQLAlchemy's imperative mapping supports this well) or hand-written mapping. Active
  Record ORMs (Django) are fundamentally incompatible with a pure domain layer, and pretending
  otherwise produces the worst of both.
- **Indirection cost for readers.** The reader trace (PY-502 L10 §2.1) is longer.
- **It is not free to reverse.** Adding layers later is cheap; removing them is not, because
  everything depends on them.

**When it is worth it:** the domain has genuine rules that change independently of the
technology; the system will live for years; multiple entry points (HTTP + CLI + consumer)
drive the same logic; you need fast tests of complex logic; a technology swap is plausible.

**When it is not:** CRUD over a schema, where the "domain logic" is validation and the
"use case" is a database write. There, the layers are pure ceremony and a well-organized
framework application is better. The most valuable skill here is being able to say
*"this application does not need this"* and mean it.

### 2.6 Pragmatic Python

You do not have to take the whole package. In increasing order of investment:

**Level 0 — no layers.** Framework views calling the ORM. Correct for small CRUD.

**Level 1 — extract the domain.** Pure functions and value objects in a `domain` package
that imports nothing. Views still call the ORM but call the domain for decisions. **This is
the highest-value single step and it costs almost nothing.** Most applications should be at
least here.

**Level 2 — driven ports for the volatile dependencies.** Repository and gateway Protocols
in the domain, adapters outside, wired in the composition root. Fakes and contract tests.

**Level 3 — explicit use cases and a Unit of Work.** Application layer with transaction
boundaries.

**Level 4 — full separation** with a mapper layer and no ORM types in the domain.

Choose a level deliberately and *write it in an ADR* (L10). Most systems belong at level 2
or 3; level 4 is justified by a genuinely complex domain, and level 0 by a genuinely simple
one. The failure is drifting between levels without deciding, which produces a codebase with
the costs of level 4 and the benefits of level 1.

### 2.7 Persistence ignorance in practice

The domain should not know it is persisted. Practical means, from least to most invasive:

- **SQLAlchemy imperative (classical) mapping** — map plain domain classes to tables in a
  separate `orm.py`. The domain class has no ORM base, no columns, no metaclass. This works
  well and is the standard answer in Python.
- **Hand-written mappers** — `to_row(order) -> dict` and `from_row(row) -> Order`. Explicit,
  tedious, complete control, no framework surprises. Correct when the domain shape and the
  storage shape genuinely differ.
- **Separate persistence models** — an ORM model *and* a domain model, with translation.
  Most work, most independence. Justified when the schema is shared or legacy.

Things that break persistence ignorance and that you should watch for: lazy loading (the
domain touches an attribute and a query fires — an invisible I/O dependency); identity maps
leaking session semantics into domain equality; and cascade rules encoding domain
invariants in the ORM configuration where nobody looks for them.

## 3. Construction: layering a real feature

Take one feature end-to-end: "cancel an order, refund the payment, notify the customer".

**Step 1 — write the domain first, with no imports.**

```python
# domain/order.py
@define(frozen=True)
class Order:
    id: OrderId
    status: OrderStatus
    lines: tuple[OrderLine, ...]
    paid_at: datetime | None

    def cancel(self, now: datetime, policy: CancellationPolicy) -> "CancellationResult":
        if self.status is not OrderStatus.PAID:
            raise IllegalTransition(self.status, OrderStatus.CANCELLED)
        if not policy.allows(self, now):
            raise CancellationWindowClosed(self.paid_at, now)
        return CancellationResult(
            order=evolve(self, status=OrderStatus.CANCELLED),
            refund=Refund(self.id, self.total),
            events=(OrderCancelled(self.id, now),),
        )
```

Note: `now` is a parameter; the method returns a *result object* describing what should
happen rather than doing it. That is the functional core (L03 §2.6) and it is what makes
this testable with no doubles.

**Step 2 — the ports, in the inner layer.**

```python
# domain/ports.py
class OrderRepository(Protocol):
    def get(self, id: OrderId) -> Order | None: ...
    def save(self, order: Order) -> None: ...

class PaymentGateway(Protocol):
    def refund(self, refund: Refund) -> RefundId: ...

class EventPublisher(Protocol):
    def publish(self, events: Sequence[DomainEvent]) -> None: ...
```

**Step 3 — the use case.**

```python
# application/cancel_order.py
@dataclass(frozen=True)
class CancelOrder:
    orders: OrderRepository
    payments: PaymentGateway
    events: EventPublisher
    uow: UnitOfWork
    clock: Clock
    policy: CancellationPolicy

    def __call__(self, id: OrderId) -> None:
        with self.uow:
            order = self.orders.get(id)
            if order is None:
                raise OrderNotFound(id)
            result = order.cancel(self.clock.now(), self.policy)
            self.orders.save(result.order)
            self.payments.refund(result.refund)      # ← see step 5
            self.uow.commit()
        self.events.publish(result.events)
```

No business rules. Orchestration only.

**Step 4 — adapters and the composition root.** A Postgres repository, a Stripe gateway, an
outbox publisher, and the in-memory fakes for each. Contract tests over both.

**Step 5 — find the bug.** The refund is a call to an external system *inside a database
transaction*. Consequences: the transaction is held open across a network call (lock
contention, connection exhaustion); if the commit fails after the refund succeeds, you have
refunded and not recorded it; if the refund times out, you do not know whether it happened.

This is a real, extremely common defect, and the architecture made it *visible* — that is
the argument for the architecture, made concretely. The fix is the outbox pattern
(DI-721 L08): commit the state change and an intent record atomically, then perform the
external call from a separate process, idempotently. Implement it.

**Step 6 — test the layers separately.** Domain: pure, no doubles, dozens of cases in
milliseconds. Application: fakes for all ports, asserting on state. Adapters: contract tests
against real infrastructure. Entrypoint: one smoke test. Report the runtime of each tier.

**Step 7 — count the cost.** Files touched, lines of mapping code, and the reader trace for
"what happens when an order is cancelled". Compare with the same feature written directly in
a framework view. Report both honestly and state at which of §2.6's levels this application
should sit.

## 4. Failure modes

- **Business rules in adapters.** The most common violation.
- **The domain importing the ORM, the settings module, or `datetime.now()`.**
- **An interface for the driving port** (one implementation, many callers) and a direct call
  for the driven one. Backwards.
- **Anemic domain model** — entities with only data, all logic in "services". You have
  layers and no domain; this is L06's central warning.
- **Mapping layers that do nothing** — a DTO identical to the entity.
- **Unenforced layering.** Decays within months.
- **External calls inside transactions.** §3 step 5.
- **Fighting an Active Record ORM** to achieve persistence ignorance.
- **Applying level 4 to a CRUD app.**
- **Lazy loading in the domain** — invisible I/O.
- **No fakes, so the layering buys no test speed** — you did the work and skipped the payoff.

## 5. Exercises

### Warm-up (25 min)

**W1.** Take a codebase and determine, with the import graph, whether any layering exists.
Write the `import-linter` contract that describes the layering you *wish* existed, and run
it. Report the violation count.

**W2.** Find a business rule living in an adapter. Move it to the domain. Report what else
had to move.

**W3.** Find an external call inside a database transaction. Describe the three failure
modes it creates.

### Core (2.5 h)

**C1 — The feature, layered.** Complete §3, all seven steps including the outbox fix.
Deliverable: the code, the contract tests, the per-tier test runtimes, the cost count, and a
600-word note choosing a level from §2.6 for this application with justification.

**C2 — Persistence ignorance.** Take an entity currently coupled to an ORM. Make it
persistence-ignorant using SQLAlchemy imperative mapping (or hand-written mappers). Report:
lines of mapping code added, what the domain tests no longer need, and one thing that got
harder.

**C3 — Level assessment.** For three systems you know, assess which of §2.6's levels each
sits at, whether that is the right level, and what the next step up or down would cost.
Write it as an ADR (L10) for one of them.

**C4 — The anemic check.** For a domain package, compute the ratio of methods-with-logic to
plain-data-accessors on entities, and count business rules found in the application layer.
Report. If the domain is anemic, move three rules into it and report what improved.

### Challenge

**X1.** Read Cockburn's original "Hexagonal Architecture" post, Martin's *Clean
Architecture* Part V, and Percival & Gregory's *Architecture Patterns with Python*
chs. 2–6. Then read a serious critique of clean architecture. Write 1,500 words: state the
strongest case that these architectures produce more ceremony than they repay, answer it
with evidence from your C1 measurements, and give a decision procedure for when to adopt
which level.

**X2.** Implement the outbox pattern completely: atomic write of state + intent, a relay
process, idempotent delivery with retries, deduplication at the consumer, and monitoring for
stuck messages. Write the tests including: crash between commit and relay, duplicate
delivery, and relay running twice concurrently. This is DI-721 L08 material done early and
it is the single most useful pattern in this lesson.

## 6. Self-check

1. State the dependency rule and how you would enforce it mechanically.
2. Distinguish driving from driven ports, and say which needs an interface and why.
3. Give the two properties of a good adapter.
4. Where do format validation and domain validation each belong, and why the split?
5. Give five honest costs of this architecture.
6. Give the five levels of §2.6 and say which step has the best cost/benefit.
7. Name three things that break persistence ignorance in practice.
8. Why is an external call inside a transaction a defect, and what is the fix?

## 7. Primary sources

- Cockburn, "Hexagonal Architecture" (2005).
- Martin, *Clean Architecture* (2017), Part V.
- Percival & Gregory, *Architecture Patterns with Python* (2020) — the Python-native
  treatment; chs. 2–7.
- Palermo, "The Onion Architecture" (2008).
- Fowler, "AnemicDomainModel" (bliki).

---

**Previous:** [L04](L04-design-patterns.md) · **Next:**
[L06 — Domain-Driven Design: Language, Model, Boundaries](L06-domain-driven-design.md)
