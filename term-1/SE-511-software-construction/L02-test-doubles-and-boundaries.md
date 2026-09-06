# SE-511 · Lesson 02 — Test Doubles and the Boundaries They Imply

**Estimated study time:** 3.5 hours
**Prerequisites:** L01

---

## 1. Orientation

Two tests of the same function:

```python
def test_sends_welcome_email():
    mailer = Mock()
    register(user, mailer)
    mailer.send.assert_called_once_with(to=user.email, template="welcome")
```

```python
def test_sends_welcome_email():
    mailer = InMemoryMailer()
    register(user, mailer)
    assert mailer.sent == [Email(to=user.email, template="welcome")]
```

They look equivalent. They are not. The first asserts that `register` **called a method**;
the second asserts that **an email exists**. Change `send(to=..., template=...)` to
`send(Message(...))` and the first test fails while the system still works. Introduce a
second call path that sends the email twice and the second test catches it while the first
passes if you used `assert_called_with` rather than `assert_called_once_with`.

Choosing a double is choosing what your test is coupled to. That is the subject here.

## 2. Theory

### 2.1 The taxonomy (Meszaros)

Five kinds, with distinct purposes:

| Kind | What it is | Asserts on |
|---|---|---|
| **Dummy** | A placeholder, never used | nothing |
| **Stub** | Returns canned answers to queries | the system's *output* |
| **Fake** | A real, working, simplified implementation | the system's *output* |
| **Spy** | Records calls, delegates or returns defaults | interactions, after the fact |
| **Mock** | Pre-programmed with expectations, verifies them | interactions, as the test's *purpose* |

The distinction that matters most: **stubs and fakes support state verification; spies and
mocks support behaviour verification.**

`unittest.mock.Mock` is all five at once, which is why the vocabulary collapsed in Python
practice and why people reach for it reflexively. Knowing the taxonomy lets you ask "which
of these do I actually want?" and the answer is usually "a fake".

### 2.2 Commands and queries

Meyer's command–query separation gives the decision rule.

- A **query** returns a value and has no side effects. Double it with a **stub** — you need
  it to produce input for the system under test, and you assert on what the system did with
  it. Never assert that the query was called: that is an implementation detail.
- A **command** changes state elsewhere and returns nothing meaningful. There is no return
  value to assert on, so the *only* way to verify it happened is to observe the effect —
  either in a fake, or by asserting the interaction with a mock.

So: **stub queries, mock commands, and never the reverse.** Asserting `assert_called_with`
on a query is the single most common mocking error, and it produces exactly the brittle
tests that give mocking its bad reputation.

### 2.3 "Don't mock what you don't own"

A rule from the London school, and it is correct.

When you mock `boto3.client("s3").put_object(...)`, your test asserts your understanding of
S3's API. If that understanding is wrong — wrong parameter name, wrong exception type,
wrong behaviour on retry — your test passes and production fails. Worse, when AWS changes
something, your test keeps passing. **A mock of a third-party API tests nothing about the
third party and freezes your possibly-wrong beliefs about it.**

The alternative is an *adapter you own*:

```python
class BlobStore(Protocol):
    def put(self, key: str, data: bytes) -> None: ...
    def get(self, key: str) -> bytes: ...

class S3BlobStore:      # thin; the only place boto3 appears
    ...

class InMemoryBlobStore:    # a fake, used by every other test
    ...
```

Now: everything above `BlobStore` is tested against the in-memory fake, fast and
deterministically. `S3BlobStore` is tested against real S3 or `moto`/LocalStack, in a small
number of slow tests. The boundary is explicit, the surface you must trust is one file, and
your beliefs about S3 are verified in one place.

This is the same structure SE-521 L05 calls a *port and adapter*. The testing argument and
the architecture argument are the same argument.

### 2.4 The cost of mocks, stated precisely

Every mock encodes an assumption. Three specific costs:

**Refactoring resistance.** Mocks assert on the call graph. A pure refactoring — one that
provably preserves behaviour — breaks them. Your test suite now punishes exactly the
activity it was supposed to enable.

**False confidence.** A suite of heavily-mocked unit tests can pass with every unit correct
and the *composition* broken. The classic: A returns `None` on failure, B expects an
exception. Both units are tested; both units are correct against their mocked
counterparts; the system is broken. This is why L05's test pyramid must have integration
tests, not merely as a formality.

**Specification drift.** A stub returns what you *believe* the collaborator returns. When
the collaborator changes, nothing tells you. Consumer-driven contract testing (L05 §2.5)
exists to close this gap and is the correct answer for service boundaries.

Against these: mocks are fast, deterministic, and let you test error paths that are hard to
produce for real (a network timeout at exactly the third retry). That is a genuine and
important benefit. The judgement is about where to spend the coupling.

### 2.5 `unittest.mock` in practice

The library is powerful and its defaults are hazardous.

