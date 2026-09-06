# SE-521 · Lesson 08 — Evolutionary Architecture and Fitness Functions

**Estimated study time:** 3.5 hours
**Prerequisites:** L01–L07; SE-511 L08, L10

---

## 1. Orientation

Every architecture decays. Not through incompetence — through a thousand individually
reasonable decisions, each of which slightly violated a constraint nobody wrote down. The
domain imports the ORM "just this once". A service reads another's table because the API
call was slow that day. A module's dependency count creeps from four to nineteen over three
years.

The response is not more documentation or more review. It is **executable constraints**: an
architecture whose important properties are checked by the build, so that violating them
fails like a broken test rather than being noticed in a design review eighteen months later.

Ford, Parsons & Kua call these **fitness functions**, borrowing from evolutionary computing:
an objective measure of how close a system is to its architectural goals.

## 2. Theory

### 2.1 Why architectures decay

Three mechanisms, and they are all structural rather than personal:

- **Constraints are implicit.** A rule that lives in one person's head, or in a wiki page
  from 2023, is not a constraint. Nobody violates it deliberately; people violate it because
  they do not know it exists.
- **The cheapest local action is often the wrong global one.** Reading another team's table
  is faster than getting them to add an endpoint. Under deadline, local wins.
- **Nobody owns "the architecture".** Everyone owns their feature. The property that emerges
  from the whole is nobody's job unless it is measured.

The response to each: **write the constraint down as code that runs.**

### 2.2 What a fitness function is

An objective, automated measure of an architectural characteristic. Examples across
categories:

| Characteristic | Fitness function |
|---|---|
| Layering | `import-linter` contract: no import from domain to infrastructure |
| Coupling | Cycle detection; max efferent coupling per package |
| Modularity | No package may import more than N others |
| Performance | p99 latency for the checkout path < 300 ms in a load test |
| Scalability | Throughput scales ≥0.8× linearly to 4 instances |
| Security | No high-severity CVE in the lockfile; no secret in the repo |
| Resilience | Chaos test: killing one instance keeps error rate < 0.1% |
| Data integrity | Reconciliation job reports zero cross-aggregate violations |
| Deployability | Build + deploy of a one-line change completes in < 15 min |
| Compatibility | Consumer contract tests pass against the new provider |
| Observability | Every endpoint emits a trace span with a request id |
| Cost | Monthly cloud spend per 1,000 requests < $X |

Classifications that help you choose:

- **Atomic** (one measure, one module) vs **holistic** (an emergent property of the whole
  system, e.g. latency under a realistic mixed workload).
- **Triggered** (runs in CI) vs **continuous** (runs in production as monitoring).
- **Static** (analysis) vs **dynamic** (measurement under load).

The insight worth taking away: **a production SLO is a fitness function**, and a
CI architecture check is a fitness function. They are the same idea, and thinking of them
together closes the gap between "architecture" and "operations" (CA-731 L07).

### 2.3 Designing one that survives

A fitness function that fails constantly is switched off, exactly like a lint rule with a
high false-positive rate (SE-511 L08 §2.1). The design rules:

- **Objective.** "Code should be maintainable" is not a fitness function. "No package
  imports more than 12 others" is.
- **Fast enough for its tier.** Static checks in the PR gate; load tests post-merge; cost
  and drift checks nightly (SE-511 L10 §2.2).
- **Introduced with a baseline and a ratchet.** Never "fix 400 violations first". Record the
  current number; fail if it increases; drive it down over time.
- **Owned, with a rationale.** Every fitness function links to the ADR that justifies it
  (L10). Without that link, the first person inconvenienced deletes it, correctly.
- **Failure message says what to do.** "Cycle detected between `orders` and `billing`; see
  ADR-014 — extract the shared concept or invert the dependency."
- **Reviewed.** A fitness function is a decision; decisions expire. Review the set once or
  twice a year and delete the ones that no longer reflect intent.

### 2.4 A worked set

For a layered Python application, the fitness functions worth having, roughly in order of
value per unit effort:

**1. Layering** (`import-linter`, seconds):

```ini
[importlinter]
root_package = myapp

[importlinter:contract:layers]
name = Layered architecture
type = layers
layers =
    myapp.entrypoints
    myapp.application
    myapp.domain

[importlinter:contract:domain-purity]
name = Domain imports no infrastructure
type = forbidden
source_modules = myapp.domain
forbidden_modules = sqlalchemy, httpx, redis, myapp.adapters, myapp.config
```

