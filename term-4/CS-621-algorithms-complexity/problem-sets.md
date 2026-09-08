# CS-621 — Problem Sets

**A note on how these are marked.** In this course, a correct proof written badly scores
below a correct proof written well, and a wrong proof written confidently scores lowest of
all. Write proofs by hand first, then type them. State every assumption. Where you are
unsure, say so — an honest "I could not close this case" scores far above a hand-wave.

**A note on proofs.** Handwritten first, and a proof with a missing case is an incomplete proof, not
a proof with a small error. Where a part says *prove*, it means a proof an examiner would accept:
every case, every quantifier, and the base case actually stated.

**A note on the constructions.** Each set builds on the lessons' §3 constructions. Where a part
names lesson stages, do those first.

**A note on measurement.** Several parts ask you to compare an asymptotic prediction against a
measured runtime. Where they diverge — and they will — the divergence is the assessed content, not a
failure of the exercise. Asymptotics hide three things (L01 §2.3); find out which one bit you.

---

## Problem Set 1 — Analysis and Design, With Proofs
**Covers L01–L04 · Budget: 16–20 hours**
*Builds on: L01 §3 (all stages), L02 §3 stages 1–6, L03 §3 (all stages), L04 §3 (all stages)*

**Part A — Asymptotics (L01).** All eight items of Exercise A, proved from the definitions.
Items 3 and 8 require counterexamples. Plus the ordering exercise (W2) with justification for
the two you found hardest.

**Part B — Amortized analysis (L01).** All four of Exercise B with an explicit potential
function, the verification that Φ ≥ 0 and Φ(D₀) = 0, and — for the dynamic array with
shrinking — the adversarial sequence demonstrating why the half-empty shrink policy is Θ(n)
per operation, measured.

**Part C — The quadratic hunt (L01 C3).** Exercise C with measurements confirming each
analysis, plus one genuine accidental quadratic found in a real codebase, with the analysis,
the fix, and the measured improvement at realistic n.

**Part D — Recurrences (L02).** All eight of Exercise A, stating the method used and, for each
one where the Master Theorem does not apply, *why*.

**Part E — Divide and conquer (L02).** Inversion counting and maximum subarray (three ways),
with brute-force verification and measured growth. Plus the 300-word answer to "what
structural property does Kadane's algorithm exploit that divide and conquer does not?"

**Part F — Greedy (L03).** All eight items of Exercise A, each with a full proof
(stays-ahead or exchange) or a minimal counterexample. Plus the counterexample-searching tool
of Exercise C, run on three greedy criteria including one of your own invention.

**Part G — Dynamic programming (L04).** The five problems of Exercise A, each with the written
five-step derivation, four implementations, and cross-verification. Plus the exponential-to-
polynomial demonstration (Exercise C) with both plots.

**Design note (1,200–1,600 words).** Which proof technique you found hardest and why. The
accidental quadratic you found and how it survived. The greedy criterion of your own that
turned out to be wrong, and what the counterexample searcher revealed. The DP whose state
space was too large, and what you would do instead.

### Marking emphasis

Proof completeness. Every case, every quantifier, every base case. A proof sketch scores in the
lowest band regardless of how correct its idea is.

---

## Problem Set 2 — A Modelling Problem, Solved and Analysed
**Covers L05–L07 · Budget: 16–20 hours**
*Builds on: L05 §3 (all stages), L06 §3 stages 1–7, L07 §3 (all stages)*

**Part A — The graph toolkit (L05).** All eight algorithms of Exercise A, with iterative DFS,
`networkx` cross-verification on 1,000 random graphs each, and empirical complexity
confirmation.

**Part B — Modelling (L05).** All five problems of Exercise B: the five-step model, the
algorithm, the complexity, the implementation, and brute-force verification. Plus the 600-word
note on which model was hardest to see.

**Part C — Scale (L05 C3).** The memory and time comparison across representations and
libraries, the crossovers, the cache-miss measurement, and the vertex-reordering locality
result.

**Part D — Randomization (L06).** Exercise A's five proofs on paper. Exercise B's six
implementations with empirical verification of each probabilistic claim, including the
balls-in-bins versus power-of-two-choices comparison and the consistent-hashing load
distribution with and without virtual nodes.

**Part E — Adversarial inputs (L06 C3).** Both attacks constructed — sorted input to
deterministic quicksort, and colliding keys to a fixed-seed hash table — with the runtime
ratios and the 400-word note on hash randomization as a security feature.

**Part F — Sketches (L07).** All five sketches of Exercise A implemented and validated against
their theoretical guarantees. The merge verification of Exercise B including the *wrong*
quantile-averaging demonstration. The HLL intersection trap of Exercise C with the error curve.

