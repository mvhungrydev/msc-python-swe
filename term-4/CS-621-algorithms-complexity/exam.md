# CS-621 — Written Examination

**Time allowed: 3 hours. Closed book, no calculator, no interpreter.**
**Answer FOUR from Section A and ONE from Section B.**
Section A: 15 marks each. Section B: 40 marks.

Proofs must be complete. State every assumption. Where a proof has cases, do all of them.

---

## Section A — answer FOUR

### Asymptotics, models, and amortized analysis (L01)

**A1.** State the definitions of O, Ω, Θ and o precisely, and give the quantifier difference between
O and o. Prove `3n² + 5n + 2 = O(n²)` and prove `n log n ≠ O(n)`. *(15)*

**A2.** Name three things asymptotic analysis hides, with a concrete case where each one dominates
in practice. Then name four models of computation and one result that depends on each. *(15)*

**A3.** Prove the comparison-sorting lower bound and state its two assumptions. Then distinguish
amortized from average-case analysis precisely, give the potential method's definition, and give the
heuristic for choosing Φ. *(15)*

### Divide and conquer (L02)

**A4.** Give the divide-and-conquer recurrence form and say what a, b and f each represent. For each
of the three Master Theorem cases, give a recurrence and say whether the root, the leaves, or all
levels dominate. *(15)*

**A5.** State the regularity condition and explain why case 3 requires it. Give three recurrences
the Master Theorem cannot solve and the method appropriate to each. *(15)*

**A6.** Explain Karatsuba's trick and derive the resulting exponent. Explain what "binary search on
the answer" is and the two conditions it requires, what property a combine step must have to
parallelise and why, and why real quicksort implementations recurse on the smaller partition. *(15)*

### Greedy algorithms (L03)

**A7.** Give the greedy pattern and say where the entire design decision lies. State the shape of a
"greedy stays ahead" argument and apply it in full to interval scheduling. *(15)*

**A8.** State the shape of an exchange argument and apply it to shortest-job-first scheduling.
Explain why ratio-greedy is optimal for fractional knapsack and fails for 0/1, identifying precisely
where the argument breaks. *(15)*

**A9.** State the three matroid axioms and the intuition the exchange axiom captures. State the cut
property and what it proves. Explain why Dijkstra requires non-negative weights, identifying the
exact step of the proof that fails, and give four acceptable uses of a greedy algorithm and the one
unacceptable one. *(15)*

### Dynamic programming (L04)

**A10.** State the two conditions a problem must satisfy for dynamic programming and give a problem
lacking each. Give the five-step design procedure and say at which step most failures occur. *(15)*

**A11.** Give the complexity formula for a DP in terms of states and transitions, and apply it to
matrix chain multiplication. Compare memoization and tabulation across five dimensions and say which
you would write first and why. *(15)*

**A12.** Explain the knapsack rolling-array iteration order and state precisely what the wrong
direction computes instead. Define pseudo-polynomial and explain why knapsack's Θ(nW) is not
polynomial. Then state what Hirschberg's algorithm achieves and how, and give five signals that a
problem is a DP problem and three that it is not. *(15)*

### Graph algorithms (L05)

**A13.** Give four graph representations with their space and query costs, and say which is the
right default and why. State what a back edge in a DFS tells you and name three algorithms built on
the DFS edge classification. *(15)*

**A14.** Give the appropriate shortest-path algorithm for each of: unweighted graphs, non-negative
weights, negative weights, and all-pairs — with the complexity and the reason for each choice. State
the complexity of Union–Find with both optimisations and give three uses. *(15)*

**A15.** Define the condensation of a directed graph and state what it always is. State the max-flow
min-cut theorem and give the bipartite matching reduction. Then give the five modelling steps for
recognising a graph problem, and explain why graph algorithms are cache-hostile and what can be done
about it. *(15)*

### Randomized algorithms (L06)

**A16.** Distinguish Las Vegas from Monte Carlo algorithms and say how to convert between them in
each direction. State linearity of expectation and explain why the "regardless of dependence" clause
is the part that does the work. *(15)*

