# SE-511 · Lesson 01 — Testing as Specification

**Estimated study time:** 3 hours
**Prerequisites:** none

---

## 1. Orientation

Dijkstra's line is quoted so often that it has stopped meaning anything:

> "Program testing can be used to show the presence of bugs, but never to show their
> absence."

It is worth reading as an actual claim. Testing is *sampling*. A test suite examines a
finite subset of the input space and reports on that subset. Everything you believe about
the untested inputs comes from an inductive argument you have made — usually implicitly,
often badly.

The whole craft of testing is: **make that inductive argument explicit, and choose the
sample so that the argument is strong.** A test that exercises `add(2, 2) == 4` supports
almost no induction. A test that asserts `add(a, b) == add(b, a)` for arbitrary `a, b`
supports a great deal. Same effort, different epistemic value.

This lesson reframes tests as *claims about a specification*, which changes what you write
and, more importantly, changes what you delete.

## 2. Theory

### 2.1 What a test is evidence of

A test is a triple: **a state of the world, an action, an assertion.** It asserts that a
specific claim about the system holds in a specific circumstance.

The question to ask of every test is: *what claim is this evidence for, and what would its
failure tell me?* Three answers, in decreasing value:

1. **"The system satisfies this requirement."** The test encodes something a user or
   another component depends on. Failure means a real defect. These are the tests worth
   having.
2. **"This function does what it did yesterday."** A characterization test. Failure means
   *something changed* — possibly intentionally. Valuable around legacy code you do not
   understand; a liability when it pins down accidental behaviour and blocks refactoring.
3. **"This code calls these methods in this order."** An implementation test. Failure means
   the implementation changed. Almost always a cost with no matching benefit: it fails on
   every refactor and passes through every real bug that preserves the call sequence.

Most low-value test suites are full of category 3 masquerading as category 1. The
diagnostic question: **if I rewrote the implementation completely but preserved the
observable behaviour, would this test still pass?** If not, it is testing the
implementation.

### 2.2 Tests as executable specification

A specification says what the system must do. Prose specifications rot because nothing
checks them. Tests are a specification that is *executed*, and therefore cannot silently
drift — but only if written to read like claims.

Compare:

```python
def test_process():
    r = process([1, 2, 3])
    assert r == [2, 4, 6]
```

with

```python
def test_process_doubles_each_element():
    assert process([1, 2, 3]) == [2, 4, 6]

def test_process_preserves_order():
    assert process([3, 1, 2]) == [6, 2, 4]

def test_process_of_empty_is_empty():
    assert process([]) == []
```

The second version is three claims. The first is one example. When the second's middle test
fails, you know what broke *from the name alone*. This is not cosmetic: the name is the
specification and the body is the evidence.

**Naming convention that scales:** `test_<subject>_<condition>_<expectation>`. Long names
are correct here; you never call these functions, you only read them in failure output.

### 2.3 Arrange–Act–Assert, and one act per test

```python
def test_withdrawal_below_balance_succeeds():
    account = Account(balance=Money(100, "USD"))       # arrange
    account.withdraw(Money(30, "USD"))                 # act
    assert account.balance == Money(70, "USD")         # assert
```

The discipline: **one act.** Multiple acts in one test means a failure does not localize —
you know the sequence failed, not which step. Multiple *assertions* about the same act are
fine and often good (they describe the post-state fully).

The related discipline: **assert on outcomes, not on steps.** `assert account.balance == ...`
is an outcome. `mock_ledger.record.assert_called_once_with(...)` is a step, and belongs
only where the interaction *is* the requirement (L02).

### 2.4 The test as a design instrument

The strongest argument for writing tests first is not defect reduction; the empirical
evidence on that is genuinely mixed. It is **feedback on design**. A unit that is hard to
test is telling you something:

| Test pain | What it means |
|---|---|
| Needs six objects to construct | Too many collaborators; the unit has too many reasons to change |
| Needs a database/network to do anything | Business logic is entangled with I/O |
| Needs `freezegun`, monkeypatched clock, or `time.sleep` | Time is an implicit dependency; inject it |
| Needs to reach into privates to set up | The public interface is not sufficient to express a valid state |
| Test setup is longer than the test | The unit's state space is too large |
| Test must assert on log output | The behaviour has no return value; consider one |

Treat each of these as a design signal first and a testing problem second. The refactoring
that makes the test easy is nearly always the one that makes the code better — this is the
central claim of *Growing Object-Oriented Software, Guided by Tests* and it holds up.

### 2.5 What TDD actually is, and its honest limits

The loop: red (write a failing test) → green (make it pass, minimally) → refactor (improve
structure with tests holding behaviour fixed).

What it reliably provides:

- Every line of code exists because a test demanded it. Coverage is a by-product, not a
  target.
- The design is exercised from the outside before it is committed to.
- A refactoring safety net is built *before* it is needed, when it is cheap.

What it does not provide, and where honest practitioners disagree:

