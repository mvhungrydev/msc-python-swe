# SE-511 · Lesson 04 — Property-Based Testing

**Estimated study time:** 4 hours
**Prerequisites:** L01–L03
**Tooling:** `hypothesis`

---

## 1. Orientation

Example-based tests encode the cases you thought of. Property-based tests encode a
*universal claim*, and let a machine search for a counterexample.

```python
from hypothesis import given, strategies as st

@given(st.lists(st.integers()))
def test_sorting_is_idempotent(xs: list[int]) -> None:
    assert sorted(sorted(xs)) == sorted(xs)
```

That is not one test; it is a theorem statement plus an attempt to refute it over hundreds
of generated inputs, with automatic minimization of any counterexample found. The shift in
thinking is from "what should this return for input X?" to "what is always true?" — which
is the same shift as from testing to specification, and it is why this technique is
disproportionately valuable at the graduate level.

## 2. Theory

### 2.1 The idea

QuickCheck (Claessen & Hughes, 2000): given a property `∀x. P(x)` and a *generator* of
values of `x`'s type, sample the input space, evaluate `P`, and report a counterexample if
found. Two additional ideas make it practical:

- **Shrinking.** A random counterexample is usually large and incomprehensible. Shrinking
  searches for a *minimal* failing input by repeatedly simplifying — a 400-element list
  becomes `[0, 0]`, a 30-character string becomes `"\x00"`. The minimal counterexample is
  usually diagnostic on sight.
- **Targeted generation.** Naive uniform sampling wastes effort. Good libraries bias
  towards boundary values (`0`, `1`, `-1`, `2**31`, empty, single-element, `NaN`,
  surrogates, duplicates) because that is where bugs live.

`hypothesis` adds a third: it **remembers failures** in `.hypothesis/examples` and replays
them first on subsequent runs, so a fixed bug stays fixed. Commit that database in CI (or
use `hypothesis` profiles with `derandomize=True` for determinism) — this is a real
decision with trade-offs, treated in §2.6.

### 2.2 Finding properties: the standard catalogue

The hard part is not the tooling; it is knowing what to assert. Six reliable patterns:

**1. Round-trip (there and back again).** The most productive property in existence.

```python
@given(st.text())
def test_encode_decode_roundtrip(s: str) -> None:
    assert decode(encode(s)) == s
```

Applies to: serialization, parsers/printers, compression, encryption, ORM save/load,
URL encoding. Almost every codebase has several and almost none test them this way.

**2. Invariance under transformation (metamorphic).**

```python
@given(st.lists(st.integers()))
def test_sort_is_permutation_invariant(xs: list[int]) -> None:
    assert sorted(xs) == sorted(random.sample(xs, len(xs)))
```

Applies when you cannot state the output but can state that some change to the input must
not (or must predictably) change it: `search(q)` results are a superset of
`search(q + " AND x")`; adding an ignored field does not change a hash; scaling all prices
by 2 doubles the total.

**3. Comparison against a model (oracle).** A slow, obviously-correct reference:

```python
@given(st.lists(st.integers()))
def test_fast_median_matches_naive(xs: list[int]) -> None:
    assume(xs)
    assert fast_median(xs) == sorted(xs)[len(xs) // 2]
```

Applies to: any optimization (test the fast path against the slow one), caches (test against
uncached), incremental algorithms (test against full recomputation). **This is the single
best use of property testing in industrial code** — you almost always have an obviously
correct slow version, or can write one in ten lines.

**4. Algebraic laws.** Commutativity, associativity, identity, idempotence, distributivity,
involution (`f(f(x)) == x`).

```python
@given(money(), money())
def test_addition_commutes(a: Money, b: Money) -> None:
    assume(a.currency == b.currency)
    assert a + b == b + a
```

If your type claims to be a monoid, a semigroup, or an ordering, those claims are testable
theorems. PY-501 L05's `Money` and L06's `Vector` are full of them.