**`Mock()` accepts everything.** `m.anything.you.want()` returns a new `Mock`. A typo in an
assertion silently passes:

```python
m = Mock()
m.assert_called_once()          # correct
m.assert_called_onse()          # typo — creates an attribute, always "passes"
```

Mitigations: `autospec`. `create_autospec(SomeClass)` or
`mock.patch(..., autospec=True)` builds a double whose signature matches the real object,
so wrong argument names and misspelled methods fail. **Use `autospec=True` always.** The
few cases where it is inconvenient are cases where you should be using a fake.

Since 3.5, `assert_called*` on a `Mock` raises `AttributeError` for names starting with
`assert_` that are not real methods — a partial fix. `NonCallableMock` and `spec_set` add
further tightening.

**`mock.patch` patches where it is *looked up*, not where it is defined.**

```python
# mymodule.py
from utils import fetch

def run(): return fetch()
```

```python
mock.patch("utils.fetch")       # WRONG — mymodule already bound the name
mock.patch("mymodule.fetch")    # right
```

This follows directly from PY-501 L01: `from utils import fetch` binds a *new name* in
`mymodule`'s namespace, pointing at the same object. Patching `utils.fetch` rebinds the
name in `utils`, and `mymodule.fetch` still points at the original. Understanding the
object model makes this obvious rather than folkloric.

**`patch` is a design smell in proportion to its frequency.** Each `patch` is a dependency
that was not injected. A module needing five patches to test is telling you its
dependencies are hard-coded. Sometimes that is acceptable (patching `time.time` in a small
utility); when it is the norm, the design is the problem.

### 2.6 Fakes are usually the right answer

A fake is a real implementation with different trade-offs: an in-memory repository, a
`FakeClock`, a `FakeMailer`, an in-process queue. Properties:

- Tests written against it assert on **state**, so they survive refactoring.
- One fake serves hundreds of tests, so the investment amortizes.
- It can enforce the interface's invariants (a fake repository can reject duplicate keys),
  turning the fake into a *specification of the port*.
- It is real code, so it can have bugs — which is why the fake and the real implementation
  should be run against **the same test suite** (a "contract test" for the port):

```python
class BlobStoreContract:                 # shared test base
    @pytest.fixture
    def store(self) -> BlobStore: raise NotImplementedError

    def test_get_after_put_returns_data(self, store): ...
    def test_get_missing_raises_keyerror(self, store): ...

class TestInMemory(BlobStoreContract):
    @pytest.fixture
    def store(self): return InMemoryBlobStore()

@pytest.mark.slow
class TestS3(BlobStoreContract):
    @pytest.fixture
    def store(self): return S3BlobStore(bucket=test_bucket())
```

This pattern is the single highest-leverage thing in this lesson. It gives you a fast fake
you can *trust*, because the same tests prove the fake and the real thing behave alike.

### 2.7 When mocks are right

Not never. Use a mock when:

- The interaction **is** the requirement. "An audit record is written for every denied
  request" is a claim about a call. Assert on it.
- You must simulate a failure that is impractical to produce (a socket timing out mid-body,
  a disk full at byte 4096).
- The collaborator is expensive and has no reasonable fake (a payment gateway), *and* you
  have a contract test at the boundary.
- You are characterizing legacy code and need to pin current behaviour before changing it.

In each case, mock **your own abstraction**, not the third party.

## 3. Construction: testing an order service

A service with three collaborators — a repository (state), a payment gateway (external
command with a return), and an event publisher (command).

**Version 1 — all mocks.**

```python
def test_place_order():
    repo, gateway, events = Mock(), Mock(), Mock()
    gateway.charge.return_value = ChargeResult(ok=True, id="ch_1")
    svc = OrderService(repo, gateway, events)

    svc.place(order)

    repo.save.assert_called_once()
    gateway.charge.assert_called_once_with(order.total, order.card)
    events.publish.assert_called_once()
```

What this test actually verifies: that `place` calls three methods. It does not verify that
the order was saved *with the charge id*, that the event carries the right data, that a
failed charge prevents the save, or that saving happens before publishing. Four real
requirements, none tested. And it will break if you rename `charge` to `authorize`.

**Version 2 — fakes for state, mock for the true command.**

```python
def test_successful_order_is_persisted_with_charge_id():
    repo = InMemoryOrderRepo()
    gateway = FakeGateway(approve=True)
    events = RecordingPublisher()
    svc = OrderService(repo, gateway, events)

    svc.place(order)

    saved = repo.get(order.id)
    assert saved.status is OrderStatus.PAID
    assert saved.charge_id == gateway.last_charge_id

def test_declined_payment_leaves_no_order():
    repo = InMemoryOrderRepo()
    svc = OrderService(repo, FakeGateway(approve=False), RecordingPublisher())

    with pytest.raises(PaymentDeclined):
        svc.place(order)

    assert repo.get(order.id) is None

def test_order_placed_event_is_published_after_persistence():
    repo, events = InMemoryOrderRepo(), RecordingPublisher()
    svc = OrderService(repo, FakeGateway(approve=True), events)

    svc.place(order)

    assert events.published == [OrderPlaced(order_id=order.id, total=order.total)]
```