- It does not find defects in things you did not think to test. It is a design discipline
  with a testing side effect, not a verification method.
- It is poor for exploratory work where you do not yet know the interface. Writing tests
  for an interface you will discard three times is waste. *Spike then stabilize*: explore
  freely, throw the spike away, rebuild test-first.
- It is poor for algorithms with a hard-to-state expected output (a layout engine, a
  numerical method, a compiler optimization). There, property-based testing (L04),
  golden/approval tests, and differential testing against a reference implementation are
  stronger tools.

State this clearly in your own practice. A methodology you cannot name the limits of is a
belief, not a technique.

### 2.6 Fixtures, and the cost of shared setup

`pytest` fixtures are dependency injection for tests. They are excellent and they are the
main way test suites become unreadable.

```python
@pytest.fixture
def account() -> Account:
    return Account(balance=Money(100, "USD"))

def test_withdrawal_reduces_balance(account: Account) -> None:
    account.withdraw(Money(30, "USD"))
    assert account.balance == Money(70, "USD")
```

Fine. The failure mode is the *mega-fixture*: a fixture that builds a whole world, used by
200 tests, where changing it breaks 40 of them and no test states its own preconditions.

Rules that keep fixtures useful:

- A fixture should build **one thing**, named after it.
- Data that a test's assertion depends on should appear **in the test**, not the fixture.
  If `test_withdrawal_reduces_balance` asserts `70`, the `100` should be visible in the
  test. Use a fixture that takes parameters, or a builder.
- Prefer **object mothers / builders** over fixtures for domain objects:
  `an_account().with_balance(100).build()`. Explicit, composable, greppable.
- Fixture scope (`function`, `class`, `module`, `session`) is a shared-mutable-state
  decision. `scope="session"` on anything mutable will eventually produce order-dependent
  failures. Use `pytest -p no:randomly` versus `pytest-randomly` to find them.

### 2.7 Flaky tests

A flaky test is worse than a failing test, and it is worth understanding exactly why.

A failing test carries information: something is broken. A flaky test carries *negative*
information: it trains the team to disregard red, which disables the entire suite's
function as a signal. One flaky test in a thousand, retried automatically, quietly
converts your CI from a gate into a decoration.

Causes, in order of frequency: shared state between tests; time (real clocks, timezones,
DST, leap seconds); ordering assumptions on unordered collections; concurrency and
`sleep`-based synchronization; real network calls; randomness without a fixed seed;
resource exhaustion (ports, file handles) under parallel execution.

Policy that works: **quarantine immediately, fix within a fixed window, delete if not
fixed.** A quarantined test is not evidence, and pretending otherwise is the actual
failure.

## 3. Construction: specifying a rate limiter

Build the test suite *first*, and watch it drive the design.

**Step 1 — the claims, in prose.** Before any code:

1. Within a window, the first N requests are allowed.
2. The N+1th request within the window is denied.
3. After the window elapses, requests are allowed again.
4. Limits are per-key: exhausting key A does not affect key B.
5. A denied request does not consume capacity. *(Is that true? Decide. It is a real design
   question — token bucket vs fixed window differ here.)*
6. Concurrent requests never allow more than N in a window.

Six claims. Notice that writing them exposed a decision (5) you had not made.

**Step 2 — the first test forces the interface.**

```python
def test_first_request_is_allowed():
    limiter = RateLimiter(limit=2, window=timedelta(seconds=60))
    assert limiter.allow("user-1") is True
```

This one test already commits you to: a constructor taking limit and window, a method
`allow` taking a key, and a boolean return. Consider alternatives before accepting: should
it raise instead of returning `False`? Should it return the retry-after delay? A boolean
throws away information the caller needs for a `Retry-After` header — so the test made you
notice a design flaw before you wrote the class.

Revise:

```python
def test_first_request_is_allowed():
    limiter = RateLimiter(limit=2, window=timedelta(seconds=60))
    assert limiter.check("user-1") == Decision(allowed=True, retry_after=None)
```

**Step 3 — claim 3 forces time to be injected.**

```python
def test_capacity_resets_after_window():
    clock = FakeClock(start=datetime(2026, 1, 1, tzinfo=UTC))
    limiter = RateLimiter(limit=1, window=timedelta(seconds=60), clock=clock)
    assert limiter.check("k").allowed
    assert not limiter.check("k").allowed
    clock.advance(timedelta(seconds=61))
    assert limiter.check("k").allowed
```

Writing this test without a `clock` parameter requires `time.sleep(61)` (unacceptable) or
monkeypatching `time.time` (works, but couples the test to the implementation's choice of
time source and breaks the moment you switch to `time.monotonic`). **Injecting the clock is
the design the test demanded**, and it is better independently: it makes time an explicit
dependency, which you will want again for testing expiry, retries, and timeouts.

Note that `FakeClock` is a **fake**, not a mock — a real working implementation with
different characteristics. L02 makes that distinction load-bearing.

