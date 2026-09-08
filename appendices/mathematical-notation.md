# Appendix — Mathematical Notation

A reference for the notation used across the programme. Nothing here is advanced; it is collected so
that meeting an unfamiliar symbol never blocks a lesson.

If a symbol is unfamiliar, look it up here, use it, and move on. Fluency comes from use, not from
studying notation in advance.

---

## 1. Logic

| Notation | Read | Meaning |
|---|---|---|
| `¬P`, `~P` | not P | negation |
| `P ∧ Q`, `P /\ Q` | P and Q | conjunction |
| `P ∨ Q`, `P \/ Q` | P or Q | disjunction (inclusive) |
| `P ⇒ Q`, `P => Q` | P implies Q | false only when P true and Q false |
| `P ⇔ Q`, `P <=> Q` | P iff Q | equivalence |
| `∀x ∈ S . P(x)` | for all x in S, P | universal quantification |
| `∃x ∈ S . P(x)` | there exists x in S with P | existential quantification |
| `∃!x . P(x)` | there is exactly one x | unique existence |
| `⊢ P` | P is derivable | provable in the system |
| `⊨ P` | P is valid | true in all models |

**The implication trap.** `P ⇒ Q` is *true* whenever P is false. This is why a property whose
antecedent is never satisfied is vacuously true — the source of FM-751 L03 §2.7's vacuity problem.

**Quantifier order matters.** `∀x ∃y . P(x,y)` (for each x there is *some* y) is much weaker than
`∃y ∀x . P(x,y)` (there is *one* y that works for all x). Distributed systems results turn on this
distinction constantly.

## 2. Sets

| Notation | Meaning |
|---|---|
| `{a, b, c}` | set by enumeration |
| `{x ∈ S : P(x)}` | set-builder (filter) |
| `{f(x) : x ∈ S}` | image (map) |
| `x ∈ S`, `x ∉ S` | membership |
| `∅`, `{}` | the empty set |
| `S ⊆ T`, `S ⊂ T` | subset, proper subset |
| `S ∪ T`, `S ∩ T`, `S \ T` | union, intersection, difference |
| `S × T` | Cartesian product: `{(s,t) : s ∈ S, t ∈ T}` |
| `𝒫(S)`, `SUBSET S` | power set: all subsets of S |
| `\|S\|`, `Cardinality(S)` | size |

Standard sets: `ℕ` naturals, `ℤ` integers, `ℚ` rationals, `ℝ` reals, `𝔹` booleans.

**Quorum intersection** (DS-701 L03) is a set fact: any two subsets of an n-element set, each of size
greater than n/2, must intersect — because otherwise their union would exceed n.

## 3. Functions and relations

| Notation | Meaning |
|---|---|
| `f : A → B` | f maps A to B |
| `f(x)`, `f[x]` | application (TLA+ uses square brackets) |
| `dom f`, `DOMAIN f` | domain |
| `f ∘ g` | composition: `(f ∘ g)(x) = f(g(x))` |
| `[x ∈ S ↦ e]` | function definition (TLA+) |
| `[f EXCEPT ![x] = e]` | f with one point changed (TLA+) |
| `[S → T]` | the set of all functions from S to T |
| `R ⊆ A × B` | a relation is a set of pairs |
| `R⁺`, `R*` | transitive closure, reflexive-transitive closure |

**Injective** (one-to-one), **surjective** (onto), **bijective** (both). A **partial order** is
reflexive, antisymmetric and transitive; a **total order** additionally compares every pair. A
**partial order** is exactly what the happens-before relation is (DS-701 L02) — and the fact that it
is *partial* is why concurrent events exist.

A **lattice** is a partial order in which every pair has a least upper bound (join). CRDT merge is a
join in a semilattice (DS-701 L07), and that is what makes it commutative, associative and
idempotent.

## 4. Asymptotics

| Notation | Meaning | Read |
|---|---|---|
| `f = O(g)` | f grows no faster than g | upper bound |
| `f = Ω(g)` | f grows at least as fast as g | lower bound |
| `f = Θ(g)` | both | tight bound |
| `f = o(g)` | f grows strictly slower | strict upper |
| `f = ω(g)` | f grows strictly faster | strict lower |

Formally, `f = O(g)` iff there exist c and n₀ such that `f(n) ≤ c·g(n)` for all `n ≥ n₀`.

**The equals sign is an abuse of notation** — `O(g)` is a set, and `f = O(g)` means `f ∈ O(g)`. It is
universal and harmless once you know.

