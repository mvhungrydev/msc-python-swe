# SE-521 — Problem Sets

**A note on evidence.** An architectural claim needs grounds. "This decomposition is better" is an
opinion; "here are the six change scenarios, here is how many modules each touches under both
decompositions, and here is the one where the new design is worse" is an answer. Every part that
says *argue* means with named alternatives and a stated cost.

**A note on the constructions.** Each set builds on the lessons' §3 constructions. Where a part
names lesson stages, do those first.

**A note on what you give up.** Every part asking for a design also asks what the design costs. A
submission that lists only benefits cannot score in the upper band — that rule is the whole
discipline of this course in one sentence.

---

## Problem Set 1 — A Decomposition, Argued and Measured
**Covers L01–L04 · Budget: 12–16 hours**
*Builds on: L01 §3 (all stages), L02 §3 stages 1–7, L03 §3 stages 1–6, L04 §3 stages 1–5*

Take a real system, ideally one you did not write alone.

**Part A — Two decompositions (L01 §3).** The processing-step decomposition and the
information-hiding decomposition. Fifteen design decisions with a "how many places know
this today" column. Module secrets, one sentence each. A five-change comparison table
counting modules touched in each decomposition. The reader-trace cost of each.

**Part B — Measurement (L02 §3).** The import graph with Ca, Ce, I, A per package plotted
against the main sequence. The outliers. The modules-touched-per-commit distribution from
git. The top twenty co-changing file pairs. **The reconciliation**: where static and
historical coupling disagree, and what hidden coupling that reveals. This is the assessed
centrepiece.

**Part C — Connascence audit** of the worst boundary, by form, with a proposed change
reducing the strongest.

**Part D — One change made.** Break a cycle, remove control coupling, or inline a shallow
module. Measure before and after: total lines, tests needed for full path coverage, and the
reader trace.

**Part E — The pattern criteria (L04 §3).** Apply the dispatch-refactoring criteria table
to six conditionals in the system. Report how many you left alone.

**Design note (1,200–1,600 words).** What the measurements told you that reading did not.
The hidden coupling you found. Whether the information-hiding decomposition is worth its
indirection cost *for this system*, with the numbers.

### Marking emphasis

The change scenarios. A decomposition defended in the abstract scores poorly; one defended by
counting the modules each concrete change touches, including the change where your design is
worse, scores well.

---

## Problem Set 2 — A Domain Model with Explicit Boundaries
**Covers L05–L07 · Budget: 16–20 hours**
*Builds on: L05 §3 (all stages), L06 §3 stages 1–7, L07 §3 (all stages)*

This is the core of the Term 2 build artifact.

**Part A — The domain (L06 §3).** Glossary of 20–50 terms, reviewed by someone who knows
the business, with corrections marked. The event list. Value objects with their rules
(expect to eliminate a large amount of scattered validation). Entities with stated
invariants and an API that makes invalid states unrepresentable. The context map with
relationship types and owners, and a one-sentence justification of which context is the core
domain.

**Part B — The layering (L05 §3).** One feature end to end: pure domain, ports declared in
the inner layer, a use case doing orchestration only, adapters, in-memory fakes, contract
tests over both, and the composition root. Enforce with `import-linter`. Report per-tier
test runtimes.

**Part C — Consistency (L07 §3).** The invariant list with the *business consequence of a
brief violation* for each. Aggregate boundaries derived from it, with the contention
analysis. Optimistic concurrency implemented and a passing race test. The outbox implemented
end to end with an idempotent consumer and a reconciliation job. One saga with compensations
and failure tests at each step. The §2.7 consistency table for the whole system.

**Part D — The bug hunt.** Find and fix the external-call-inside-a-transaction (L05 §3
step 5). If your system does not have one, find one in an open-source project and write it
up.

**Design note (1,500–2,000 words).** Which level of §2.6 (L05) this system sits at and why.
The aggregate boundary decisions and the business conversations that determined them. Every
row of the consistency table whose mechanism was "nothing", and what you did. The honest
cost: files, mapping code, reader trace, compared with the same feature written directly in
a framework view.

### Marking emphasis

Boundary reasoning. The marks are in where you put the boundaries and why — specifically in the
invariant that forced each aggregate's size, and in the one you had to ask the business about.

---

## Problem Set 3 — Fitness Functions, Legacy Strategy, and an ADR Log
**Covers L08–L10 · Budget: 12–16 hours**
*Builds on: L08 §3 (all stages), L09 §3 stages 1–6, L10 §3 (all stages)*

**Part A — Fitness functions (L08 §3).** Three named architectural characteristics, ranked,
with what you are trading away. A layering contract with a baseline and a ratchet. Cycle
detection. Technology containment. One dynamic function. One production function
(a reconciliation job or an SLO with an alert). Every function linked to a rationale, with
failure messages that say what to do. The CI-time cost.

**Part B — Reversibility audit (L08 C2).** The ten most significant decisions in the system,
with the cost to reverse each today, and whether it was deferred to the last responsible
moment.

**Part C — A service-boundary analysis (L09 §3).** A real candidate split, all eight steps
with numbers, including the co-change rate, the crossing invariants with designed
compensations, the data migration plan, the availability arithmetic, and the
modular-monolith alternative. A defended recommendation — including "not yet", if that is
the answer.

**Part D — Legacy rescue (L10 §3).** The worst module by churn and test coverage:
characterization tests with a mutation score, seams introduced properly, one sprouted unit,
one branch-by-abstraction sequence as separate commits, and the patch-count metric before
and after.

**Part E — The ADR log.** Eight ADRs, at least four retroactive. Each with rejected options
and reasons, consequences including bad ones, and a revisit trigger. Reviewed by someone who
knows the system.

**Design note (1,200–1,600 words).** What the fitness functions caught in their first week.
The reversibility audit's most expensive decision and what would have made it cheaper. Your
service-boundary recommendation and the strongest case against it. What the ADR backfill
revealed that nobody could explain.

### Marking emphasis

Survivability. A fitness function that a team would disable within a month has failed regardless
of what it checks. Justify each one against the five design rules.

---

## Term 2 build artifact

A non-trivial domain model, assembled from PS2 and PS3: documented architecture, ADR log,
automated fitness functions, a test suite whose shape follows the architecture, and a
written argument for the boundaries. Not a CRUD app.

---

## Course position paper (1,500 words)

**Driving question: what distinguishes a structure that can absorb change from one that
cannot?**

Claim, grounds, rebuttal, limits. Your grounds must include measurements from PS1 Part B
(change coupling), PS2 Part D (the cost count), and PS3 Part A (what the fitness functions
caught). The rebuttal must state fairly the case that architecture is mostly post-hoc
rationalization — that systems survive because of the teams maintaining them rather than
their structure — and answer it.

---

## Submission checklist (per set)

- [ ] Artifacts in `courses/se521/psN/`.
- [ ] Every measurement reproducible: the script that produced it is committed.
- [ ] Diagrams as text where possible (Mermaid, or ASCII) so they diff.
- [ ] `NOTE.md` — the design note.
- [ ] `MARK.md` — self-marked with one line per criterion.
- [ ] `log/failures.md` updated.
