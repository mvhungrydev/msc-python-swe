# SE-511 · Lesson 03 — Coverage, Adequacy, and Mutation Testing

**Estimated study time:** 3.5 hours
**Prerequisites:** L01, L02

---

## 1. Orientation

```python
def discount(price: float, is_member: bool) -> float:
    if is_member:
        price = price * 0.9
    return round(price, 2)

def test_discount():
    assert discount(100.0, True) == 90.0
    assert discount(100.0, False) == 100.0
```

100% line coverage. 100% branch coverage. And the function is wrong for negative prices,
wrong for `NaN`, and the `round` uses banker's rounding so `discount(0.125, False)` returns
`0.12`, not `0.13` — a real defect in a money context that no coverage metric will ever
mention.

Coverage measures **what the tests executed**. Adequacy asks **what the tests would have
caught**. These are different questions, and only the second one matters.

## 2. Theory

### 2.1 Coverage criteria, from weakest

- **Statement (line) coverage** — every line executed. Weakest useful criterion.
- **Branch (decision) coverage** — every branch of every decision taken both ways. Strictly
  stronger: `if x: f()` with no `else` has 100% line coverage from one test but only 50%
  branch coverage.
- **Condition coverage** — every atomic boolean sub-expression evaluated both ways. Note
  this does *not* imply branch coverage: for `if a and b`, the pair (a=T,b=F), (a=F,b=T)
  gives full condition coverage while the branch is never taken.
- **MC/DC (modified condition/decision coverage)** — every condition shown to
  *independently* affect the outcome. Required by DO-178C for avionics software. For an
  n-condition decision it needs n+1 tests rather than 2ⁿ, which is what makes it practical.
- **Path coverage** — every execution path. Exponential in branches, infinite with loops.
  Not achievable in general.

`coverage.py` measures statement coverage by default and branch coverage with
`--branch`. **Always turn on branch coverage**; statement coverage alone routinely reports
95% for suites that never test a single `else`.

```toml
[tool.coverage.run]
branch = true
source = ["mypkg"]

[tool.coverage.report]
exclude_also = ["if TYPE_CHECKING:", "raise NotImplementedError", "@overload"]
```

### 2.2 Why coverage is a bad target

Coverage is a **necessary but wildly insufficient** condition. Uncovered code is definitely
untested; covered code may be equally untested. Three specific failure modes:

1. **Execution without assertion.** The opening example. A test that runs code and asserts
   something trivial produces coverage indistinguishable from a good test.
2. **Coverage of the wrong thing.** A test of `main()` that exercises everything transitively
   covers the whole codebase while asserting one output.
3. **Goodhart's law.** "When a measure becomes a target, it ceases to be a good measure."
   A coverage gate produces coverage-shaped tests: parametrized loops over inputs with weak
   assertions, `assert result is not None`, tests written after the fact to hit lines.

The standard defence — a coverage *floor* rather than a target, plus review — helps, but the
honest position is: **coverage tells you where you definitely have no evidence, and nothing
more.** Use it as a search tool, not a score.

There is one strong, non-obvious use: **coverage deltas on a diff.** "This PR adds 40 lines
and 0 covered lines" is a genuinely useful signal, far better than an absolute percentage.

### 2.3 Mutation testing: measuring adequacy directly

The idea (DeMillo, Lipton & Sayward, 1978): take the program, introduce a small change (a
**mutant**), run the test suite. If the suite fails, the mutant is **killed**. If it passes,
the mutant **survived** — and the surviving mutant is a concrete, executable demonstration
that your suite would not catch that bug.

**Mutation score** = killed / (total − equivalent). This is a direct measure of adequacy: it
answers "what would my tests catch?" rather than "what did my tests run?"

Typical mutation operators:

| Operator | Example |
|---|---|
| Arithmetic replacement | `a + b` → `a - b` |
| Relational replacement | `<` → `<=`, `>`, `==` |
| Boundary shift | `x < n` → `x < n + 1` |
| Logical replacement | `and` → `or` |
| Constant replacement | `0` → `1`, `""` → `"X"` |
| Negation | `if c` → `if not c` |
| Statement deletion | remove a line |
| Return replacement | `return x` → `return None` |

Applied to the opening example, a mutant changing `0.9` to `0.8` is killed; a mutant
changing `round(price, 2)` to `round(price, 3)` **survives**, telling you the rounding
behaviour is unspecified by your tests. That is precisely the defect coverage missed.