**5. Invariants preserved.** After any sequence of operations, the data structure is still
valid: a balanced tree is still balanced, an account balance never goes negative, a queue's
length equals pushes minus pops. Combine with stateful testing (§2.4).

**6. Never crashes / always terminates.** The weakest property and still worth having for
parsers and anything taking untrusted input: `parse(s)` either returns a value or raises
`ParseError` — never `AttributeError`, never `RecursionError`, never hangs.

### 2.3 Strategies

`hypothesis` builds generators compositionally:

```python
import hypothesis.strategies as st

st.integers(min_value=0, max_value=100)
st.text(alphabet=st.characters(min_codepoint=32, max_codepoint=126), min_size=1)
st.lists(st.integers(), min_size=1, unique=True)
st.dictionaries(st.text(), st.integers(), max_size=10)
st.datetimes(timezones=st.just(UTC))
st.decimals(allow_nan=False, allow_infinity=False, places=2)
```

Composing domain objects — always prefer `st.builds` over hand-written composites:

```python
currencies = st.sampled_from(["USD", "EUR", "GBP"])
amounts = st.decimals(min_value=0, max_value=10**6, places=2)
money = st.builds(Money, amount=amounts, currency=currencies)
```

For dependent data, `@st.composite`:

```python
@st.composite
def order_with_lines(draw):
    currency = draw(currencies)
    lines = draw(st.lists(
        st.builds(Line, price=st.builds(Money, amounts, st.just(currency)),
                        qty=st.integers(1, 10)),
        min_size=1, max_size=5))
    return Order(lines=lines)
```

Note the pattern: draw the *constraining* value first, then use it to constrain the rest.
This is how you generate structurally valid data rather than filtering.

**`filter` vs `assume` vs constrained generation.** All three narrow the input space, and
they are not equivalent:

- `st.integers().filter(lambda x: x % 2 == 0)` — retries generation. Fine for high
  acceptance rates; `hypothesis` will error if too many are rejected.
- `assume(cond)` inside the test — discards the example. Same caveat.
- `st.integers().map(lambda x: x * 2)` — **generates** only valid values. Always preferable
  when expressible: no waste, no rejection limit, better shrinking.

A test that filters out 95% of examples is effectively running 5% of the iterations you
think it is.

### 2.4 Stateful testing

For objects with a lifecycle, generate *sequences of operations* and check invariants after
each:

```python
from hypothesis.stateful import RuleBasedStateMachine, rule, invariant, precondition

class AccountMachine(RuleBasedStateMachine):
    def __init__(self):
        super().__init__()
        self.account = Account(balance=Money(0, "USD"))
        self.model = Decimal(0)          # the oracle

    @rule(amount=st.decimals(min_value=0, max_value=1000, places=2))
    def deposit(self, amount):
        self.account.deposit(Money(amount, "USD"))
        self.model += amount

    @precondition(lambda self: self.model > 0)
    @rule(amount=st.decimals(min_value=0, max_value=1000, places=2))
    def withdraw(self, amount):
        try:
            self.account.withdraw(Money(amount, "USD"))
            self.model -= amount
        except InsufficientFunds:
            assume(amount > self.model)   # only acceptable if the model agrees

    @invariant()
    def balance_never_negative(self):
        assert self.account.balance.amount >= 0

    @invariant()
    def matches_model(self):
        assert self.account.balance.amount == self.model

TestAccount = AccountMachine.TestCase
```

This finds bugs that no example-based test will: the interleaving that produces an invalid
state on the fourth operation. It is the closest thing to model checking available in an
ordinary test suite (FM-751 L02 makes the connection explicit), and it is the right tool for
caches, state machines, connection pools, and anything with a protocol.

### 2.5 Reading a counterexample

`hypothesis` reports the *shrunk* input. Interpretation matters:

- `xs=[0, 0]` for a sorting bug means the bug involves **duplicates**, not values.
- `s=''` means the bug is about **emptiness**.
- `x=0` in arithmetic almost always means a division, a modulus, or a "first element"
  assumption.