Common orders, ascending: `1 < log log n < log n < √n < n < n log n < n² < n³ < 2ⁿ < n!`.

**Amortised** analysis bounds the average over a sequence of operations, not each one (CS-621 L01).
**Expected** bounds average over the algorithm's randomness. **Worst-case** bounds every case. These
are three different claims and conflating them is a common error.

## 5. Probability and statistics

| Notation | Meaning |
|---|---|
| `P(A)` | probability of event A |
| `P(A \| B)` | conditional probability of A given B |
| `E[X]` | expected value |
| `Var(X)`, `σ²` | variance |
| `σ` | standard deviation |
| `X ~ D` | X is distributed as D |
| `p50`, `p99` | the 50th and 99th percentiles |

**Bayes**: `P(A|B) = P(B|A)·P(A) / P(B)`.

**Linearity of expectation**: `E[X + Y] = E[X] + E[Y]`, always — even when X and Y are dependent.
This is the workhorse of randomised algorithm analysis (CS-621 L06).

**Percentiles do not average.** The mean of two instances' p99s is not the p99 of their combined
traffic. Aggregate histograms, not percentiles (PY-602 L01, DS-701 L10).

**Distribution shift** (ML-741 L06) uses `P(X)`, `P(y)` and `P(y|X)`: covariate shift changes the
first, label shift the second, concept drift the third.

## 6. Temporal logic (FM-751)

| Notation | Read | Meaning over a behaviour |
|---|---|---|
| `□P` | always P | P holds in every state |
| `◇P` | eventually P | P holds in some state |
| `P ⇝ Q` | P leads to Q | `□(P ⇒ ◇Q)` |
| `○P` | next P | P holds in the next state |
| `□◇P` | infinitely often P | P recurs forever |
| `◇□P` | eventually always P | P becomes permanently true |
| `x'` | x prime | the value of x in the next state |
| `[Next]_v` | | a step satisfying Next, or leaving v unchanged |
| `WF_v(A)`, `SF_v(A)` | | weak, strong fairness on action A |

## 7. Inference rules (CS-641, FM-751)

```
    premise₁    premise₂
    ────────────────────  (RULE-NAME)
         conclusion
```

Read: if the premises hold, the conclusion holds. Rules with no premises are axioms.

| Notation | Meaning |
|---|---|
| `Γ ⊢ e : τ` | in context Γ, expression e has type τ |
| `Γ, x : τ` | Γ extended with the binding x : τ |
| `e → e'` | e steps to e' (small-step) |
| `e ⇓ v` | e evaluates to value v (big-step) |
| `e[v/x]` | e with v substituted for x |
| `τ₁ <: τ₂` | τ₁ is a subtype of τ₂ |
| `{P} S {Q}` | Hoare triple |
| `⊥`, `⊤` | bottom, top |

## 8. Distributed systems (DS-701)

| Notation | Meaning |
|---|---|
| `a → b` | a happens before b |
| `a ∥ b` | a and b are concurrent (neither happens before the other) |
| `L(e)` | Lamport timestamp of event e |
| `V(e)` | vector clock of event e |
| `V₁ ≤ V₂` | ∀i: V₁[i] ≤ V₂[i] |
| N, R, W | replicas, read quorum, write quorum |

`V₁ ≤ V₂ ∧ V₁ ≠ V₂` means the first happened before the second; if neither `V₁ ≤ V₂` nor
`V₂ ≤ V₁`, they are concurrent.

## 9. Reading a formula

A method for an unfamiliar formula:

1. **Identify the top-level structure.** Is it a conjunction, an implication, a quantification?
2. **Name the bound variables** and their domains.
3. **Read the innermost predicate in English.**
4. **Work outward.**
5. **Try a small instance** — two elements, three states. Concrete beats abstract every time.
6. **Construct one thing that satisfies it and one that does not.** If you can do both, you have
   understood it.

Step 6 is the one that works. A formula you have satisfied and violated by hand is a formula you
know.

## 10. Writing mathematics readably

For your own specifications and proofs:

- **Name things.** `Quorum(S)` beats `|S| > n/2` repeated eleven times.
- **Use aligned conjunction lists** (TLA+ style): bullets of `∧` at the same indentation, so the
  structure is visible without counting parentheses.
- **State the domain of every quantifier.** `∀n ∈ Nodes`, not `∀n`.
- **Define before use.**
- **Prefer several small definitions to one large formula.**
- **Say what a formula means in English** immediately after stating it. A reader who understands the
  English will read the formula for precision; one who does not has no way in.
