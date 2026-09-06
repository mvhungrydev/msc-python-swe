# SE-511 · Lesson 05 — Test Architecture: Levels, Contracts, and Cost

**Estimated study time:** 3.5 hours
**Prerequisites:** L01–L04

---

## 1. Orientation

Two teams, same product. Team A has 12,000 unit tests running in 40 seconds and a
production defect every week. Team B has 800 tests running in 9 minutes and a defect every
two months. Neither team's problem is "not enough tests".

Test architecture is the question of **what confidence you buy at what cost, and where.**
It is an engineering trade-off with a real budget — CI minutes, developer wait time,
maintenance effort — and it should be designed as deliberately as the system's architecture.
Mostly it is not; it accretes.

## 2. Theory

### 2.1 Levels, and why the usual names are unhelpful

"Unit", "integration", "end-to-end" are used inconsistently enough to be nearly useless in
discussion. Google's *size/scope* split is sharper and worth adopting:

**Size** — what resources the test may use. This is mechanically checkable and determines
speed and flakiness:

| Size | May use | Typical time |
|---|---|---|
| Small | One process, no I/O, no sleep, no network, no filesystem | < 100 ms |
| Medium | One machine: localhost, filesystem, containers, real DB | < 5 s |
| Large | Multiple machines, real external services | seconds to minutes |

**Scope** — how much code the test is *about*: one function, one class, one module, one
service, the system.

The two are independent. A small test can have wide scope (an in-memory end-to-end test of
your whole application, with fakes at the edges — often the best test you can write). A
large test can have narrow scope (one query against a real database).

Adopting the size vocabulary lets you make enforceable rules: *small tests run on every
save; medium on every push; large on merge to main and nightly.* Mark them
(`@pytest.mark.medium`) and enforce the constraint (a fixture that fails a small test which
opens a socket). Enforcement matters: without it, a small test acquires a network call and
the whole tier becomes slow and flaky.

### 2.2 The pyramid, the trophy, and what they are really arguing about

The **test pyramid** (Cohn): many unit, fewer integration, fewest end-to-end. The
justification is cost: higher-level tests are slower, flakier, and harder to diagnose, so
push detection down.

The **testing trophy** (Dodds): a fat integration layer, on the argument that unit tests of
heavily-mocked units verify little (L02 §2.4), and integration tests are where the real
confidence is.

They are not actually in conflict. The real variable is **how much of your system's risk
lives in the composition versus in the units.**

- A system whose complexity is *algorithmic* — a compiler, a solver, a pricing engine —
  has its risk in units. Pyramid.
- A system whose complexity is *integration* — a service that mostly moves data between an
  HTTP API, a queue, and a database — has its risk in the wiring. Trophy.

Most business software is the second kind, which is why the trophy argument gained ground.
Most of the code *you* find hard is the first kind, which is why the pyramid feels right to
the people who wrote it.

The useful formulation: **put the test at the lowest level that can actually observe the
risk.** If the risk is "the SQL is wrong", a mocked repository test cannot see it and a unit
test is worthless there — you need a real database. If the risk is "the discount
calculation is wrong", an end-to-end test is an absurdly expensive way to find out.

### 2.3 The cost model

For each test, four costs and one benefit:

| | |
|---|---|
| **Authoring cost** | one-off |
| **Runtime cost** | per run × runs per day × developers |
| **Maintenance cost** | changes required per unrelated refactor — this dominates over a system's life |
| **Diagnosis cost** | time from red to root cause |
| **Benefit** | probability of catching a defect × cost of that defect escaping |

Two consequences people miss:

**Maintenance cost dominates.** A test suite is a second implementation of your system's
behaviour. Every coupling between test and implementation (L02) is maintenance debt that
you pay on every change, forever. This is why deleting bad tests is a net positive and why
"more tests" is not a goal.

**Diagnosis cost scales with scope.** A failed unit test names the function. A failed
end-to-end test says "checkout returned 500". Diagnosis of the second can take hours. This
is the strongest argument for the pyramid, and it is an argument about *humans*, not about
correctness.

### 2.4 Test doubles at the architectural boundary

From L02 §2.3: define ports you own, fake them, contract-test the adapters. Architecturally
this produces a specific and very effective shape:

```
     ┌─────────────────────────────────────────┐
     │  Domain + application logic              │   ← small tests, wide scope,
     │  (no I/O, pure, fast)                    │      real objects, no mocks
     └───────────────┬─────────────────────────┘
                     │ ports (Protocols you own)
     ┌───────────────┴─────────────────────────┐
     │  Adapters: DB, HTTP, queue, clock, fs    │   ← medium tests, narrow scope,
     └───────────────┬─────────────────────────┘      real dependencies, contract tests
                     │
              external systems                       ← a few large tests
```