- `s='\x00'` or `'\ud800'` (a lone surrogate) means an **encoding** assumption.
- A float `0.0` vs `-0.0` distinction means an **identity/sign** assumption.

Shrinking is doing analysis for you: the minimal counterexample names the category of the
bug. Read it before reading your code.

### 2.6 Determinism, CI, and the flakiness objection

The standard objection: "property tests are flaky — they fail randomly."

They are not flaky; they are *sampling*, and a test that fails on 1 in 500 inputs is
telling you about a real bug that your example tests miss entirely. But the operational
concern is legitimate: a PR failing on a bug unrelated to the change is disruptive.

The workable policy:

- **In PR CI:** a fixed profile — `derandomize=True` or a fixed seed, modest
  `max_examples` (50–100). Deterministic, fast, catches regressions in known-failing inputs
  via the example database.
- **Nightly:** a wide profile — `max_examples=1000`, no derandomization, longer deadlines.
  Failures here file a bug rather than block a merge.
- **Always commit the example database** (or use `@example(...)` decorators to pin
  discovered counterexamples explicitly). Pinning as an explicit `@example` is better: it
  is visible in the source, survives cache clearing, and documents the bug.

```python
@given(st.text())
@example("")                    # found by hypothesis, 2026-03-04, issue #212
@example("\ud800")
def test_encode_decode_roundtrip(s): ...
```

- **Set `deadline`** explicitly. `hypothesis`'s default per-example deadline causes
  spurious failures on loaded CI machines; either raise it or set `deadline=None` and rely
  on the overall test timeout.

## 3. Construction: properties for a real parser

Take a configuration parser (INI-like, or a small expression language).

**Step 1 — the round trip.** Immediately productive:

```python
@given(configs())
def test_parse_render_roundtrip(cfg: Config) -> None:
    assert parse(render(cfg)) == cfg
```

Writing `configs()` forces you to state what a valid `Config` *is* — which is a
specification exercise you probably skipped. Expect to discover that your model allows
things your parser cannot produce (keys with newlines, empty section names) and vice versa.
Every mismatch is either a generator bug or a real defect; work each one out.

Note the direction. `render(parse(s)) == s` is the *wrong* round trip: it is false for any
input with insignificant whitespace or comments. The correct forms are
`parse(render(x)) == x` (structure → text → structure) or the normalizing version
`render(parse(render(x))) == render(x)`.

**Step 2 — never crashes on arbitrary bytes.**

```python
@given(st.binary())
def test_parse_never_crashes(data: bytes) -> None:
    try:
        parse(data.decode("utf-8", errors="replace"))
    except ParseError:
        pass
```

Run this once and you will find a `KeyError`, an `IndexError`, or a `RecursionError` on
deeply nested input. This test alone justifies the lesson.

**Step 3 — the oracle.** If a reference parser exists (`configparser`, `tomllib`), compare:

```python
@given(st.text())
def test_matches_reference(s: str) -> None:
    mine = try_parse(s)
    theirs = try_parse_reference(s)
    assert (mine is None) == (theirs is None)
    if mine is not None:
        assert mine == theirs
```

This will fail, and the failures are the interesting part: each is a place where your
understanding of the format differs from the reference's. Some will be reference bugs.

**Step 4 — metamorphic properties.** Comments and blank lines do not change the parse.
Section order does not change lookups. Whitespace around `=` is insignificant. Each is one
line and each pins a real requirement.

**Step 5 — a stateful machine** for the mutable `Config` object: `set`, `delete`,
`merge`, `reload`, with the invariant that `get(k)` after `set(k, v)` returns `v` unless a
later `delete(k)` or `merge` overwrote it. Model it with a plain dict.

## 4. Failure modes

- **Restating the implementation as the property.** `assert f(x) == x * 2 + 1` where `f` is
  `lambda x: x * 2 + 1` proves nothing. The property must be independently justified.
- **Over-filtering.** §2.3. Check `hypothesis`'s health-check output.
- **Under-constrained generators.** Generating structurally invalid data and then asserting
  the code rejects it is a fine test — but it is not the test you meant if you wanted to
  exercise the happy path.