Now every test states a requirement, none mentions a method name of a collaborator, and all
three survive any refactoring that preserves behaviour. `RecordingPublisher` is a *spy*
implemented as a fake — the honest form of "assert the command happened".

**Version 3 — the ordering requirement.** "Publish only after the save commits" cannot be
verified by either version. Options: give the fake repo a transaction that the publisher can
observe; record a global sequence in both fakes and assert on the order; or restructure so
the ordering is structural (outbox pattern — DI-721 L08) and cannot be got wrong. The third
is the right answer and the test drove you to it.

## 4. Failure modes

- **Mocking queries and asserting they were called.** §2.2.
- **Mocking third-party libraries.** §2.3.
- **`Mock()` without `autospec`.** Typos pass; signature changes pass.
- **Patching the wrong path.** §2.5.
- **`assert_called_with` when you meant `assert_called_once_with`.** The first only checks
  the *most recent* call.
- **Asserting on `call_count` as a proxy for behaviour.** Couples to the implementation.
- **A fake that drifts from the real implementation.** Fix with the contract-test pattern,
  §2.6.
- **`patch` as the default.** A count of `@patch` decorators per test file is a useful
  design metric; anything above two per test deserves a look at the constructor.
- **Mocking the system under test.** Partial mocks (`patch.object(svc, "_helper")`) test a
  chimera that does not exist in production. If you need this, the helper wants to be a
  collaborator.

## 5. Exercises

### Warm-up (25 min)

**W1.** Write the same test twice, once with a mock and once with a fake. Then perform a
behaviour-preserving refactor of the production code and report which test broke.

**W2.** Demonstrate the `patch` path problem of §2.5 with a two-module example, and explain
it in terms of PY-501 L01's binding model.

**W3.** Show that `Mock().assert_called_onse()` passes, and that `autospec` prevents an
equivalent error.

### Core (2 h)

**C1 — The port and its contract.** Take an external dependency in your work (an object
store, a queue, an HTTP API). Define a `Protocol` for the narrow slice you use. Write a
contract test suite. Implement a real adapter and an in-memory fake, both passing the
contract. Then rewrite three existing tests to use the fake. Report: lines of test code,
runtime, and what the contract test caught about your real adapter (it will catch
something).

**C2 — The order service.** Complete §3 through version 3, including the ordering
requirement. Deliverable: the tests, the fakes, the implementation, and a 400-word note on
which requirements were untestable in version 1 and what that says about mock-heavy suites
generally.

**C3 — Mock audit.** Take a test suite with heavy mocking. For every mock, classify: is the
doubled method a query or a command? Is the double owned by you? Is `autospec` used?
Produce a table. Then rewrite the worst five tests and measure the change in how many tests
break under a scripted refactor (e.g. rename a method with `rope` or an IDE).

**C4 — Failure injection.** Build a fake HTTP client that can be configured to fail in
seven realistic ways: connection refused, DNS failure, TLS error, timeout before headers,
timeout mid-body, 500, 429 with `Retry-After`. Test your retry logic (PY-501 L10 C1)
against all seven. Report which cases your original code got wrong.

### Challenge

**X1.** Read Fowler's "Mocks Aren't Stubs" and the London-vs-Chicago (mockist vs classicist)
debate. Write 1,000 words taking a position, with evidence from your own C3 results. Address
the strongest argument on the other side.

**X2.** Build a tool that statically finds tests whose assertions are *only* on mock
interactions and reports them as candidates for state-based rewriting. Run it on an
open-source project. Report the proportion, and hand-check 20 findings for false positives.

## 6. Self-check

1. Name the five kinds of test double and what each supports.
2. State the command/query rule for choosing a double.
3. Why does "don't mock what you don't own" follow from what a mock actually asserts?
4. Give the three costs of mocking, precisely.
5. Why does `mock.patch` need the *lookup* path rather than the definition path?
6. What is a contract test for a port, and what problem does it solve?
7. Give three cases where a mock is the right choice.
8. Why is a partial mock of the system under test a bad idea?

## 7. Primary sources

- Meszaros, *xUnit Test Patterns*, ch. 11 ("Using Test Doubles").
- Fowler, "Mocks Aren't Stubs" (2007).
- Freeman & Pryce, *GOOS*, chs. 2, 6, 8, 20.
- `unittest.mock` documentation — the "Autospeccing" and "Where to patch" sections.

---

**Previous:** [L01](L01-testing-as-specification.md) · **Next:**
[L03 — Coverage, Adequacy, and Mutation Testing](L03-coverage-adequacy-mutation.md)
