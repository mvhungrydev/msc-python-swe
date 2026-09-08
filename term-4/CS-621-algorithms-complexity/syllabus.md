# CS-621 — Algorithms, Complexity, and Computability

**Term:** 4 · **Credits:** 15 · **Nominal hours:** 140
**Prerequisites:** none beyond `appendices/mathematical-notation.md`
**Co-requisite:** CS-641

---

## Driving question

> Which problems are hard, and how would you know?

This is the course with no immediate payoff and the largest effect on your ceiling. It is
also the one most often skipped by self-taught engineers, and the gap it leaves is
recognizable: an inability to tell the difference between "this is slow because my code is
bad" and "this is slow because the problem is genuinely hard", and an inability to recognize
a known problem in an unfamiliar disguise.

Three things this course gives you that nothing else does:

1. **A lower bound on effort.** Knowing a problem is NP-hard stops you from spending three
   weeks looking for an exact polynomial algorithm.
2. **Recognition.** Most practical problems are a known problem wearing different words. The
   scheduling problem your team is arguing about is bipartite matching, or it is bin packing,
   and which one it is determines everything.
3. **Design technique.** Divide and conquer, greedy exchange, dynamic programming, and
   randomization are *ways of thinking* that generate algorithms, not just a catalogue to
   look up.

## Learning outcomes

On completion you will be able to:

1. **Prove** asymptotic bounds from the definitions, and carry out amortized analysis by
   aggregate, accounting, and potential methods.
2. **Solve** recurrences by substitution, recursion tree, and the Master Theorem, and state
   when the Master Theorem does not apply.
3. **Design** algorithms by divide and conquer, greedy choice, dynamic programming, and
   randomization, and **prove them correct** — exchange arguments, loop invariants, and
   optimal substructure.
4. **Analyse** graph algorithms and choose the right one for a real modelling problem.
5. **Apply** probabilistic tools: expectation, linearity, Markov, Chebyshev, Chernoff, and
   the union bound.
6. **Reduce** one problem to another and prove NP-completeness.
7. **Choose** a coping strategy for a hard problem: approximation with a guarantee,
   parameterization, special structure, or heuristics with measurement.
8. **State and use** the core computability results: the halting problem, Rice's theorem,
   and their practical consequences for tooling.

## Lessons

| # | Title | Hours |
|---|---|---|
| L01 | Asymptotics, Models, and Amortized Analysis | 4 |
| L02 | Divide and Conquer, and Recurrences | 4 |
| L03 | Greedy Algorithms and Exchange Arguments | 4 |
| L04 | Dynamic Programming | 5 |
| L05 | Graph Algorithms and Modelling | 5 |
| L06 | Randomized Algorithms and Concentration | 4.5 |
| L07 | Hashing, Sketches, and Streaming | 4 |
| L08 | NP-Completeness and Reductions | 5 |
| L09 | Coping with Hardness | 4 |
| L10 | Computability, Halting, and Rice's Theorem | 4 |

## Assessment

| Component | Weight |
|---|---|
| Problem set 1 (L01–L04): analysis and design, with proofs | 20% |
| Problem set 2 (L05–L07): a modelling problem, solved and analysed | 20% |
| Problem set 3 (L08–L10): a hardness proof and a coping strategy | 20% |
| Written exam | 25% |
| Course position paper (1,500 words) | 15% |

## Term 4 build artifact (with CS-641)

An interpreter with a type checker for a small language of your own design (CS-641), plus —
from this course — a written complexity analysis of its core algorithms and one component
where you chose a data structure or algorithm on the basis of a proved bound rather than a
measurement.

## How to study this course

**Write the proofs by hand, on paper.** Not typed, not skimmed. The difference between
following a proof and producing one is the difference between recognizing a technique and
owning it, and only writing produces the second.

**Do not look up the answer.** A problem you struggled with for forty minutes and failed
teaches more than one you solved in five by recalling the trick. Budget the struggle; it is
the work.

**Implement after proving.** Every lesson has an implementation exercise, and the
implementation is the check on the proof: a proof of an algorithm you cannot implement is
usually a proof of something slightly different from what you thought.

**Expect this course to be slower than the others.** 140 hours is the honest estimate and it
assumes you find the mathematics unfamiliar. Budget it in your best hours, not your
leftovers.

## Required reading

- Kleinberg & Tardos, *Algorithm Design* (2005) — the primary text. Chapters 1–8, 11, 13.
  Its strength is teaching you to *design*, and its proofs are unusually readable.
- Sipser, *Introduction to the Theory of Computation*, 3rd ed. — chapters 3–5, 7. For L08
  and L10.
- Cormen, Leiserson, Rivest & Stein, *Introduction to Algorithms*, 4th ed. — as a reference.
- Turing (1936); Cook (1971); Karp (1972).

## Recommended

- Mitzenmacher & Upfal, *Probability and Computing*, 2nd ed. — for L06–L07.
- Arora & Barak, *Computational Complexity: A Modern Approach* — where to go next.
- Erickson, *Algorithms* (free online) — excellent alternative explanations, especially for
  dynamic programming and recurrences.
- Skiena, *The Algorithm Design Manual*, 3rd ed. — the "war stories" and the problem
  catalogue in part II are genuinely useful for recognition.
