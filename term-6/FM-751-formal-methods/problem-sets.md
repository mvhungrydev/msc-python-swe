# FM-751 — Problem Sets

The three problem sets build formal evidence for the systems you have already constructed. Problem
Set 2 specifies and checks a protocol you wrote in DS-701; Problem Set 3 connects that specification
to the running code. Do them in order.

**A note on what counts as a result.** Finding a bug is a good outcome. Finding none and being able
to state precisely what you checked, under what assumptions, within what bounds, is an equally good
outcome. **Finding nothing because your model had no failure actions in it is not** — a
specification of the happy path cannot fail, and it is the most common way this work is wasted.
Every
deliverable below that says "report what you found" includes reporting an honest negative.

**A note on proofs by hand.** Problem Set 1 requires hand proofs before any tool. This is
deliberate: the tool does the mechanical part, and the skill it cannot supply is the invariant. An
engineer who can find the invariant is useful without any tool at all; one who can only run the tool
is stuck the moment it says "could not prove".

**A note on time.** TLA+'s learning curve is concentrated in the first six hours. Budget them
together, use Lamport's video course alongside L05, and expect the second specification to take a
quarter of the time the first did.

---

## Problem Set 1 — Specifications, Proofs, and a First Model Check
**Covers L01–L04 · Budget: 22–26 hours**

**Part A — The ambiguity hunt (L01).** Stages 1–2: five ambiguities in a real prose specification,
each with two readings and a scenario in which they differ observably; and one requirement expressed
at all seven rungs of the precision ladder, with a note of what became decided at each rung.

**Part B — Assumptions and guarantees (L01).** Stages 3, 8: the assumption/guarantee/interface
document for a component you have built, with the guarantee whose assumptions you had not written
identified; and the two-question adversarial review with what each question produced.

**Part C — Safety and liveness (L01, L03).** L01 Stage 4 and L03 Stage 2: twenty real requirements
classified, with conjunctions decomposed, plus a note of how many of the liveness ones your existing
tests address.

**Part D — State machines and traces (L01).** Stages 5–6: the explicit state machine for a
non-trivial component; five hand-written behaviours including two interleavings you had not
considered; and the tighten-then-verify-you-did-not-over-tighten exercise.

**Part E — Hoare logic by hand (L02).** Stages 1–2: five programs proved with every rule application
and every use of consequence written out; and ten standard algorithms with their loop invariants,
all three conditions verified, and their variants.

**Part F — Breaking and fixing (L02).** Stages 3–4: three subtle off-by-ones each localised to a
specific failing invariant condition, with the smallest exhibiting input and a comparison against
what your test suite would have caught; and binary search proved with the overflow precondition
analysed across languages.

**Part G — Representation invariants (L02).** Stages 5–6: three data structures from earlier terms
with their invariants stated and `_check_invariant()` implemented and exercised; and the concurrent
structure with its violation window identified and the critical section shown to cover exactly it —
including the deliberately narrowed version demonstrated failing.

**Part H — Temporal properties (L03).** Stages 1, 3–4: twenty English statements as temporal
formulas with satisfying and violating behaviours for five; all five Raft safety properties plus
liveness and fairness stated formally, with the fairness assumptions checked against your
implementation; and three safety counterexamples and two lassos written out by hand.

**Part I — Fairness and vacuity (L03).** Stages 5–6: mutual exclusion checked under no fairness,
weak fairness and strong fairness with the real scheduler's guarantee identified; and a vacuous
leads-to property constructed with the reachability check that catches it, adopted as a habit.

**Part J — A model checker (L04).** Stages 1–3, 5–6: the explicit-state safety checker; the state
explosion measured with your one-minute and one-hour configuration budgets; symmetry reduction with
the measured reduction factor; five buggy protocols with the written five-step counterexample
analysis for each; and the too-permissive model with the assumption you had failed to encode.

**Design note (1,500–2,000 words).** The ambiguity that most surprised you, and what it would have
cost downstream. The invariant you found hardest, and what made it hard. Which of your existing
systems has liveness requirements that nothing currently checks. And an honest account of what
proving five small programs by hand taught you that reading about Hoare logic had not.

---

## Problem Set 2 — A Distributed Protocol, Specified, Checked, and Refined
**Covers L05–L07 · Budget: 26–30 hours**

