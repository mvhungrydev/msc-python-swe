# SE-511 — Written Examination

**Time allowed: 3 hours. Closed book. No interpreter, no notes, no search.**
**Answer FOUR questions from Section A and ONE from Section B.**
Section A: 15 marks each. Section B: 40 marks.

Mark one week later against `00-program/assessment-and-rubrics.md` §4.

---

## Section A — answer FOUR

**A1.** State Dijkstra's claim about testing precisely, and derive from it a principle for
choosing test inputs. Then give the diagnostic question that distinguishes a behaviour test
from an implementation test, and apply it to three tests you invent. *(15)*

**A2.** Define the five kinds of test double. State the command/query rule for choosing
between them and justify it. Then explain, with a concrete example, why "don't mock what you
don't own" follows from what a mock actually asserts, and describe the architectural shape
that follows from taking the rule seriously. *(15)*

**A3.** Rank statement, branch, condition, MC/DC, and path coverage by strength. Give a
program separating each adjacent pair. Then state precisely what coverage tells you and what
it does not, and explain why mutation testing measures something different. *(15)*

**A4.** Define mutation score. Explain equivalent mutants and why they cannot all be
detected (name the theoretical result). Then give three techniques for making mutation
testing viable in CI, with the trade-off each makes. *(15)*

**A5.** Name the six property-testing patterns from L04 and give an example of each for a
system you know. Then explain shrinking: what it does, why it makes property testing usable,
and what a shrunk counterexample of `[0, 0]` tells you before you read any code. *(15)*

**A6.** Define test *size* and *scope* and explain why they are independent. Give an example
of a small-size, wide-scope test and explain why it is often the best test available. Then
state the principle that resolves the pyramid-versus-trophy argument. *(15)*

**A7.** Define consistency in a gradual type system and prove it is not transitive. Explain
why `Any` and `object` are opposites. Then name four distinct sources of unsoundness in
Python's type system and give a program exploiting each. *(15)*

**A8.** State the rule that determines the variance of a generic type parameter, and derive
from it the variance of `Sequence[T]`, `list[T]`, `Callable[[T], R]`, and `Mapping[K, V]`.
Then explain the practical consequence for how you should annotate function parameters.
*(15)*

**A9.** State the rule for choosing between `typing.Protocol` and an abstract base class.
Explain what `@runtime_checkable` does and does not check. Then explain how defining a
Protocol in the consuming module changes the direction of a dependency, and why that
matters. *(15)*

**A10.** Explain why a static-analysis rule with a 10% false-positive rate has negative
value. Then design a project-specific check of your choosing: state the invariant, the
technique, the expected false-positive sources, the message text, and the rollout strategy.
*(15)*

**A11.** Name five things a lockfile does *not* guarantee. Explain dependency confusion and
the specific configuration that causes it. Then give the reproducibility ladder and argue
where a typical team should stop climbing. *(15)*

**A12.** Explain three distinct mechanisms by which CI pipeline duration propagates
backwards into system architecture. Then give the feedback-latency tiers with budgets, and
state the rule about what may and may not appear in a blocking gate. *(15)*

---

## Section B — answer ONE

**B1. (40)** You join a team with the following situation: 14,000 tests, 91% line coverage,
a 52-minute CI pipeline with a 6% flake rate and automatic retries, `mypy` configured but
with `ignore_errors = True` on 60% of modules, `requirements.txt` with unpinned transitive
dependencies, and roughly one production defect per week — mostly `AttributeError` and
`TypeError` on paths involving data from a third-party API.

Write a plan for the first ninety days. It must include:

- Your diagnosis: which of the observed facts are causes, which are symptoms, and what
  evidence would confirm each.
- What you would measure first, and why that measurement rather than another.
- The ordered sequence of changes, with the reasoning for the order.
- For each change: what it costs, who has to agree, and how you would know it worked.
- What you would deliberately *not* do in ninety days, and why.

Marks are for causal reasoning and sequencing. A plan that does everything at once scores
poorly; so does one that cannot say what evidence would falsify its diagnosis.

**B2. (40)** Design the complete verification strategy for a payments service: an HTTP API,
a Postgres database, a queue consumer, and a call to an external payment provider. Money is
involved; duplicate charges are unacceptable; the provider is occasionally slow and
occasionally lies.

Your answer must cover:

- The risk enumeration and the level at which each risk is observable.
- Test architecture with a written budget (tiers, counts, wall-clock).
- Where you use fakes, where mocks, where the real thing, with justification per boundary.
- What you verify with property-based testing and what properties specifically.
- What you verify at the type level, including how data from the provider is prevented from
  entering the domain as `Any`.
- What you cannot establish with tests at all, and what you would do instead
  (contract tests, formal methods, production monitoring, idempotence by construction).
- The pipeline that runs all of it inside a defensible feedback budget.

Marks are for coverage of the risk space and for honesty about the limits of each
instrument.

**B3. (40)** Argue for or against:

> "Static type annotations in Python are, for most teams, a net negative: the effort spent
> annotating, fighting the checker, and maintaining stubs exceeds the defects prevented,
> and the false confidence of a green check is itself a hazard."

Your answer must: state the strongest version of the position you oppose; give at least four
specific pieces of evidence, including at least one quantitative measure you know how to
obtain (mutation score, `Any` surface, defect categories); address the argument from
refactoring and the argument from tooling separately, since they have different strength;
and end with a falsifiable claim about what evidence would change your mind.

Even-handedness is assessed. An answer that does not state the opposing case fairly cannot
score above 24.

---

## Marking guidance

Section A, per question: 6 marks for the standard correct answer; 4 for precision and edge
cases; 3 for an example not from the lessons; 2 for a stated limitation of your own answer
or a connection to PY-501.

Section B: see the rubric. A correct, complete answer is 24/40. The remaining marks are for
judgement, sequencing, and stated uncertainty.