**2. No cycles.** A cycle means the modules involved are one module for change purposes
(L02 §4).

**3. Technology containment.** `psycopg` importable only from `myapp.adapters.postgres`;
the web framework only from `myapp.entrypoints`.

**4. Public API surface.** A test asserting the set of names exported from the package's
`__init__`, so that additions are deliberate (this is a Hyrum's Law defence, L01 §2.3).

**5. Contract tests** between services (SE-511 L05 §2.5).

**6. Performance tripwire.** Not a precise benchmark — CI is too noisy — a check that the
critical path has not regressed by more than 2×.

**7. Dependency freshness and vulnerability.** `pip-audit`, plus a check that no direct
dependency is more than N releases behind.

**8. Migration safety.** Every migration applies to a copy of the production schema and is
reversible, or is explicitly marked as not.

**9. Consistency reconciliation** (L07 §2.7) running in production and alerting on drift.

**10. Cost per unit of work**, tracked over time. Cheap to add, and it is the fitness
function most likely to get an architecture change approved.

### 2.5 Reversibility as the real goal

Fowler's reframing is the most useful thing in this area:

> **Architecture is the set of decisions that are hard to change. So make fewer decisions
> hard to change.**

If a decision is cheap to reverse, it is not architecture and does not need a heavyweight
process. If it is expensive to reverse, it deserves an ADR, a prototype, and a fitness
function to keep it true.

The corollary is a design objective: **increase reversibility.** Concretely:

- Prefer decisions you can undo in a week over ones that take a year, even at some cost in
  fit.
- **Defer the irreversible ones to the last responsible moment** — the point at which
  deferring further costs more than deciding now. This is not procrastination; it is buying
  information.
- Where you must commit, **isolate the commitment behind a boundary** (L05), so that
  reversal is localized.

The classic irreversible decisions, worth knowing: the primary datastore and its data model;
the service decomposition; the public API contract; the language and runtime; the
authentication and identity model; and anything with a data migration measured in months.

### 2.6 Guided evolution

A fitness function tells you when you have moved *away* from a goal. It does not tell you
how to move toward one. That requires:

- **A stated architectural characteristic** with a priority. You cannot maximize
  everything; "scalable, secure, maintainable, cheap, fast" is not a design. Pick the three
  that matter and say what you are trading away — Richards & Ford's point that architecture
  is "the science of trade-offs" is exactly this.
- **A direction of travel.** "We are moving from a shared database to per-service
  ownership." Written down, with the current position.
- **A ratchet per step.** The number of cross-service table reads may only decrease.

The failure this prevents is the migration that stalls at 60%, leaving two architectures and
the costs of both. That outcome is extremely common and it is almost always caused by having
no mechanism that makes the *old* pattern harder over time.

### 2.7 Architectural characteristics: choosing and trading

Name the characteristics explicitly. A usable shortlist: availability, performance
(latency vs throughput — different!), scalability, elasticity, security, deployability,
testability, evolvability, observability, cost, and simplicity.

Then rank. Three is the practical maximum for "we optimize for these". Everything else is a
constraint to satisfy, not a goal to maximize.

For each of the top three, write the fitness function. If you cannot write one, either the
characteristic is not really a priority or you have not defined it precisely enough — and
that ambiguity would otherwise surface in an argument two years from now about whether the
architecture "worked".

## 3. Construction: a fitness function suite

**Step 1 — name the characteristics.** Pick three for a real system and rank them. Write one
sentence on what you are deliberately trading away.

**Step 2 — write the layering contract.** `import-linter` for the layering you *intend*, not
the one you have. Run it. Record the violation count as a baseline.

**Step 3 — ratchet.** CI fails if the count increases. Commit the baseline file. Then fix
five violations and lower the baseline.

**Step 4 — technology containment and cycles.** Add both. Report what you find — most
codebases have at least one cycle nobody knew about.

**Step 5 — one dynamic function.** A performance tripwire or a resilience check. Make it
robust to CI noise (generous thresholds, medians of several runs, or run it post-merge
rather than in the PR gate).

**Step 6 — one production function.** A reconciliation job or an SLO with an alert. This is
the one that connects architecture to operations, and it is the one usually missing.

**Step 7 — link every function to a rationale.** An ADR or, at minimum, a comment with the
reason and the date. Then write the failure messages so that a developer hitting one knows
what to do.

**Step 8 — measure the cost.** How much CI time did you add? If it is more than a couple of
minutes in the PR gate, move something to a later tier.

## 4. Failure modes

- **Fitness functions with no baseline.** 400 violations on day one; everyone disables them.
- **Subjective "functions".** "Code should be clean."
- **No rationale.** Deleted at the first inconvenience, correctly.
- **Too slow for their tier.** Moved out of the gate, then ignored.
- **Noisy dynamic checks in a blocking gate.** Performance tests on shared CI runners.
- **Measuring what is easy rather than what matters.** Cyclomatic complexity thresholds are
  cheap to add and correlate weakly with anything.
- **No production fitness functions.** All the checks are static; the emergent properties go
  unmeasured.
- **Never reviewing them.** Constraints outlive their reasons.
- **Ratchets that only ever go one way in theory.** Someone raises the baseline "just for
  this PR"; require a reason and an approver.
- **Optimizing for characteristics you did not choose.**
- **A stalled migration with no ratchet making the old way harder.**

## 5. Exercises

### Warm-up (25 min)

**W1.** Write an `import-linter` contract for a real codebase and report the violations.

**W2.** Find a dependency cycle. Add a check that prevents new ones.

**W3.** For a system you know, name its top three architectural characteristics as currently
*revealed by decisions*, not as stated. They usually differ, and the gap is the finding.

### Core (2 h)

**C1 — The suite.** Complete §3, all eight steps. Deliverable: the functions, the baselines,
the ratchet mechanism, the rationale links, the failure messages, and the CI-time cost. This
is part of the Term 2 build artifact.

**C2 — Reversibility audit.** List the ten most significant decisions in a system you know.
For each: estimate the cost to reverse today, and whether it was deferred to the last
responsible moment or made early by default. Report the three most expensive and what would
have made them cheaper.

**C3 — A production fitness function.** Implement a reconciliation job for a cross-aggregate
invariant (L07 §2.7), with alerting and a runbook entry. Run it. Report what it found — in
most systems, it finds something.

**C4 — Stalled migration.** Find a migration in your organization that is partially
complete. Determine why it stalled. Design the ratchet that would have prevented it, and the
one that could restart it now. Present as a one-page proposal.

### Challenge

**X1.** Read Ford, Parsons & Kua, *Building Evolutionary Architectures*, and Richards &
Ford, *Fundamentals of Software Architecture*, chs. 4–7. Write 1,500 words applying the
characteristics-and-trade-offs framing to a real system: name the three characteristics its
architecture actually optimizes for (as revealed by decisions, not as claimed), the ones it
trades away, whether that is the right trade for the business today, and what you would
change.

**X2.** Build a fitness-function dashboard: every function's current value plotted over the
last year, computed by replaying git history. (Static checks can be run against historical
commits.) Report which characteristics improved and which decayed, and correlate with team
or process changes. This is a genuinely novel view of a codebase and is often the most
persuasive artifact you can bring to a technical leadership conversation.

## 6. Self-check

1. Give the three mechanisms by which architecture decays.
2. Define a fitness function and give the atomic/holistic and triggered/continuous
   distinctions.
3. Give five design rules for a fitness function that survives.
4. Name five fitness functions worth having in a layered Python application.
5. State Fowler's reframing of architecture and its corollary for design.
6. What is the "last responsible moment" and how does it differ from procrastination?
7. Why is three the practical maximum for optimized characteristics?
8. What mechanism prevents a migration stalling at 60%?

## 7. Primary sources

- Ford, Parsons & Kua, *Building Evolutionary Architectures*, 2nd ed.
- Richards & Ford, *Fundamentals of Software Architecture*, chs. 4–7 (characteristics and
  trade-offs).
- Fowler, "Who Needs an Architect?" (IEEE Software, 2003) — the source of the
  irreversibility definition.
- `import-linter` documentation.
- Beyer et al., *Site Reliability Engineering*, ch. 4 (SLOs) — production fitness functions.

---

**Previous:** [L07](L07-aggregates-and-consistency.md) · **Next:**
[L09 — Service Boundaries and the Distribution Decision](L09-service-boundaries.md)