The payoff: **the majority of your tests are small tests with wide scope.** They exercise
whole use cases through the real domain objects, with fakes only at the outer boundary. They
are fast, they do not mock internals, and they survive refactoring because they assert on
outcomes.

This is why SE-521's hexagonal architecture lesson and this lesson are the same idea seen
from two directions. A system that is hard to test at this shape usually has business logic
in its adapters.

### 2.5 Contract testing

The problem: service A calls service B. A's tests stub B. B's team changes B. A's tests
still pass. Production breaks.

**Consumer-driven contract testing** (Pact and similar) closes this:

1. A's test suite records the requests it makes and the responses it expects — the
   *contract*.
2. The contract is published to a broker.
3. B's CI *replays* the contract against real B and fails if B no longer satisfies it.

The result: B cannot break A without B's build going red, and neither team needs a shared
end-to-end environment. This is the single highest-leverage testing technique for
service-oriented systems, and it is underused because it requires cross-team agreement
rather than just code.

Its limits, which you should state whenever you advocate it: it verifies *the interactions
the consumer actually makes*, not B's whole API; it does not verify semantics beyond the
message shape (B can return a well-formed but wrong answer); and it requires discipline
about contract versioning. Schema registries (DI-721 L09) address the shape half at the
data layer with the same philosophy.

### 2.6 Test data and environments

Three strategies, in increasing order of maintainability:

- **Shared fixtures / seeded database.** Fast to start, fatal over time: tests become
  order-dependent, nobody can change the seed data, and every test's preconditions are
  invisible.
- **Per-test construction via builders.** Each test builds exactly what it needs. Explicit,
  slower, correct. Strongly preferred.
- **Generated data** (L04). For properties and for load.

For the database specifically: run a real one in a container (`testcontainers`), one
schema per test session, and wrap each test in a transaction that is rolled back. This is
fast enough (milliseconds per test) that "we mock the database because it's slow" is
usually a stale belief worth re-measuring.

### 2.7 What to do about end-to-end tests

They are expensive, flaky, and slow — and you cannot have zero, because some risks only
exist in the assembled system (config, deployment, network policy, auth). The workable
position:

- Keep a **small, fixed number** — the critical user journeys, perhaps five to fifteen.
  Enumerate them; do not let the set grow by accretion.
- Treat them as **smoke tests for the deployment**, not as functional coverage.
- Run them **against a real deployed environment**, post-deploy, as part of a progressive
  rollout with automatic rollback. At that point they are indistinguishable from monitoring,
  and that is the right way to think about them (CA-731 L07: *testing in production* is not
  an abdication, it is the recognition that some properties are only observable there).
- Every E2E failure must produce an artifact — screenshot, HAR, trace ID — or diagnosis
  cost swamps the benefit.

## 3. Construction: designing a suite for a real service

Take a service with an HTTP API, a database, a queue consumer, and a third-party payment
call. Design its test architecture explicitly.

**Step 1 — enumerate the risks.** Not the components — the *risks*.

| Risk | Where observable |
|---|---|
| Pricing/discount logic wrong | domain, pure |
| Order state machine allows an invalid transition | domain, pure |
| SQL returns wrong rows under concurrency | real database |
| Migration breaks on existing data | real database, real migration |
| Payment gateway retry double-charges | adapter + contract |
| Queue message handled twice | adapter + integration |
| Auth misconfigured | assembled system |
| API contract broken for the mobile client | contract test |

**Step 2 — assign each risk to the lowest level that can see it.** Half the table lands in
"domain, pure" — small tests, no doubles at all, testing real objects. That is the bulk of
the suite and it should be.

**Step 3 — write the budget.**

| Tier | Count | Wall clock | When |
|---|---|---|---|
| Small (domain + application, fakes at ports) | ~600 | < 20 s | every save |
| Medium (adapters vs real DB/queue in containers) | ~90 | < 3 min | every push |
| Contract (consumer + provider verification) | ~15 | < 30 s | every push |
| Large (E2E against deployed env) | 8 | < 6 min | post-deploy |

Numbers are illustrative; the *practice* of writing the budget down is the deliverable.
A team with an explicit budget makes different decisions than one without.