- **Properties that are only true "usually".** Floating-point associativity is *false*;
  asserting it produces a test that fails legitimately and teaches you to ignore it. Use
  `math.isclose` with a justified tolerance, or restrict the generator to values where the
  law holds exactly (e.g. small integers as floats).
- **Ignoring shrunk output.** The minimal counterexample is the diagnosis; read it first.
- **Wrong round-trip direction.** §3 step 1.
- **Not pinning discovered counterexamples.** They come back.
- **Using property tests only for pure functions.** Stateful testing (§2.4) is where the
  expensive bugs are.

## 5. Exercises

### Warm-up (25 min)

**W1.** Write three properties for `str.split`/`str.join` that together nearly specify
them. Find the input where the obvious round-trip fails, and explain.

**W2.** Write a property test that passes for the wrong reason (over-filtering). Show the
`hypothesis` health check that catches it.

**W3.** Take a shrunk counterexample from any failing property test and, *before* looking at
your code, write down what category of bug it indicates. Then check.

### Core (2.5 h)

**C1 — Six properties for one function.** Choose a non-trivial function from your own code.
Write one property of each of the six kinds in §2.2 (or argue convincingly why one kind does
not apply). Report which found bugs.

**C2 — The parser.** Complete §3 through step 5. Deliverable: the generators, the five
property groups, every counterexample found (pinned as `@example`), and a 500-word note on
what the round-trip property forced you to specify that you had not.

**C3 — Stateful testing on a cache.** Build an LRU cache and a `RuleBasedStateMachine` with
`get`, `put`, `evict`, `resize` rules, an `OrderedDict` model as the oracle, and invariants
(size ≤ capacity; every key in the cache was put and not evicted; eviction order is LRU).
Then plant three subtle bugs (off-by-one on capacity, wrong recency update on `get`,
non-atomic resize) and confirm each is caught, reporting the shrunk sequence.

**C4 — Retrofit.** Take an existing example-based test module and replace as many tests as
possible with properties. Measure: number of tests, lines of code, mutation score (L03) of
the module before and after. Report honestly — property tests do not always win, and finding
where they don't is the interesting result.

### Challenge

**X1.** Read Claessen & Hughes (2000) and MacIver's articles on `hypothesis`'s internal
representation (the "conjecture" byte-stream model). Write 1,000 words explaining how
`hypothesis` shrinks without knowing anything about your data types, and why that design
shrinks better than QuickCheck's type-directed approach.

**X2.** Apply property-based testing to a distributed component — a client with retries,
idempotency keys, and at-least-once delivery. State the properties (no duplicate side
effects under retry; every accepted request eventually appears exactly once). Then explain
what property testing *cannot* establish here, and what tool would (preview of FM-751).

## 6. Self-check

1. What are the two ideas beyond random generation that make property testing practical?
2. Name the six property patterns and give an example of each from your own domain.
3. Why is `map` preferable to `filter` in a strategy?
4. Which round-trip direction is correct for a parser, and why is the other one wrong?
5. What does a shrunk counterexample of `[0, 0]` tell you before you read any code?
6. Describe a workable CI policy for property tests and justify each element.
7. What does stateful testing find that ordinary property testing does not?
8. Give a property that is *false* for floats and explain what to do instead.

## 7. Primary sources

- Claessen & Hughes, "QuickCheck: A Lightweight Tool for Random Testing of Haskell
  Programs" (ICFP 2000). Ten pages; read it all.
- `hypothesis` documentation, particularly "What you can generate and how" and the stateful
  testing chapter.
- MacIver, "Anatomy of a Hypothesis Based Test" and the shrinking articles.
- Hughes, "QuickCheck Testing for Fun and Profit" (2007) — industrial experience.

---

**Previous:** [L03](L03-coverage-adequacy-mutation.md) · **Next:**
[L05 — Test Architecture: Levels, Contracts, and Cost](L05-test-architecture.md)
