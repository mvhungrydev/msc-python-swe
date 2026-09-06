# SE-511 · Lesson 10 — CI as a Design Constraint

**Estimated study time:** 3 hours
**Prerequisites:** L01–L09

---

## 1. Orientation

A team's CI pipeline takes 45 minutes. Consequences, none of which are about CI:

- Developers batch changes to avoid the wait, so pull requests get bigger, so review gets
  worse, so defects rise.
- A failing build blocks everyone, so people merge on red "just this once".
- Branches live longer, so merge conflicts grow, so people avoid refactoring.
- Nobody runs the full suite locally, so failures are discovered late.

The pipeline's duration has reshaped the team's design behaviour. **CI is not a
chore attached to the end of development; it is a constraint that propagates backwards into
architecture.** A system that cannot be built and tested in ten minutes will be a system
that is refactored less, released less, and understood less.

This lesson is about designing that constraint deliberately.

## 2. Theory

### 2.1 What continuous integration actually means

Not "we have a Jenkins server". The original meaning: **every developer integrates their
work into the shared mainline at least daily, and the mainline is always releasable.**

The two halves matter equally. Long-lived feature branches are *not* continuous
integration, regardless of how much automation runs on them — the integration risk is
merely deferred and concentrated. The empirical work (Forsgren, Humble & Kim, *Accelerate*,
and the DORA reports) consistently finds trunk-based development — branches living less
than a day, merged to a single mainline — among the strongest predictors of delivery
performance. Read that work critically: it is survey-based and the causal direction is
arguable. But the mechanism is plausible and the correlation is robust across years.

### 2.2 The feedback-latency budget

Design the pipeline as a series of gates with increasing cost and decreasing frequency:

| Stage | Budget | Contents |
|---|---|---|
| Editor / on save | < 1 s | formatter, linter, type check on the open file |
| Pre-commit hook | < 5 s | formatter, linter, type check on changed files |
| Pre-push / PR fast gate | < 5 min | small tests, full type check, lint, lockfile check |
| PR full gate | < 15 min | medium tests, contract tests, build, security scan |
| Post-merge | < 30 min | large tests, mutation testing on the diff, integration env deploy |
| Nightly | hours | full mutation testing, fuzzing, long property runs, dependency audit |

Two rules that make this work:

- **Anything in a gate must be deterministic.** A flaky check in a blocking gate is worse
  than no check (L01 §2.7). Non-deterministic checks belong in the nightly tier where they
  file issues rather than block merges.
- **Fail fast, and order by cost.** Lint before tests, unit before integration. A 12-minute
  test run that fails on a lint error at minute 11 is a waste that will happen every day.

**Ten minutes is the number that matters** for the PR gate. Beyond it, people stop waiting
and start context-switching, and the cost of the interruption exceeds the cost of the
build.

### 2.3 How CI shapes architecture

This is the part that makes it a *design* topic:

**Build time forces modularity.** If the whole system must be rebuilt and retested for any
change, build time grows with the system and eventually caps its size. Solutions —
incremental builds, test selection by changed-file impact, module-level caching — all
require the system to *have* modules with clear boundaries. Bazel-style build systems make
this explicit; `pytest --testmon`, `nx`, and dependency-graph-based test selection are the
lighter-weight versions.

**Deployability forces decoupling.** "Can this be deployed independently?" is the same
question as "does this have an independent contract?" A pipeline that must deploy six
services together in a fixed order has told you those six services are one system with
extra network hops (SE-521 L09, DS-701 L01).

**Test environment cost forces the port/adapter shape.** If your tests need a full
environment, you will have few of them. If your domain logic is testable in-process, you
will have many (L05 §2.4). The pipeline's economics select for the architecture.

**Rollback capability forces backward compatibility.** If you can roll back, you must
ensure version N-1 can read data written by version N. That constraint — expand/contract
migrations, additive schema changes, tolerant readers — is an architectural discipline
imposed by an operational requirement. DI-721 L09 covers it properly.

### 2.4 Merge strategy and change size