**Part G — The metrics pipeline (L07 D).** Built, with the memory/latency/error comparison
against exact computation.

**Design note (1,500–2,000 words).** The modelling insight that was hardest to see, and what
made it click. The gap between the theoretical guarantee and the measured error for each
sketch — and any bug the validation found in your implementation. The error budget you would
set for a real metrics system, and how you would label the approximations so a consumer is not
misled.

### Marking emphasis

The modelling step. Recognising which classical problem you have is worth more than the
implementation. Show the reduction explicitly, in both directions where correctness requires it.

---

## Problem Set 3 — A Hardness Proof and a Coping Strategy
**Covers L08–L10 · Budget: 16–20 hours**
*Builds on: L08 §3 (all stages), L09 §3 stages 1–7, L10 §3 stages 1–5*

**Part A — Reductions (L08).** All five of Exercise A, on paper, with membership in NP shown,
f given, polynomiality argued, and *both* directions of correctness proved. Items 4 (3-SAT ≤
subset sum) and 5 (Hamiltonian cycle ≤ TSP) are the assessed ones.

**Part B — Recognition (L08 B).** Three real problems from your own work, precisely stated as
decision problems, classified against the recognition tables, with the reduction or the
polynomial algorithm given. For any that fit neither table, the written analysis of what makes
them different.

**Part C — The phase transition (L08 C).** The empirical study: time against n for brute force
and a SAT solver, the edge-density sweep, the located phase transition, and the 400-word note
on what it means for interpreting "NP-hard" as an engineering claim.

**Part D — The full coping study (L09 §3).** All nine steps on a real NP-hard problem:
instance characterization including structure measurement, ground truth, a solver encoding,
the parameter question, an approximation with its measured ratio, heuristics with measured
gaps, large-neighbourhood search, the comparison table with anytime plots, and the
recommendation with a monitoring design.

**Part E — Approximation ratios measured (L09 C2).** Three approximation algorithms, actual
ratio distributions over 1,000 instances against the proved bounds, plus a constructed
worst-case instance for each showing the bound is tight.

**Part F — Undecidability (L10).** Exercise A's five proofs. Exercise B's tool classification
with a constructed counterexample for each of five tools. Exercise C's termination analyzer
with its coverage statistics and both counterexamples. Exercise D's provably-terminating
configuration language with its termination proof.

**Design note (1,500–2,000 words).** The reduction you found hardest and where the backward
direction nearly failed. What the solver achieved versus what you expected — and whether, in
retrospect, the incumbent heuristic should be replaced. The gap between proved approximation
ratios and measured ones, and what that implies about how to read a theoretical guarantee.
What each of your five tools' green result actually licenses you to believe.

### Marking emphasis

The gap measurement. A heuristic shipped without a way to measure its optimality gap is the
failure this set exists to prevent. The four gap-measurement techniques are the assessed content.

---

## Term 4 build artifact (with CS-641)

The interpreter and type checker from CS-641, plus from this course:

- A written complexity analysis of its core algorithms (parsing, name resolution, type
  inference, evaluation), stating the model and the operation counted.
- One component where you chose a data structure or algorithm on the basis of a proved bound
  rather than a measurement, with the reasoning written down — and then a measurement
  checking it.
- A statement of which analyses your type checker performs are decidable, which are
  conservative approximations, and where it errs (L10 §2.6).

---

## Course position paper (1,500 words)

**Driving question: which problems are hard, and how would you know?**

Claim, grounds, rebuttal, limits. Grounds from your own work: the recognition exercise in
PS3 Part B, the phase-transition measurement, and the coping study's comparison table.

The rebuttal must state fairly the position that complexity theory is largely irrelevant to
working engineers — that worst-case asymptotics rarely predict real performance, that
NP-hardness rarely stops anyone because solvers work, and that the time spent on this course
would have been better spent on measurement — and then answer it. Note that this position is
partly right, and an answer that concedes precisely where it is right will score higher than
one that does not.

---

## Submission checklist (per set)

- [ ] **Proofs handwritten first**, then typed. Photograph or scan the handwritten versions
      and commit them; the working is part of the deliverable.
- [ ] Every algorithm implemented and verified against a brute-force reference on random
      inputs.
- [ ] Every complexity claim confirmed empirically at three input sizes.
- [ ] Code in `courses/cs621/psN/`.
- [ ] `NOTE.md` — the design note.
- [ ] `MARK.md` — self-marked, one line per criterion.
- [ ] `log/failures.md` — every proof you could not close and every prediction you got wrong.
      This course should produce the longest entry of the program.
