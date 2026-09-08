# SE-511 — Written Examination

**Time allowed: 3 hours. Closed book. No interpreter, no notes, no search.**
**Answer FOUR questions from Section A and ONE from Section B.**
Section A: 15 marks each. Section B: 40 marks.

Mark one week later against `00-program/assessment-and-rubrics.md` §4.

---

## Section A — answer FOUR

### Testing as specification (L01)

**A1.** State Dijkstra's claim about testing precisely, and derive from it a principle for choosing
test inputs. Then give the diagnostic question that distinguishes a behaviour test from an
implementation test, and apply it to three tests you invent. *(15)*

**A2.** Name four kinds of test pain and the design defect each indicates. Explain why one *act* per
test matters while one *assertion* per test is not a rule, and what is wrong with a fixture that
contains the value the test asserts on. *(15)*

**A3.** State two things TDD provides and two things it does not. Explain why a flaky test is worse
than a failing one — being specific about what it costs beyond the wasted reruns — and give the
procedure for checking that a newly written test is capable of failing. *(15)*

### Test doubles and boundaries (L02)

**A4.** Define the five kinds of test double and what each supports. State the command/query rule
for choosing between them and justify it from what each double actually asserts. *(15)*

**A5.** Explain, with a concrete example, why "don't mock what you don't own" follows from what a
mock asserts, and describe the architectural shape that follows from taking the rule seriously.
Then give the three costs of mocking, precisely. *(15)*

**A6.** Explain why `mock.patch` requires the *lookup* path rather than the definition path, and
give an example where getting this wrong produces a test that passes while patching nothing. Then
define a contract test for a port, state the problem it solves, and give three cases where a mock is
genuinely the right choice. *(15)*

### Coverage, adequacy, and mutation (L03)

**A7.** Rank the coverage criteria by strength and give an example separating each adjacent pair.
Explain why full condition coverage does not imply branch coverage, and state precisely what
coverage tells you and what it does not. *(15)*

**A8.** Define mutation score. Explain what an equivalent mutant is and why they cannot all be
detected — connecting your answer to a result from computability. Then give three ways to make
mutation testing practical in CI. *(15)*

**A9.** Give the risk-based argument for where to concentrate testing effort, name the quadrant most
commonly neglected, and say why. Then explain when metamorphic testing is the right tool, and why
coverage-delta-on-a-diff is more useful than an absolute coverage number. *(15)*

### Property-based testing (L04)

**A10.** Name the two ideas beyond random generation that make property-based testing practical, and
explain what each contributes. Then explain what a shrunk counterexample of `[0, 0]` tells you
before you have read any code. *(15)*

**A11.** Name the six property patterns and give an example of each from a domain you know. Explain
why `map` is preferable to `filter` in a strategy, and which round-trip direction is correct for a
parser and why the other is wrong. *(15)*

**A12.** Describe a workable CI policy for property-based tests and justify each element — the
example count, the seed handling, the failure database, and the time budget. Then state what
stateful testing finds that ordinary property testing does not, and give a property that is false
for floats with the correct alternative. *(15)*

### Test architecture (L05)

**A13.** Define test *size* and test *scope* as independent axes, and give an example of a
small-size/wide-scope test. Then state the principle that resolves the pyramid-versus-trophy
argument, rather than restating either position. *(15)*

**A14.** Name the four costs of a test and say which dominates over a system's lifetime. Explain
why diagnosis cost scales with scope and what follows for how a suite should be shaped. *(15)*

**A15.** Draw the port-and-adapter test shape and say which tier holds most of the tests and why.
State what consumer-driven contract testing guarantees and what it does not, give three rules for
keeping end-to-end tests useful, and explain why "we deliberately don't test X" is a sign of a
healthy test architecture rather than a gap. *(15)*

### Gradual typing (L06)

**A16.** Define consistency in the gradual typing sense and explain why it is not transitive. Then
distinguish `Any` from `object` and say when each is correct. *(15)*

**A17.** Give four sources of unsoundness in Python's type system, with a concrete program
exhibiting each. State what annotations buy and what they do not, and explain the `Sequence` for
parameters / `list` for returns rule. *(15)*

**A18.** Give the migration strategy for an untyped codebase and explain why a big-bang annotation
effort fails. Explain what `assert_never` does and which class of bug it prevents, and state where
runtime validation must live and how that relates to where `Any` appears in your codebase. *(15)*

### Generics, variance, and protocols (L07)

**A19.** State the substitution rule that defines subtyping. Give the input/output rule for variance
and use it to derive `Callable`'s variance in both its parameter and return positions. *(15)*