Tools: `mutmut` and `cosmic-ray` for Python. Both are slow — each mutant requires a test
run — which is the technique's real cost.

**Making it practical:**

- Run mutation testing on **changed files only**, in CI, on a schedule (nightly) rather
  than per-commit.
- Restrict to the **core domain** — the code where a silent wrong answer is expensive. Do
  not mutate glue code, CLI parsing, or logging.
- Use the test suite's own timing: run the fastest tests first and stop at the first
  failure (both tools support this).
- Treat **surviving mutants as a work queue**, not a score to maximize. Each survivor is a
  question: "would I care if this happened?" Sometimes the answer is no.

**Equivalent mutants** — mutants that change the code but not its behaviour (e.g. changing
a variable that is subsequently overwritten) — cannot be killed and must be excluded by
hand. Detecting them is undecidable in general (a corollary of Rice's theorem, CS-621 L10),
which is why mutation scores are never 100% and why the tools need a suppression mechanism.

### 2.4 What to test, and the risk argument

Neither coverage nor mutation score tells you *where to spend effort*. That is a risk
question:

**Expected cost of a defect = probability × impact.** Spend testing effort in proportion.

- **High impact, high probability:** money, auth, data destruction, anything irreversible,
  and any code with a history of defects (bug density is strongly autocorrelated — past
  defects predict future ones better than complexity metrics do). Test heavily; consider
  property-based (L04) and formal methods (FM-751).
- **High impact, low probability:** disaster recovery, failover, the retry path at attempt
  4. These are *never exercised in production until they matter*, so tests are the only
  evidence. Explicitly budget for them; they are the tests most often missing.
- **Low impact:** display formatting, log messages. A characterization test at most.

Complexity metrics (cyclomatic complexity, cognitive complexity) are useful as a *search*
heuristic — high-complexity functions with low mutation scores are the top of your queue —
but they are poor as targets for the same Goodhart reason.

### 2.5 Other adequacy signals

- **Fault injection / chaos.** Coverage of failure paths. DS-701 L10.
- **Differential testing.** Run two implementations on the same inputs and compare. Superb
  when a reference exists (a new parser vs the old one, your implementation vs the standard
  library's). Requires no oracle beyond agreement.
- **Metamorphic testing.** When you cannot state the expected output, state a *relation*
  between outputs: `sort(shuffle(xs)) == sort(xs)`, `search(q) ⊆ search(broader(q))`,
  `render(x)` and `render(x)` are identical. Powerful for ML systems (ML-741 L07) and
  anything without a ground truth.
- **Fuzzing.** `atheris` (libFuzzer for Python) for parsers and anything consuming untrusted
  bytes. Coverage-guided, and it finds classes of bug that example-based testing never will.

## 3. Construction: raising a real mutation score

Take a small module of your own — 100–200 lines of genuine logic, not glue.

**Step 1 — baseline.**

```bash
uv run pytest --cov=mypkg --cov-branch --cov-report=term-missing
uv run mutmut run --paths-to-mutate mypkg/core.py
uv run mutmut results
```

Record both numbers. A typical, honestly-written suite lands at 90–98% branch coverage and
**55–75% mutation score**. That gap is the lesson.

**Step 2 — triage the survivors.** For each surviving mutant, classify:

- **(a) Real gap** — the mutation is a bug you would care about. Write the test.
- **(b) Equivalent** — behaviour unchanged. Suppress with a comment and a reason.
- **(c) Don't care** — a real behaviour change you are indifferent to (a log message, a
  performance heuristic). Suppress with a reason.

Category (c) is where judgement lives, and *writing down the reason* is what turns this from
a metric-chasing exercise into a design conversation with yourself.

**Step 3 — the boundary mutants.** The most informative survivors are almost always
boundary shifts (`<` → `<=`). They reveal that your tests use values in the middle of
ranges rather than at edges. The fix is systematic: for every numeric or length condition,
test at `boundary−1`, `boundary`, `boundary+1`. This single habit typically moves mutation
score by 10–15 points.

**Step 4 — the deletion mutants.** A surviving statement-deletion mutant means a line has no
observable effect that any test checks. Either the line is dead (delete it) or your
assertions are incomplete. Both outcomes are wins.

**Step 5 — record the number and defend it.** In your README:

> Mutation score: 84% on `mypkg/core` (nightly, `mutmut`). The 31 surviving mutants are
> classified in `MUTANTS.md`: 19 equivalent, 9 in logging/formatting, 3 accepted risks
> documented with reasoning.

That paragraph is worth more than any coverage badge, and it is a distinguishing feature of
a professional package.

## 4. Failure modes

- **Coverage as a target.** §2.2.
- **Statement coverage without `--branch`.** Overstates by 10–20 points typically.
- **Excluding files to raise the number.** Trivially detectable in review and always a bad
  sign.
- **Running mutation testing on everything, every commit.** It will be turned off within a
  week. Scope it.
- **Chasing 100% mutation score.** Equivalent mutants make it unattainable and the last 15
  points cost more than they return.
- **Ignoring the low-probability/high-impact quadrant.** §2.4. The disaster-recovery path
  is untested in most systems, which is only discovered during a disaster.
- **Treating complexity metrics as targets.** Refactoring to lower cyclomatic complexity
  without changing what is tested is theatre.
- **`# pragma: no cover` without a reason.** Require a comment; grep for bare ones.

## 5. Exercises

### Warm-up (25 min)

**W1.** Write a function with 100% statement coverage and 50% branch coverage from one test.

**W2.** Write a decision where full condition coverage does not imply branch coverage.

**W3.** Take a well-tested function of your own and hand-mutate it five ways. Count how many
your suite kills.

### Core (2 h)

**C1 — Mutation score, honestly.** Complete §3 on a real module. Deliverable: before/after
coverage and mutation scores, `MUTANTS.md` with every survivor classified and justified, and
a 400-word note on the three most informative survivors and what they revealed about your
testing habits.

**C2 — MC/DC by hand.** Take a four-condition decision from real code. Construct the minimal
MC/DC test set (n+1 = 5 cases) and prove each condition independently affects the outcome.
Then explain, in 250 words, why avionics standards require MC/DC rather than branch or path
coverage, referring to the cost of each.

**C3 — Differential testing.** Pick a standard-library function with an obvious
reimplementation (`str.split`, `textwrap.wrap`, `urllib.parse.urlparse`). Write your own,
then a differential test generating random inputs and comparing. Report every discrepancy
and, for each, decide whether the standard library or your implementation is "right" — you
will find at least one case where the documentation does not settle it.

**C4 — The risk map.** For a system you work on, produce a two-axis map of modules by
defect probability (use `git log` to count past fixes per file) and impact (your judgement,
documented). Overlay current test investment. Report the mismatches and propose a
reallocation. This is a deliverable you can take to a real team.

### Challenge

**X1.** Read DeMillo, Lipton & Sayward (1978) and the "coupling effect" hypothesis it rests
on — that tests detecting simple faults also detect complex ones. Find the empirical
literature testing that hypothesis (Offutt's work is the entry point) and write 1,000 words
on how well it holds up, and what that implies for how much you should trust a mutation
score.

**X2.** Build a mutation-testing harness that runs only the tests that *cover* the mutated
line (use `coverage.py`'s per-test contexts). Measure the speedup on a real project versus
`mutmut`'s default. Report the engineering difficulties — there are several around test
isolation.

## 6. Self-check

1. Rank the coverage criteria by strength and give an example separating each adjacent pair.
2. Why does full condition coverage not imply branch coverage?
3. State precisely what coverage tells you and what it does not.
4. Define mutation score. What is an equivalent mutant and why can't we detect them all?
5. Give three ways to make mutation testing practical in CI.
6. What is the risk-based argument for where to test, and which quadrant is most commonly
   neglected?
7. When is metamorphic testing the right tool?
8. Why is coverage-delta-on-a-diff more useful than absolute coverage?

## 7. Primary sources

- DeMillo, Lipton & Sayward, "Hints on Test Data Selection: Help for the Practicing
  Programmer" (IEEE Computer, 1978).
- Chilenski & Miller, "Applicability of Modified Condition/Decision Coverage to Software
  Testing" (1994) — the MC/DC rationale.
- Winters et al., *Software Engineering at Google*, ch. 11's section on coverage; their
  position is refreshingly blunt.
- `coverage.py` and `mutmut` documentation.

---

**Previous:** [L02](L02-test-doubles-and-boundaries.md) · **Next:**
[L04 — Property-Based Testing](L04-property-based-testing.md)
