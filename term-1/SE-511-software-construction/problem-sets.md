# SE-511 — Problem Sets

Three sets plus the Term 1 build artifact. Marked against the five-criterion rubric in
`00-program/assessment-and-rubrics.md`.

---

## Problem Set 1 — A Test Suite Rebuilt From First Principles
**Covers L01–L05 · Budget: 12–16 hours**

Take a module of real code with an existing test suite — yours, or an open-source project
you know. 300–800 lines of production code is the right size.

### Part A — Diagnose

Produce a table with one row per existing test:

| Test | Claim it encodes | Kind (requirement / characterization / implementation) | Size | Scope | Survives a behaviour-preserving refactor? |

Then report: the proportion of implementation tests, the branch coverage, and the mutation
score (L03).

### Part B — Rebuild

Delete the suite. Rewrite it, from the module's behaviour, applying:

- Claims as test names (L01 §2.2).
- Fakes at boundaries, contract-tested, with mocks only where the interaction *is* the
  requirement (L02).
- At least four property tests, using at least three of the six patterns (L04 §2.2).
- A stated size/scope tier for every test, mechanically enforced (L05 §2.1, C4).

### Part C — Compare

Report before/after on: test count, lines of test code, wall-clock runtime, branch
coverage, mutation score, and the number of tests that break under a scripted refactor
(rename a method, extract a class, change a private helper's signature).

### Design note (1,200–1,500 words)

- Which metric improved most, and is that the metric you care about?
- Where did the rebuild *lose* something? (It will. Old suites contain accidental
  regression coverage nobody remembers writing.)
- The mutation score gap: which surviving mutants did you accept, and why?
- If you had to keep only 20% of your new suite, which would it be, and what does that tell
  you about the other 80%?

---

## Problem Set 2 — A Fully Typed Library and a Custom Check
**Covers L06–L08 · Budget: 12–16 hours**

### Part A — Type it

Take an untyped or partially typed module of 400+ lines. Bring it to `mypy --strict` clean
and `pyright` clean under comparable settings.

Required to appear, each used because it is *right* and not to tick a box:

- A generic class or function with a bound `TypeVar`, with the variance justified.
- A `Protocol` defined in the consuming module for a dependency (L07 C2).
- An exhaustiveness check with `assert_never` over a closed union.
- `NewType` for at least two identifier types.
- A decorator typed with `ParamSpec`.
- At least one `@overload`, or a written argument for why the API should be split instead.
- Runtime validation at every trust boundary (L06 §2.7).

Report the `Any`-expression percentage before and after (`--any-exprs-report`).

### Part B — A custom check

Implement two project-specific static checks (L08 §3), with:

- Tests, including the tricky negatives.
- A measured false-positive rate on a real codebase, with every finding hand-reviewed.
- A message that tells the reader what to do instead.
- CI wiring with a baseline ratchet.

### Part C — Checker disagreement

Catalogue every place `mypy` and `pyright` disagree on your code. For each, consult the
typing specification and say which is right.

### Design note (1,000–1,400 words)

- The three places where the type was hard to write, and what design problem each revealed.
- Every `# type: ignore` you kept, with its justification.
- Three runtime errors your `--strict`-clean code still permits, classified by which
  unsoundness enabled them (L06 §2.2).
- Whether the effort was worth it, with your evidence. A negative answer, well argued, is
  a full-credit answer.

---

## Problem Set 3 — A Reproducible, Published, Continuously Integrated Package
**Covers L09–L10 · Budget: 10–14 hours**

This set is the Term 1 build artifact, assessed.

### Deliverables

1. **A published package** on PyPI (or a private index), with: `pyproject.toml` using
   PEP 621 metadata and PEP 735 dependency groups, a lockfile with hashes committed, a
   pinned interpreter, `py.typed`, a LICENSE, a CHANGELOG, and a documented public-API
   boundary.

2. **A `REPRODUCIBILITY.md`** stating exactly what is guaranteed and what is not — the
   ladder in L09 §2.4, with each rung implemented or explicitly declined with a reason,
   plus the two-build digest comparison and an account of the residual non-determinism.

3. **A pipeline** built to a written budget: tiers, time limits, every step as a locally
   runnable script, actions pinned by SHA, least-privilege permissions, and measured
   p50/p95 per job over at least twenty runs.

4. **A dependency audit**: every direct dependency with what it does, how much you use,
   maintenance signals, transitive count, and a removal recommendation for at least one.

5. **A supply-chain runbook** for a compromised-dependency incident, with the detection
   half actually tested against your current tooling.

### Design note (1,000–1,400 words)

- What you cut from the pipeline to stay inside the budget, and how you decided.
- One architectural property your pipeline currently selects for or against.
- The dependency you removed or would remove, and the trade.
- What in your reproducibility story you know to be a lie, and how much it matters.

---

## Course position paper (1,500 words)

**Driving question: what makes code trustworthy?**

Required structure (per `00-program/assessment-and-rubrics.md` §6): claim, grounds,
rebuttal, limits.

Your claim must be falsifiable and must be *yours*. Draw the grounds from your own
measurements in these three problem sets — the mutation scores, the `Any` surface, the
false-positive rates, the pipeline numbers. The rebuttal section must state, fairly, the
strongest case that the instruments you have invested in are theatre; then answer it.

Have one competent person read it and attack section 2.

---

## Submission checklist (per set)

- [ ] Code in `courses/se511/psN/`, runnable from a clean clone via the README.
- [ ] Tests passing; the tier constraints mechanically enforced.
- [ ] `mypy --strict` and `ruff` clean, or every exception documented.
- [ ] `NOTE.md` — the design note.
- [ ] `MARK.md` — self-marked against the rubric with one line of justification per
      criterion.
- [ ] `log/failures.md` updated.