**Part A — TLA+ mechanics (L05).** Stages 1–3: the counter specification with all four deliberate
failure signatures recorded; binary search specified, checked, and broken with the counterexample
compared against your L02 invariant analysis; and the PlusCal lost-update demonstration with the
300-word write-up on what label placement asserts about your implementation.

**Part B — Mutual exclusion and the network (L05).** Stages 4–5: Peterson's algorithm with safety
and starvation-freedom checked, broken, diagnosed, and its fairness requirement identified; and the
reusable message-passing module with reordering and duplication verified genuinely possible.

**Part C — Two-phase commit (L05).** Stage 6: 2PC specified with agreement, validity and liveness;
the coordinator crash modelled; and **the blocking problem produced as a concrete counterexample**.
DS-701 L08 stated this as a theorem; here you produce the trace.

**Part D — The environment module (L06).** Stage 1: the complete failure model as TLA+ actions —
loss, duplication, reordering, crash with volatile/durable distinction, restart, and optionally
partition and heal — with each behaviour verified possible by a temporary refuting invariant.

**Part E — Raft, built up (L06).** Stages 2–5: the naive plurality election with its counterexample;
the Raft skeleton checked at two then three nodes; the voting rule broken two ways with both
counterexamples written up; and log replication with the log-consistency check broken and its
counterexample.

**Part F — The inductive invariant (L06).** Stage 6: leader completeness strengthened until
inductive, with **every conjunct recorded and justified**. This list is the required deliverable —
it is Raft's design rationale, recovered from first principles.

**Part G — Deriving the persistence requirements (L06).** Stage 7: `currentTerm`, `votedFor` and the
log each made volatile in turn, with the resulting safety violation found and explained. Report
which of DS-701's stated persistence requirements you derived and whether you derived any it does
not state.

**Part H — Liveness and bounds (L06).** Stages 8–9: fairness added and leader election liveness
checked; the split-vote livelock produced as a lasso by removing randomised timeouts; the state
space
measured against node count, term bound and log bound with symmetry applied; and **the honest
results
statement** in the required form.

**Part I — Refinement (L07).** Stages 1–4: the ten-line abstract specification; the refinement
mapping attempted, failing, and diagnosed as one of the three causes; and the intermediate layer
with the layered-versus-direct comparison.

**Part J — Auxiliary variables (L07).** Stages 5–6: a history variable with verification that it
does not constrain the protocol; and a prophecy variable for a lock-free structure whose
linearization point is determined later, with the demonstration that no state-only mapping works.

**Part K — Your own protocol (L06 Stage 10).** A protocol *you* designed — the saga engine, a CRDT
merge, or a state machine from your platform — specified, its properties stated, and checked. Report
what you found, and if nothing, report what writing the specification forced you to decide.

**Design note (1,500–2,000 words).** The counterexample that most changed your understanding of the
protocol. What deriving the persistence requirements experimentally taught you that the Raft paper's
prose had not. The refinement mapping's first failure and which of the three causes it was. And the
honest results statement, with a paragraph on what it does *not* establish.

---

## Problem Set 3 — Lightweight Verification Applied to Real Systems
**Covers L08–L10 · Budget: 22–26 hours**

**Part A — Properties and generators (L08).** Stages 1–3: thirty properties across ten components
with their patterns named and the bug-find rate reported; the generator coverage improvement
measured before and after; and the shrinking comparison including a custom shrinker.

**Part B — Model-based testing (L08).** Stage 4: your DI-721 LSM tested against a dict model over a
hundred thousand generated operation sequences. **Report the bug found** — and if none, report the
generator fix (interleaved deletes and compactions) that produced one.

**Part C — Algebraic laws (L08).** Stage 5: CRDT merge commutativity, associativity and idempotence
property-tested over random update sequences and delivery orders; strong eventual consistency
verified; and a subtle merge break caught.

**Part D — Specification-derived tests (L08).** Stages 6–7: your L06 specification's invariants
translated to runtime properties and its actions to a command generator with guards as
preconditions, with divergences reported; and **every model-checker counterexample from Problem Set
2 encoded as a directed implementation test**, with a report of how many the implementation actually
fails.

**Part E — Concurrency and fuzzing (L08).** Stages 8–9: concurrent linearizability testing with a
deliberate race caught and the run count to detection reported; and an hour of fuzzing a parser with
crashes and coverage reported, compared and then combined with a round-trip property test.