**Step 4 — claim 6 is not a unit test.** Concurrency safety cannot be established by
example. Options: a stress test that runs 1,000 threads and asserts the invariant (finds
gross errors, proves nothing); a property test over interleavings; or a model check
(FM-751 L03). Write the stress test, and write in the docstring exactly what it does and
does not establish. Being explicit about the weakness of a test is a graduate-level habit
and almost nobody does it.

**Step 5 — now write the implementation.** It will be about 30 lines. Notice that you have
not yet decided between fixed-window, sliding-window, and token-bucket. Your six claims
under-determine the implementation, and claim 5 is the one that discriminates. Go back and
decide, then add the test that pins it.

## 4. Failure modes

- **Testing the implementation.** §2.1's diagnostic question.
- **Assertion-free tests.** A test that calls the code and asserts nothing "passes"
  forever. Grep for tests with no `assert`.
- **Tests that cannot fail.** `assert result is not None` after a function that cannot
  return `None`. Check by *mutating the code and confirming the test goes red* — this is
  mutation testing in miniature (L03), and doing it by hand once per new test is cheap.
- **Over-fixture-ing.** §2.6.
- **Conditional logic in tests.** An `if` in a test means it is two tests. A loop over cases
  means it is a parametrized test (`@pytest.mark.parametrize`), which reports each case
  separately.
- **Testing the framework.** Asserting that `dataclass` generates `__eq__` correctly is not
  your job.
- **Tolerating flakiness.** §2.7.
- **`assert` in production code as validation.** PY-501 L08 §2.4 — removed under `-O`.
- **Docstring tests as the primary suite.** `doctest` is documentation that is checked;
  it is poor as a test framework (weak assertions, output-format-sensitive). Use it for
  examples in docs and nothing more.

## 5. Exercises

### Warm-up (20 min)

**W1.** Take ten tests from a codebase you work on. For each, answer §2.1's diagnostic
question. Report the ratio of behaviour tests to implementation tests.

**W2.** Find a test whose name does not state a claim. Rewrite the name. Then check whether
the body still matches the name — often it does not, which is the interesting finding.

**W3.** Write a test that passes but cannot fail. Then write the mutation to the production
code that should have made it fail, and confirm it does not.

### Core (2 h)

**C1 — Specification-first rate limiter.** Complete §3, including the design decision at
step 5. Deliverable: the six claims as prose; the test suite; the implementation; a
300-word note on which tests drove which design decisions, and one decision the tests
*failed* to surface (there will be one — likely around memory growth for unbounded keys).

**C2 — Test suite triage.** Take a module with at least 20 tests. Classify each as
requirement / characterization / implementation. Delete or rewrite every implementation
test. Measure: lines of test code before and after, and whether coverage changed. Write
250 words on what you learned about the ratio.

**C3 — Fixture refactor.** Find the largest fixture in a codebase you have access to.
Replace it with builders. Report the diff in test readability by a concrete measure: for
three arbitrary tests, count how many lines you must read to understand the test's
preconditions, before and after.

**C4 — Flakiness hunt.** Run a real test suite 50 times with `pytest-randomly` and
`-p xdist` parallelism. Record every intermittent failure. For each, identify the shared
state. Write up a policy proposal for your team: quarantine criteria, fix window, and what
happens at expiry.

### Challenge

**X1.** Take a well-specified standard-library function (`bisect.insort`, `heapq.heappush`,
`str.partition`). Write its specification as a numbered list of claims, from the
documentation alone. Then write a test per claim. Then read the CPython implementation and
find at least one behaviour that is real, relied upon, and *not* in the documentation.
Write it up as a specification bug report.

**X2.** Read *Growing Object-Oriented Software, Guided by Tests*, chapters 1–5 and 20. Write
1,000 words on the "listen to the tests" thesis: state it precisely, give two examples from
your own code where it held, and one where you think it does not, with an argument.

## 6. Self-check

1. What is Dijkstra's claim about testing, stated precisely, and what follows for how you
   choose test inputs?
2. Give the diagnostic question that separates behaviour tests from implementation tests.
3. Why is one act per test important, and why is one assertion per test *not* a rule?
4. Name four kinds of test pain and the design defect each indicates.
5. State two things TDD provides and two things it does not.
6. Why is a flaky test worse than a failing one?
7. What is wrong with a fixture that contains the value a test asserts on?
8. How do you check that a new test is capable of failing?

## 7. Primary sources

- Dijkstra, "Notes on Structured Programming" (1970), §3 — the actual context of the quote.
- Freeman & Pryce, *GOOS*, chs. 1–5, 20.
- Winters et al., *Software Engineering at Google*, ch. 11 ("Testing Overview") — especially
  the section on test size and the Beyoncé Rule.
- Beck, *Test-Driven Development: By Example* — Part III's patterns, skim Parts I–II.

---

**Next:** [L02 — Test Doubles and the Boundaries They Imply](L02-test-doubles-and-boundaries.md)
