# SE-521 · Lesson 03 — Dependency Inversion, Composition, and Their Limits

**Estimated study time:** 3.5 hours
**Prerequisites:** L01, L02; PY-502 L01

---

## 1. Orientation

Dependency inversion is the most misapplied principle in the SOLID set. The correct version
is a real and powerful idea. The folk version — "put an interface in front of everything" —
produces codebases full of single-implementation interfaces, a `Factory` per class, and a
DI container nobody understands, with no reduction in change cost whatsoever.

This lesson gives the correct version, and — unusually for a lesson on a design principle —
spends serious time on where applying it makes systems worse.

## 2. Theory

### 2.1 The principle, stated correctly

> **High-level modules should not depend on low-level modules. Both should depend on
> abstractions. Abstractions should not depend on details; details should depend on
> abstractions.**

The word doing the work is **inversion**. Without it:

```
OrderService  ──depends on──►  PostgresOrderRepository  ──►  psycopg
```

The high-level policy (what an order *is*, when it can be cancelled) depends on a low-level
detail (which database). Change the database, change the policy module.

With it:

```
OrderService  ──depends on──►  OrderRepository (interface)
                                       ▲
                                       │ implements
                        PostgresOrderRepository
```

The arrow from the detail to the abstraction is *inverted* relative to the flow of control:
at runtime `OrderService` calls into `PostgresOrderRepository`, but at compile/import time
the dependency points the other way.

**The part everyone gets wrong: where the interface lives.** If `OrderRepository` lives in
the infrastructure package, nothing has been inverted — the domain still imports from
infrastructure, just one level indirected. The interface must live **with the high-level
module that needs it**, because the high-level module is the one that defines the
requirement.

In Python, `typing.Protocol` makes this exact (PY-502 L01 §2.3): the domain declares the
Protocol, the infrastructure class satisfies it structurally without importing anything, and
the type checker verifies the match at the wiring point. No inheritance, no runtime
coupling, no import from domain to infrastructure *or* from infrastructure to domain except
for the domain types themselves.

```python
# domain/ports.py   — imports nothing outside domain
class OrderRepository(Protocol):
    def get(self, id: OrderId) -> Order | None: ...
    def save(self, order: Order) -> None: ...

# infrastructure/postgres.py — imports domain types only
class PostgresOrderRepository:
    def get(self, id: OrderId) -> Order | None: ...
    def save(self, order: Order) -> None: ...

# main.py — the composition root, the only place both are known
service = OrderService(repo=PostgresOrderRepository(pool))
```

Enforce it with `import-linter` (SE-511 L08 §2.4). Without enforcement, the layering decays
one import at a time and nobody notices for a year.

### 2.2 The composition root

There should be **one place** where concrete implementations are chosen and wired: the
composition root — `main.py`, the app factory, the CLI entry point. Everywhere else takes
its dependencies as parameters.

```python
def build_app(cfg: Config) -> App:
    pool = create_pool(cfg.dsn)
    repo = PostgresOrderRepository(pool)
    payments = StripeGateway(cfg.stripe_key) if cfg.live else FakeGateway()
    events = KafkaPublisher(cfg.brokers)
    return App(OrderService(repo, payments, events))
```

Properties of a good composition root: it is boring; it contains no logic beyond
configuration; it is the only place `if cfg.live` appears; and reading it tells you the
whole shape of the system. That last property is worth a great deal — a well-written
composition root is the best architecture documentation a codebase can have.

**Constructor injection is the default.** Dependencies arrive as constructor parameters and
are stored. Not: looked up from a global registry, imported at module level, or fetched from
a service locator. The test is whether an object's dependencies are visible in its
signature; if they are not, you have hidden coupling (L02 §2.6).

**Do you need a DI container?** In Python, usually not. Constructor injection plus a
composition root of fifty lines handles most applications. A container earns its place when
the object graph is large, has many shared instances with distinct lifetimes
(singleton/per-request/transient), and is assembled from plugins. Below that threshold it
adds a framework, obscures the graph, and moves errors from import time to runtime. If you
do use one, prefer explicit registration over auto-wiring by type — auto-wiring is where the
graph becomes unreadable.

### 2.3 Composition over inheritance, revisited

PY-501 L04 §2.7 gave the rule: *inherit to be substitutable; compose to reuse.* The
architectural elaboration:

Inheritance couples you to the *implementation structure* of the base class, not just its
interface. The **fragile base class problem**: a base class method that calls another
overridable method establishes an implicit protocol. Change which internal methods the base
calls — a change that preserves the base's own behaviour — and every subclass that overrode
one of them breaks.

```python
class Collection:
    def add(self, x): self._items.append(x)
    def add_all(self, xs):
        for x in xs: self.add(x)          # subclass overriding add() gets called

class Counting(Collection):
    def add(self, x): self.count += 1; super().add(x)
    def add_all(self, xs): self.count += len(xs); super().add_all(xs)   # double-counts!
```