**A17.** State the Markov, Chebyshev and Chernoff bounds, and say what each assumes and what each
gives. Reproduce the key insight of the randomized quicksort analysis — the condition under which
two elements are ever compared. *(15)*

**A18.** State the balls-in-bins maximum load result and the power-of-two-choices improvement, and
say why the improvement is so large. State what universal hashing guarantees and which attack it
prevents, give the Bloom filter's properties and its bits-per-element for a 1% false-positive rate,
and explain why the sample size for a given error does not depend on the population size. *(15)*

### Sketches and streaming (L07)

**A19.** State the streaming model's constraints and the fundamental lower bound for exact distinct
counting. Explain HyperLogLog's estimator, its two refinements, and its standard error formula.
*(15)*

**A20.** Explain why HyperLogLog's relative accuracy is independent of cardinality and why that
property matters operationally. State count-min sketch's guarantee and explain why it is useless for
rare items. *(15)*

**A21.** Explain why percentiles cannot be averaged and give the correct alternative. State what
MinHash estimates and what LSH adds. Then give five design questions to ask before choosing a
sketch, and five situations where a sketch is the wrong choice. *(15)*

### NP-completeness (L08)

**A22.** Give the verifier definition of NP and explain why it is the more useful of the two
standard definitions. Distinguish NP-hard from NP-complete and give a problem that is one but not
the other. *(15)*

**A23.** State the reduction direction for proving hardness and explain the mnemonic that prevents
getting it backwards. Give the five steps of an NP-completeness proof and say which is most often
botched. *(15)*

**A24.** State the Cook–Levin theorem and its proof idea in one sentence. Give the 3-SAT ≤ₚ
Independent Set reduction with both directions of the correctness argument. Then give five things
NP-completeness does *not* mean, and distinguish weak from strong NP-completeness with why it
matters. *(15)*

### Coping with hardness (L09)

**A25.** Give the five coping strategies in order and state the question that comes before all of
them. Give the vertex cover 2-approximation with its proof, and name the general proof technique it
exemplifies. *(15)*

**A26.** Give the inapproximability hierarchy with an example problem at each level. Define
fixed-parameter tractability and contrast `O(2^k · n)` with `O(n^k)` numerically at n = 10⁶ and
k = 10. *(15)*

**A27.** Name five kinds of structure that make NP-hard problems tractable in practice. Give four
ways to measure a heuristic's optimality gap without knowing OPT, explain why an exact solver with a
time limit is often better than a heuristic, and say what monitoring a shipped heuristic needs and
why. *(15)*

### Computability (L10)

**A28.** Give the counting argument that most functions are uncomputable. Write out the halting
problem proof in full. *(15)*

**A29.** State the Church–Turing thesis and explain why it is not a theorem. State Rice's theorem
and give three undecidable and three decidable program properties. *(15)*

**A30.** Give the five coping strategies for undecidability with a real tool exemplifying each.
Explain why "halts within 1,000 steps" is decidable and what that licenses in practice, distinguish
undecidable from NP-hard, and explain why liveness is harder to verify than safety in terms of the
arithmetical hierarchy. *(15)*

---

## Section B — answer ONE

**B1. (40)** A logistics team asks you to solve this: given n delivery stops with time
windows, m vehicles with capacities, and a distance matrix, produce routes minimizing total
distance while respecting windows and capacities. They currently use a hand-tuned greedy
heuristic. n is typically 200–2,000; the answer is needed within 30 seconds.

Write the analysis and recommendation. It must include:

- Precise statement of the decision version, and its complexity class with justification
  (a reduction sketch suffices).
- What you would measure about the *instances* before deciding anything, and why each
  measurement could change the answer.
- The five coping strategies applied to this problem, with what each would give.
- Whether an exact solver is viable at n = 200 and at n = 2,000, and how you would find out.
- How you would measure the current heuristic's gap without knowing the optimum.
- Your recommendation, with the guarantee it provides.
- The monitoring you would put in place, and what it would detect.
- What you would tell the team about the request "can we just get the optimal answer?"

Marks are for the complexity reasoning, the measurement plan, and the honesty of the
guarantee you claim.