**A20.** Explain why `list[T]` is invariant while `Sequence[T]` is covariant, with a program that
demonstrates the unsoundness the invariance prevents. Then explain why splitting a repository
interface into read and write halves changes its variance, and what that buys. *(15)*

**A21.** State the rule for choosing between a `Protocol` and an ABC. Explain what
`@runtime_checkable` actually checks and what it does not, what problem `ParamSpec` solves and what
people did before it, and what `py.typed` does and what happens without it. *(15)*

### Static analysis (L08)

**A22.** Rank the static analysis techniques by power and cost, with an example of each. Explain why
a 10% false-positive rate kills a rule, and what that implies about how a rule should be introduced.
*(15)*

**A23.** Give three project-specific invariants that no off-the-shelf tool checks, and outline how
you would check one. Explain why the AST alone is insufficient for a naive-datetime rule and what is
additionally required. *(15)*

**A24.** Distinguish pattern-based security scanning from taint analysis, giving a vulnerability each
finds that the other misses. Explain what a ratchet is and why it works where a target does not,
what makes a good lint message, and why a formatter's value is not the style it produces. *(15)*

### Packaging and reproducibility (L09)

**A25.** State what PEPs 517, 518 and 621 each define, and how they fit together. Explain why
applications should pin and libraries should not, and what goes wrong in each direction if you get
it backwards. *(15)*

**A26.** Explain why dependency resolution is NP-complete and what Python's flat environment (one
version per package per environment) implies for resolvability. Then name five things a lockfile
does not guarantee. *(15)*

**A27.** Explain dependency confusion, the specific configuration that causes it, and the fix. State
why wheels reduce supply-chain risk compared with sdists, give the reproducibility ladder with the
rung at which a typical team should stop, and explain why you must test against an installed wheel
rather than the source tree. *(15)*

### CI as a design constraint (L10)

**A28.** State what continuous integration actually requires, beyond automation, and why the common
usage of the term describes something else. Give the feedback-latency tiers with their budgets and
justify the budgets. *(15)*

**A29.** Name three ways CI economics shape architecture, with a concrete architectural decision
driven by each. Explain why change size is the strongest predictor of review quality and name three
mechanisms for reducing it. *(15)*

**A30.** Explain why CI logic should live in scripts rather than in pipeline YAML, and why actions
should be pinned by SHA. State which checks belong in a blocking gate and which do not, and give the
four pipeline metrics you would track with the reason p95 matters more than p50. *(15)*

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

**B4. (40)** A team has adopted property-based testing enthusiastically. Six months later: the
property suite takes 40 minutes, it fails roughly twice a week on a different seed each time,
nobody investigates the failures because they "usually go away", and the generators have grown so
constrained that most of them produce a narrow band of similar inputs. Coverage attributable to the
property suite, measured, is 4% above what the example-based tests already reached.

Write the diagnosis and the remediation. Your answer must: identify what went wrong at each of the
three levels — the properties, the generators, and the CI policy; explain specifically how
over-constrained generators arise (name the pressure that produces them) and how you would detect
the problem quantitatively; explain what the intermittent failures actually indicate and why "it
goes away" is the most alarming detail in the description; state which of the six property patterns
this team is most likely missing given the symptoms; give the CI policy you would put in place, with
each element justified; and state what you would measure in three months to know whether the
remediation worked.

Marks are for the diagnosis. A remediation plan that does not first explain how a team arrives here
will repeat it.

**B5. (40)** You are asked to set the verification standard for a new team of eight engineers
building an internal platform. You have authority to mandate, and a strong prior that mandates
which are not obviously worthwhile get routed around.

Write the standard. It must cover: what is required before merge and what is advisory; the type
checking configuration and the migration path for code that does not yet meet it; the coverage
policy, stated in a way that cannot be gamed by the obvious means; whether and where mutation
testing runs; the property-based testing expectation; the packaging and pinning rules for both
applications and libraries; and the pipeline budget.

Then — the assessed part — for each requirement, state: the specific defect class it prevents, the
cost it imposes per change, and the evidence that would make you *remove* it. A standard with no
removal criteria is a ratchet in the wrong direction, and this course's whole argument is that every
instrument has a cost that must be paid back.

Finally, identify the single requirement you expect to be most resented, and write the argument you
would make for it — assuming your audience is competent, sceptical, and has been burned by process
before.

---

## Marking guidance

Section A, per question: 6 marks for the standard correct answer; 4 for precision and edge
cases; 3 for an example not from the lessons; 2 for a stated limitation of your own answer
or a connection to PY-501.

Section B: see the rubric. A correct, complete answer is 24/40. The remaining marks are for
judgement, sequencing, and stated uncertainty.