The bug exists because `Counting` had to know whether `add_all` calls `add`. That is
implementation knowledge, and it is not in the interface. Composition has no equivalent
failure: a wrapper knows only the public methods.

**When inheritance is right:** a genuinely closed, small set of variants; a template method
where the hook points are the *documented* interface (and are abstract, not concrete); and
value hierarchies with no behaviour. When it is wrong: code reuse, "is-kind-of" that is
really "has-a", and anything with more than three levels.

### 2.4 Where dependency inversion makes things worse

The honest part of the lesson.

**1. Single-implementation interfaces.** An interface with one implementer and no prospect
of a second adds a file, a hop, and an indirection while providing no polymorphism. The
common justification is testing — but a fake *is* a second implementation, so this is a
legitimate reason. The illegitimate version is "flexibility" for a variation nobody has
requested. YAGNI applies; the correct criterion is *reversibility* (PY-502 L10 §4): if
introducing the interface later would be cheap and localized, do it later.

**2. Inverting a stable dependency.** You do not need an abstraction over `json`,
`datetime`, `math`, or your own domain value objects. Inversion pays when the detail is
*likely to change* (Parnas, L01 §2.1). `datetime.now()` is the interesting borderline: the
*library* is stable but *testability* wants it injected — so inject a `Clock`, not because
the library might change but because time is an input to your logic. State the reason; the
reason determines the shape.

**3. Interfaces that leak.** A `Repository` Protocol with `execute_sql(query: str)` on it has
inverted nothing. Similarly, a repository whose users must know it does N+1 queries, or must
call `session.flush()` at the right moment, has an interface that does not actually hide the
implementation. The abstraction must be defined in the *domain's* vocabulary or it is a
pass-through with extra ceremony.

**4. The generic repository.** `Repository[E]` with `get/save/delete/find` looks like the
canonical example and mostly does not pay off (PY-502 L01 §3 arrived at this from the type
side). Real queries are domain-specific (`orders_awaiting_payment_older_than(d)`), do not
generalize, and end up either as a leaky `find(criteria)` that reimplements a query language
badly, or as bespoke methods that make the generic base pointless. Prefer a *specific*
interface per aggregate, named in domain terms.

**5. Over-abstracted configuration.** A `ConfigProvider` interface over what could be a
frozen dataclass constructed once at startup.

**6. Ports for things that are not boundaries.** Two modules in the same layer, both yours,
both deployed together, both changing together — an interface between them is a hop with no
inversion value.

### 2.5 Which dependencies to invert: the decision

Invert when at least two hold:

- The dependency crosses an **architectural boundary** (domain ↔ infrastructure, your code ↔
  a third party, one bounded context ↔ another).
- The detail is **likely to change** independently of the policy.
- You need a **test double** and no other seam is available.
- There genuinely are, or will imminently be, **multiple implementations**.
- The dependency is on something **slow, remote, or non-deterministic** (network, clock,
  randomness, filesystem).

Do not invert when: the dependency is on a stable standard library; both sides are yours and
change together; or the only argument is "flexibility".

The last three bullets of the first list generate a useful checklist for any module: *what
in here is non-deterministic or external?* Those are your ports. Everything else is domain
logic and should be a pure function of its inputs — which is the L05 argument, arrived at
from a different direction.

### 2.6 Functional dependency inversion

Inversion does not require an interface. Passing a *function* is dependency inversion with
less ceremony:

```python
def place_order(
    order: Order,
    *,
    save: Callable[[Order], None],
    charge: Callable[[Money, Card], ChargeId],
    now: Callable[[], datetime],
) -> Receipt: ...
```

For a small number of dependencies this is lighter than a Protocol and just as testable, and
it makes the dependencies visible in the signature. It stops scaling at about four
parameters, and it loses the ability to group related operations, which is what an interface
is *for*. Both forms are legitimate; use the function form for one or two operations and a
Protocol when the operations belong together as a concept.

The stronger version of the same idea, the **functional core / imperative shell**: keep all
decisions in pure functions that take data and return data (including descriptions of
effects), and confine the effects to a thin shell. Then most of the system needs no
inversion at all, because it has no dependencies to invert.

```python
# core: pure. No inversion needed; nothing to inject.
def decide(order: Order, now: datetime, rates: Rates) -> list[Command]: ...

# shell: impure, thin, hard to unit test and barely worth testing
def handle(order_id):
    order = repo.get(order_id)
    for cmd in decide(order, clock.now(), rates.snapshot()):
        execute(cmd)
```

This is the design that makes SE-511 L05's "small tests, wide scope" tier possible, and it
is worth reaching for before reaching for interfaces.

## 3. Construction: inverting one real dependency

**Step 1 — find a concrete dependency across a boundary.** A module of business logic that
imports a database client, an HTTP library, `datetime.now`, `random`, or `os.environ`.

**Step 2 — write down why you are inverting it.** One of the five criteria in §2.5. If you
cannot name one, stop — you have found an exercise in ceremony.