**B2. (40)** Design the algorithmic core of a real-time analytics system: ingesting 500,000
events/second, answering queries over arbitrary time ranges for distinct users, event counts,
top-K by several dimensions, and latency percentiles — with a memory budget of 64 GB across
the fleet and a query latency target of 200 ms.

Your answer must cover:

- Why exact computation is infeasible, with the space arithmetic.
- The data structure for each of the four query types, with its guarantee (ε, δ) and its
  space.
- The mergeability requirement and why it drives every choice.
- What each sketch cannot do, and which plausible user question you would therefore have to
  refuse — be specific.
- The error budget: what ε you would choose for each statistic and how you would justify it
  to the person consuming the numbers.
- How you would validate the implementations against their theoretical guarantees.
- How the results should be labelled so that a consumer is not misled.
- One place where you would keep exact computation despite the cost, and why.

**B3. (40)** Argue for or against:

> "Complexity theory is largely irrelevant to working software engineers. Worst-case
> asymptotics rarely predict real performance, since constants and cache behaviour dominate
> at realistic sizes. NP-hardness rarely stops anyone, because modern solvers handle
> industrial instances routinely. And undecidability results describe limits nobody was going
> to reach. The time spent learning this would be better spent learning to measure."

Required: state the opposing position at its strongest; give at least four specific pieces of
evidence, drawn from this course, including at least one where the theory made a *decision*
that measurement alone would not have; address the three claims (asymptotics, NP-hardness,
undecidability) separately, since they have different strengths; concede clearly where the
position is right; and end with a falsifiable claim about what evidence would change your
mind.

Even-handedness is assessed, and this position is partly correct. An answer that treats it as
obviously wrong cannot score above 24.

---

**B4. (40)** A colleague brings you this problem: given a set of 8,000 configuration items, each
with a size and a value, and a set of 40 machines each with a capacity, assign items to machines to
maximise total value. They have written a greedy algorithm (sort by value/size, place each item on
the first machine that fits) and want to know whether it is "good enough". They cannot say what
"good enough" means and have not measured anything.

Write the analysis. Your answer must: identify the problem, state its complexity class with a
justification rather than an assertion, and name the closest textbook problem; explain what the
greedy algorithm's worst case is, with a constructed instance exhibiting it; explain why the
worst case may nevertheless be irrelevant here and what property of the real instances would
determine that; give four ways to measure the optimality gap without knowing OPT, and say which is
practical at this size; state the question that should have preceded all of this, and what answers
would change the recommendation; give three alternatives ranked by effort — including at least one
that would produce a provable bound — with the expected gain and cost of each; and state what
monitoring you would attach if the greedy algorithm ships.

Marks are for the reasoning about *when the worst case matters*, which is the question the colleague
is really asking without knowing it.

**B5. (40)** You are reviewing a design document for a system that must, for each of 200 million
users per day, answer three questions: how many distinct items has this user seen; is this item one
the user has already seen; and what are this user's top ten most-seen items. Exact answers are not
required — the document says "approximate is fine" without saying how approximate. Memory is the
binding constraint.

Design the algorithmic core and interrogate the requirement. Your answer must: state, for each of
the three questions, the exact-answer space cost at this scale, so the case for approximation is
quantified rather than assumed; select a sketch for each question and state its guarantee precisely
— including what kind of error it makes, in which direction, and with what probability; identify the
question for which the obvious sketch is the *wrong* choice and explain why; work through the
parameter selection for one sketch, showing the arithmetic from a target error to a memory figure;
state the five design questions you would put back to the document's author, since "approximate is
fine" is not a specification; explain how the sketches are merged across shards and what property
makes that possible; and state what happens at the boundaries — the user with three items and the
user with fifty million.

Then state which of your three answers you are least confident in and what experiment would settle
it.

---

## Marking guidance

Section A per question: 6 for the standard correct answer, complete; 4 for precision — every
case done, every assumption stated, correct quantifiers; 3 for an example not drawn from the
lessons; 2 for a stated limitation of your own answer or a connection to another course.

A proof with a missing case scores as an incomplete proof, not as a proof with a small error.
That standard is the point of the course.

Section B: a correct, complete answer is 24/40. The rest is judgement, quantitative
reasoning, and honesty about what your recommendation does and does not guarantee.