**Change size is the strongest predictor of review quality.** Review effectiveness drops
sharply above roughly 200–400 changed lines; the empirical work here (SmartBear's Cisco
study, and Google's internal data reported in *Software Engineering at Google*) is
consistent. Google's median change is on the order of tens of lines.

Mechanisms that keep changes small:

- **Feature flags** to merge incomplete work safely. The cost is flag debt; the discipline
  is a removal date recorded when the flag is created, and a periodic audit.
- **Branch by abstraction** for large refactors: introduce an abstraction over the old
  implementation, add the new one behind it, migrate callers incrementally, remove the old.
  Every step is mergeable.
- **Expand/contract (parallel change)** for interface changes: add the new form, migrate
  consumers, remove the old form. Three small PRs instead of one large breaking one.
- **Preparatory refactoring**: a separate, behaviour-preserving PR that makes the real
  change easy. Kent Beck's "make the change easy, then make the easy change."

### 2.5 What belongs in the pipeline

Beyond tests:

- **Formatter check** (`--check` mode; the fix happens locally).
- **Linter**, including your custom rules (L08).
- **Type check**, whole project, strict where ratcheted.
- **Lockfile freshness** (`uv sync --frozen`).
- **Vulnerability scan** (`pip-audit`) — as a *report* first, a *gate* only once the
  baseline is clean, or you will start with 40 findings and disable it.
- **Build the artifact**, and test *the built artifact*, not the source tree (L09 §4).
- **License check** if you distribute.
- **Coverage delta** on the diff (L03 §2.2), as information, not a gate.
- **Migration check**: does the migration apply cleanly to a copy of production schema, and
  is it reversible?
- **Performance smoke**: a benchmark with a generous threshold, to catch 10× regressions.
  Not a precise benchmark — CI machines are too noisy (PY-602 L01) — a *tripwire*.

And what does not: anything non-deterministic, anything requiring a human, anything that
takes longer than its tier's budget.

### 2.6 Pipeline as code, and its own testing

The pipeline is software. It deserves the same treatment:

- **Versioned with the code**, in the same repository, so a change to the build and the
  change that needs it land together.
- **Reusable components** (composite actions, shared workflows) rather than copy-paste
  across twelve repositories.
- **Pinned actions by commit SHA**, not by tag. A tag is mutable; a compromised action with
  write access to your CI is a full compromise. This is a real and recurring attack.
- **Least privilege.** Default `permissions: contents: read`; grant more per-job. Secrets
  scoped to the job that needs them, never available to workflows triggered by forks.
- **Testable locally.** `act`, or — better — a structure where every CI step is a script
  (`make test`, `just lint`) that a developer can run identically. A pipeline whose logic
  lives in YAML cannot be reproduced locally, and every failure becomes a
  push-and-pray cycle.

That last point is the highest-value one. **The YAML should orchestrate; the scripts should
do the work.**

### 2.7 Measuring the pipeline

Four numbers, tracked over time:

1. **p50 and p95 pipeline duration.** p95 is the one people actually experience.
2. **Failure rate by cause** — genuine defect, flake, infrastructure, timeout. If flakes
   are more than a few percent, fix that before anything else.
3. **Time from push to feedback** for the first failing signal.
4. **Queue time.** Often larger than run time, and invisible unless measured.

Then the DORA four, at the delivery level: deployment frequency, lead time for change,
change failure rate, time to restore. Use them as a diagnostic, not a target — they are as
Goodhart-able as coverage.

## 3. Construction: designing a pipeline

For your Term 1 build artifact.

**Step 1 — write the budget** (§2.2) before writing any YAML. Decide the tiers and their
time limits.

**Step 2 — make every step a script.**

```
scripts/
  fmt-check.sh     lint.sh     typecheck.sh
  test-small.sh    test-medium.sh
  build.sh         audit.sh
```

Each runnable locally, each exiting non-zero on failure. Now `make ci` runs the whole gate
on your machine.

**Step 3 — YAML that only orchestrates.**

```yaml
name: ci
on: [push, pull_request]
permissions:
  contents: read

jobs:
  fast:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@<sha>
      - uses: astral-sh/setup-uv@<sha>
        with: { enable-cache: true }
      - run: uv sync --frozen
      - run: ./scripts/fmt-check.sh
      - run: ./scripts/lint.sh
      - run: ./scripts/typecheck.sh
      - run: ./scripts/test-small.sh

  medium:
    needs: fast
    runs-on: ubuntu-latest
    services:
      postgres: { image: "postgres:16@sha256:<digest>", env: { POSTGRES_PASSWORD: x } }
    steps:
      - uses: actions/checkout@<sha>
      - uses: astral-sh/setup-uv@<sha>
      - run: uv sync --frozen
      - run: ./scripts/test-medium.sh
```

**Step 4 — cache correctly.** Cache the dependency install keyed on the lockfile hash.
Verify the cache is actually hit (print the key and status); an ineffective cache is the
most common invisible source of slow pipelines.

**Step 5 — measure and iterate.** Run it twenty times. Record p50/p95 per job. Find the
longest step. Ask whether it belongs in this tier.

**Step 6 — add a ratchet, not a target.** Type-ignore count, non-strict module count, and
lint-baseline size may only decrease.

## 4. Failure modes

- **A slow gate.** §1. The costs are architectural, not just annoying.
- **Flaky gates with automatic retry.** Converts a signal into noise silently.
- **Logic in YAML.** Unreproducible locally; every fix is a push.
- **Unpinned actions.** A supply-chain hole with full CI privileges.
- **Broad secret scope.** A workflow triggered by a fork PR with access to deploy
  credentials is a full compromise waiting to happen. Use `pull_request` (not
  `pull_request_target`) for untrusted contributions.
- **No local equivalent.** Developers cannot pre-verify, so the pipeline becomes the
  first place anything is checked.
- **Long-lived branches with heavy CI.** Automation on top of a non-continuous process.
- **Gating on a noisy metric.** Coverage percentage, benchmark timings on shared runners.
- **Not testing the built artifact.** L09 §4.
- **Deploying on green without a rollback path.** Green means "the tests we wrote passed",
  which is a much weaker claim than people act on.

## 5. Exercises

### Warm-up (20 min)

**W1.** Time your current pipeline by stage. Compute developer-hours per year spent waiting.

**W2.** Find a CI step whose logic lives only in YAML. Extract it to a script and run it
locally.

**W3.** Check whether your workflows pin actions by SHA. Report the count that do not.

### Core (2 h)

**C1 — Design and build the pipeline.** Complete §3 for your Term 1 artifact. Deliverable:
the budget document, the scripts, the workflows, the measured p50/p95 per job, and a
250-word note on the one thing you cut to stay within budget.

**C2 — The architecture argument.** For a system you work on, identify one architectural
property that its pipeline is currently *selecting against* (e.g. slow integration tests
discourage adding them, so integration risk is unmeasured; or a shared staging environment
forces coordinated deploys, so services stay coupled). Write 600 words: the mechanism, the
evidence, and what change to the pipeline would change the architecture.

**C3 — Change size.** Measure the distribution of PR sizes in a repository you have access
to (`git log --numstat`). Correlate size with review comment count, time to merge, and —
if you can get it — subsequent bug-fix commits touching the same lines. Report the
distribution and the correlations. Publish a negative result if you find one.

**C4 — Branch by abstraction.** Take a real refactor you have been avoiding because it is
too big. Plan it as a sequence of mergeable, behaviour-preserving steps. Execute at least
the first three. Report the plan and what changed about the difficulty.

### Challenge

**X1.** Read *Accelerate* (Forsgren, Humble & Kim) with a critical eye. Write 1,200 words
assessing its methodology: what exactly was measured, what is the sampling frame, what
causal claims are made versus supported, and which of its recommendations you would adopt
on mechanism alone even if the statistics were weak. Even-handedness is assessed.

**X2.** Implement test selection by impact: given a diff, run only the tests whose coverage
includes the changed lines (use `coverage.py` contexts). Measure the reduction in CI time
on real PRs and the number of times it would have *missed* a failure that the full suite
caught (replay historical PRs). Report the miss rate — that number decides whether the
technique is usable.

## 6. Self-check

1. What does "continuous integration" actually require, beyond automation?
2. Give the feedback-latency tiers and their budgets.
3. Name three ways CI economics shape architecture.
4. Why is change size the strongest predictor of review quality, and name three mechanisms
   for keeping changes small.
5. Why should CI logic live in scripts rather than YAML?
6. Why pin actions by SHA?
7. Which checks belong in a blocking gate and which do not?
8. Which four pipeline metrics would you track, and why is p95 more important than p50?

## 7. Primary sources

- Humble & Farley, *Continuous Delivery* (2010), chs. 3–5.
- Forsgren, Humble & Kim, *Accelerate* (2018) — read critically, per X1.
- Winters et al., *Software Engineering at Google*, chs. 22–24 (large-scale changes, CI, CD).
- Fowler, "ContinuousIntegration", "BranchByAbstraction", "ParallelChange" (bliki).
- OpenSSF, "Securing GitHub Actions" guidance.

---

**Previous:** [L09](L09-packaging-and-reproducibility.md) ·
**Course complete.** Next: [problem sets](problem-sets.md), [exam](exam.md), then
[Term 2](../../term-2/PY-502-advanced-abstraction/syllabus.md).