**Step 4 — enforce the tiers mechanically.** A `conftest.py` fixture that, for tests marked
`small`, patches `socket.socket` to raise. A CI job per tier. A rule that a PR may not
increase the medium tier's runtime by more than X.

**Step 5 — decide what you will not test.** Explicitly. Write it down: "we do not test our
ORM's SQL generation", "we do not test framework routing", "we do not have UI tests; we rely
on E2E smoke plus type-checked API clients". A team that cannot state what it does not test
has no architecture, only accumulation.

## 4. Failure modes

- **The ice-cream cone** — mostly E2E, few unit. Slow, flaky, undiagnosable. Usually the
  result of a QA team writing tests separately from the developers.
- **The hourglass** — many unit, many E2E, nothing in between. Composition bugs escape to
  E2E, where they are expensive to diagnose.
- **Uncontrolled E2E growth.** Each new feature adds an E2E test "to be safe" and the suite
  takes 90 minutes.
- **Mocked database.** Tests pass; the SQL is wrong. Run a real one.
- **Shared mutable test environment.** Two pipelines, one staging database. Everything is
  flaky and nobody knows why.
- **No tier enforcement.** Small tests acquire I/O and the fast tier stops being fast.
- **Retry-on-failure in CI.** Converts a flaky-test signal into invisible risk (L01 §2.7).
- **Testing through the UI what could be tested through the API.** An order of magnitude
  more cost for the same assertion.
- **No contract at service boundaries.** §2.5.

## 5. Exercises

### Warm-up (25 min)

**W1.** Classify 20 tests from a real suite by size and by scope. Plot them on a 2×2. What
shape is your suite?

**W2.** Time your test suite by tier. Compute developer-hours per year spent waiting
(runs/day × developers × 250 days).

**W3.** Find an E2E test that could be a small test. Rewrite it. Report the time saved and
what confidence, if any, was lost.

### Core (2 h)

**C1 — The risk table and budget.** Complete §3 steps 1–5 for a system you actually work on.
Deliverable: the risk table, the tier assignment with justification for each row, the
budget, the enforcement mechanism, and the explicit not-testing list. This is a document you
can take to your team.

**C2 — Real database tests.** Convert a set of mocked-repository tests to run against a real
database in a container, with transaction rollback per test. Measure: runtime before and
after, and how many defects the real-database version catches that the mocked version does
not (write a broken query and see). Report whether the trade was worth it.

**C3 — A consumer-driven contract.** Set up a contract test between two components (they can
both be yours). Demonstrate the full loop: consumer records expectations, provider
verification fails when the provider changes, and passes when fixed. Then write 400 words on
what the contract does *not* verify and what you would add.

**C4 — Tier enforcement.** Implement a pytest plugin that fails any test marked `small`
which opens a socket, touches the filesystem outside `tmp_path`, or sleeps. Run it on a real
suite and report how many "unit" tests violate the constraint.

### Challenge

**X1.** Read the testing chapters of *Software Engineering at Google* (11–14) and Dodds's
"Write Tests. Not Too Many. Mostly Integration." Write 1,200 words reconciling them: state
the strongest version of each, identify the empirical claim on which they actually differ,
and describe an experiment that would settle it for your codebase.

**X2.** Instrument your CI to record, for every failed build over three months, the test
that failed and the time to green. Produce a cost analysis: which tests cost the most
developer time, and how much of that was genuine defect detection versus flakiness or
brittleness. Recommend deletions with the evidence.

## 6. Self-check

1. Define test *size* and *scope* and give an example of small-size/wide-scope.
2. State the principle that resolves the pyramid-vs-trophy argument.
3. Name the four costs of a test and say which dominates over a system's life.
4. Why does diagnosis cost scale with scope, and what follows?
5. Draw the port/adapter test shape and say which tier holds most of the tests.
6. What does consumer-driven contract testing guarantee, and what does it not?
7. Give three rules for keeping E2E tests useful.
8. Why is "we don't test X" a sign of a healthy test architecture?

## 7. Primary sources

- Winters et al., *Software Engineering at Google*, chs. 11–14. Ch. 11's size/scope
  taxonomy is the important part.
- Cohn, *Succeeding with Agile* — the original pyramid.
- Fowler, "TestPyramid" and "IntegrationTest" (bliki) — the second is a careful essay on
  how ambiguous the word "integration" is.
- Pact documentation, "Contract Testing" introduction.

---

**Previous:** [L04](L04-property-based-testing.md) · **Next:**
[L06 — Gradual Typing: Theory and Practice](L06-gradual-typing.md)