**Step 3 — define the Protocol in the consuming module,** in the *consumer's* vocabulary.
Not `execute(sql)`; `orders_for_customer(id)`. The naming test: could you implement this
Protocol over a completely different technology — a file, an HTTP API, an in-memory dict —
without the name becoming a lie?

**Step 4 — move the wiring to the composition root.** Remove the import. Verify with
`import-linter` that the boundary now holds.

**Step 5 — write the fake and the contract test** (SE-511 L02 §2.6). The contract test is
what makes the fake trustworthy; without it you have a fast lie.

**Step 6 — measure.** Before and after: how many tests need the real dependency; test suite
runtime; and — the honest one — the number of files a reader must open to answer "where does
an order get saved?" That number usually goes *up*. Report it. The trade is real and you
should be able to state both sides.

**Step 7 — the counter-exercise.** Find a place in the same codebase where dependency
inversion was applied and should not have been (§2.4). Inline it. Report the same
measurements. A submission with only step 1–6 shows technique; one with step 7 shows
judgement.

## 4. Failure modes

- **Interface in the wrong package.** Nothing inverted.
- **Single-implementation interfaces for "flexibility".**
- **A `Factory` for every class.**
- **A DI container in a 5,000-line application.**
- **Auto-wiring by type**, making the object graph unreadable and errors runtime-only.
- **Service locator** — a global registry that objects pull from. Hides dependencies,
  defeats the signature test, makes test isolation hard. It is the anti-pattern that
  dependency injection exists to replace.
- **Leaky ports** (`execute_sql`) that invert nothing.
- **Generic repositories.**
- **Inverting stable dependencies.**
- **Inheritance for reuse**, then the fragile base class problem.
- **No composition root** — construction scattered, and `if TESTING:` in production code.
- **Never measuring the reader cost.** §3 step 6.

## 5. Exercises

### Warm-up (25 min)

**W1.** Find an interface in your codebase and determine where it lives relative to its
consumer and implementer. Say whether anything is actually inverted.

**W2.** Construct the fragile base class bug of §2.3 and then show that a composition-based
version cannot have it.

**W3.** Find a service locator or global registry. Trace one object's real dependencies and
compare with its constructor signature.

### Core (2 h)

**C1 — Invert one, inline one.** Complete §3, all seven steps. Deliverable: both changes,
both sets of measurements including the reader-cost number, and a 600-word note on how you
decided each.

**C2 — Build a composition root.** For an application without one, extract all construction
into a single `build_app(cfg)`. Report: how many `if TESTING`/environment checks you removed
from production code, how many module-level singletons you eliminated, and what the root
reveals about the system's shape that was not previously visible.

**C3 — Functional core.** Take a service method with I/O interleaved through the logic.
Restructure into a pure decision function returning commands, plus a thin executing shell.
Write tests for the core with no doubles at all. Report: lines in core vs shell, number of
test doubles before and after, test runtime before and after.

**C4 — Repository critique.** Take a generic repository in real code. Enumerate every method
its consumers actually use. Replace it with per-aggregate interfaces named in domain terms.
Report the diff and whether the total interface surface went up or down.

### Challenge

**X1.** Read Martin's "The Dependency Inversion Principle" (1996) and the relevant chapters
of *Clean Architecture*, then read one of the substantial critiques (there are several
well-argued ones). Write 1,500 words taking a position on the claim that "clean
architecture" as popularly practiced produces more indirection than it repays, using
measurements from your own C1 and C4.

**X2.** Build a fitness function (L08) that enforces your layering: no import from an inner
layer to an outer one, ports declared only in inner layers, and concrete infrastructure
referenced only in the composition root. Run it on a real codebase, report violations, and
classify each as a genuine violation or a sign the layer definition is wrong.

## 6. Self-check

1. State DIP precisely, and say which word does the work.
2. Where must the interface live, and why does the common mistake invert nothing?
3. What is a composition root, and what are the properties of a good one?
4. Explain the fragile base class problem and why composition cannot have it.
5. Give six situations where inverting a dependency makes the system worse.
6. Give the criteria for when to invert, and say what they imply about identifying ports.
7. What is a service locator and why is it worse than constructor injection?
8. State the functional core / imperative shell idea and what it does to the need for
   inversion.

## 7. Primary sources

- Martin, "The Dependency Inversion Principle" (C++ Report, 1996) and *Clean Architecture*,
  Part III.
- Fowler, "Inversion of Control Containers and the Dependency Injection Pattern" (2004) —
  the article that named the composition root idea, and it is still the clearest.
- Seemann, *Dependency Injection Principles, Practices, and Patterns* — the composition root
  and anti-pattern chapters.
- Bernhardt, "Functional Core, Imperative Shell" (2012 talk).
- Percival & Gregory, *Architecture Patterns with Python*, chs. 2–4.

---

**Previous:** [L02](L02-coupling-cohesion-change.md) · **Next:**
[L04 — Design Patterns as Vocabulary — and the Critique](L04-design-patterns.md)