**Part F — Trace validation (L07 Stage 8).** Your DS-701 implementation instrumented to emit state
traces, validated against the L06 specification, run against your chaos suite. Report every
divergence, classified as implementation bug or specification inaccuracy, with the instrumentation's
performance cost.

**Part G — Z3 fundamentals (L09).** Stages 1–4: five puzzles solved with models and an `unsat` core;
the two validation checks established as a habit with the trivial-`unsat` demonstration; the binary
search overflow found with bitvectors and absent with the corrected expression; and three
equivalence checks with one distinguishing input found.

**Part H — Policy and reachability (L09).** Stages 5–6: access policies encoded and queried, with
any case where reading the policies gave the wrong answer reported; and your CA-731 VPC encoded as
packet constraints with a reachability question answered definitively, compared against the
configuration-level checker from CA-731 L04.

**Part I — Verification and generation (L09).** Stages 7–9: the wp verifier discharged by Z3 with a
correct-but-unprovable program explained; the configuration solver with `unsat` core diagnosis and
an optimisation objective; and the miniature symbolic executor with path explosion bounded and
coverage reported.

**Part J — Contracts and runtime verification (L10).** Stages 1–6: contracts on three components
with measured costs and a per-component production decision; the substitution rule violation
demonstrated and checked for in real code; every violation policy triggered and observed end to end;
the distributed invariant checker with its cost and false positive rate under normal churn; the
consistency checker with its detection latency; and three liveness properties rewritten as bounded
safety properties with the bounds connected to your SLOs.

**Part K — The portfolio and the claim (L10).** Stages 7–9: the technique portfolio table across the
whole term artifact with the highest-risk/weakest-evidence component identified; the retrospective
model check of a real incident with its timing and the one-page persuasion case; and **the evidence
statement**, one page, survived an adversarial reading by someone instructed to find a claim
stronger than the evidence supports.

**Design note (2,000–2,500 words).** The gap between your specification and your implementation, as
you actually measured it: what trace validation found, what specification-derived tests found, and
what each technique caught that the other did not. Then the course's central question: **for the
system you built, what do you know, how do you know it, and what remains unverified?** Write it as
you would for a colleague who will rely on your answer.

---

## Course position paper (1,500 words)

Choose one:

1. **"The largest share of formal specification's value is realised before any tool is run."**
   Defend or refute using your own experience in Problem Set 2 and the industrial reports.
2. **"Model checking is the highest-value engineering technique that almost no engineering team
   uses."** Argue it, and account for why adoption is low without dismissing the reasons.
3. **"Property-based testing derived from a specification is better value than either technique
   alone, and is the right default for most teams."** Use your Problem Set 3 measurements.
4. **"An engineer who cannot state a loop invariant cannot reason about code, whatever tools they
   have."** Defend or refute, with reference to L02.
5. **"'We proved it correct' is almost always an overclaim, and the habit of overclaiming has done
   more damage to formal methods than any technical limitation."** Argue it, and state what the
   defensible claim looks like.
6. **"SMT-based configuration and policy analysis will be adopted far more widely than model
   checking, for reasons that have nothing to do with rigour."** Defend or refute.

The structure is the one from `00-program/assessment-and-rubrics.md`: claim, grounds, the strongest
rebuttal you can construct, and the limits of your position. A paper that does not name a condition
under which its claim fails has not made a claim — which, in this course above all, is the point.

---

## Submission checklist (per set)

- [ ] Code in `courses/fm751/psN/`, runnable from a clean clone via the README.
- [ ] Tests passing, with a stated coverage figure *and* a sentence on what coverage does not tell
      you here.
- [ ] `mypy --strict` and `ruff` clean, or every exception documented with a reason.
- [ ] All measurements reproducible: every model-checking result states the configuration, the bounds, and the assumptions — an unqualified 'verified' is a fail.
- [ ] Charts and tables as files, not as descriptions — a plot referred to but not produced does not
      count.
- [ ] `NOTE.md` — the design note.
- [ ] `MARK.md` — self-marked against the five-criterion rubric with one line of justification per
      criterion.
- [ ] `log/failures.md` updated with everything you got wrong on the way, including the predictions
      that were incorrect.
