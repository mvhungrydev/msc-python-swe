# FM-751 · Lesson 08 — Property-Based Testing as Lightweight Verification

**Estimated study time:** 4.5 hours
**Prerequisites:** L01, L07; SE-511 L04 (property-based testing)

---

## 1. Orientation

SE-511 L04 introduced property-based testing as a testing technique. This lesson revisits it as
what it actually is:

> **Property-based testing is verification with the exhaustiveness removed and the code put back
> in.** A model check reasons about all behaviours of a model; property-based testing samples the
> behaviours of the real implementation. One trades coverage for fidelity; the other trades
> fidelity for coverage.

That framing makes the two techniques complementary rather than competing, and it is why this lesson
sits after refinement rather than in SE-511:

- The **specification** (L01) supplies the properties.
- The **refinement mapping** (L07) supplies the model-based test oracle.
- **Property-based testing** runs the actual code against them.

This is the practical answer to "the spec doesn't match the code" (L01 §2.6), and it is the highest
value-for-effort technique in this course for ordinary engineering. It requires no new notation, it
runs in CI, it produces minimal reproducing examples, and it finds real bugs in real code.

## 2. Theory

### 2.1 The idea, restated formally

A property is a universally quantified claim:

```
∀ x ∈ D . P(x)
```

A property-based test picks many `x` from `D` according to a generator and checks `P(x)`. Failure
disproves the claim and produces a counterexample; success is evidence, not proof.

The connection to the rest of the course: **a postcondition (L02) is a property; a safety invariant
(L03) is a property; a refinement mapping (L07) turns an abstract specification into a property.**
Every specification artifact you have built is a source of properties, and this is what makes
specification pay for itself twice.

Three mechanisms make the technique work in practice, and the third is the one people
underestimate:

1. **Generation**: producing values from a domain, with a distribution that reaches interesting
   cases. A generator that never produces an empty list will not find the empty-list bug, and
   biasing toward edge cases — empty, singleton, duplicates, extremes, boundary values — is most of
   the craft.
2. **Shrinking**: on failure, automatically reducing the counterexample to a minimal one. **This is
   what makes the technique usable**: a failure on a 300-element list of random floats teaches
   nothing; the same failure shrunk to `[0.0, -0.0]` teaches everything.
3. **Replay**: a failing seed is recorded so the failure is reproducible and becomes a permanent
   regression test.

### 2.2 Finding properties

The perennial difficulty is "what property do I test?" The standard patterns, which cover most real
cases:

- **Invariants**: something always true of the output. Sorting produces a non-decreasing sequence; a
  balanced tree's height stays within the bound; a parsed document has no null fields.
- **Round-tripping**: `decode(encode(x)) == x`. Serialisation, compression, parsers, currency
  conversion. The easiest to find and frequently productive — and note the asymmetry: the reverse
  (`encode(decode(y)) == y`) is often *false* for non-canonical formats, and discovering that is
  itself a finding.
- **Oracles**: compare against a simpler, obviously-correct implementation. Your optimised version
  against the naive one; your LSM against a dict; your index against a linear scan (DI-721 L03
  Stage 8's oracle pattern).
- **Metamorphic relations**: a known relationship between outputs for related inputs.
  `sort(xs ++ ys)` is a permutation of `sort(ys ++ xs)`; adding a constant to every input shifts the
  mean by that constant. Invaluable when there is no oracle and no closed-form answer — which is the
  usual situation for numerical and ML code.
- **Algebraic laws**: idempotence (`f(f(x)) == f(x)`), commutativity, associativity, identity
  elements. **DS-701 L07's CRDT merge laws are exactly this**, and property-testing them is the
  natural check.
- **Postconditions from contracts** (L02): the postcondition is the property.
- **Preservation**: the operation preserves what it should — a rebalance preserves the set of
  elements; a compaction preserves the live keys (DI-721 L02).

### 2.3 Stateful and model-based testing

The most powerful form, and the one that connects to refinement.

Rather than testing a function, generate a **sequence of operations** on a stateful system and check
properties throughout. The model-based version:

1. Write a **model** — a simple, obviously-correct implementation. A dict for a key-value store; a
   list for a queue; a set for a membership structure.
2. Generate a random sequence of operations.
3. Run each against both the real system and the model.
4. After each, check that the observable results agree, and that any invariants hold.

**This is refinement checking by sampling.** The model is the abstract specification (L07 §2.1); the
comparison is the refinement mapping applied at each step. What a model check proves for all
behaviours of the model, this checks for sampled behaviours of the code.

The technique finds the bugs that unit tests structurally cannot: those requiring a specific
*sequence* — insert, delete, insert the same key, compact, read. Nobody writes that test by hand;
the generator writes it in the first thousand cases.

Two refinements that greatly increase its power:

- **Preconditions on commands**, so the generator produces meaningful sequences (do not `pop` an
  empty stack unless you are testing that it raises).
- **Concurrent stateful testing**: generate operations across threads and check that the observed
  results are consistent with *some* sequential ordering — that is, check linearizability (L07
  §2.5). Hypothesis, QuickCheck's `quickcheck-state-machine`, and Erlang's original commercial
  QuickCheck all support this, and it is how the Riak and LevelDB bugs in the literature were found.

### 2.4 From specification to properties

The concrete workflow this course exists to enable:

1. **The specification's invariants become properties.** `TypeOK` becomes a check on the
   implementation's state; election safety becomes an assertion over the cluster.
2. **The specification's actions become the command generator.** Each action is a command; the
   action's guard is the command's precondition.
3. **The specification itself becomes the model.** Execute the TLA+ specification's next-state
   relation in the test harness (or reimplement it simply) and compare against the real system.
4. **Counterexamples from the model checker become directed test cases.** The eleven-step trace TLC
   found becomes an explicit test in the implementation's suite — the highest-value transfer between
   the two techniques, and permanently valuable.

Point 4 is worth emphasising. Every counterexample the model checker produced in L06 should end up
as a test in the implementation's suite. That is how a design-level bug stays fixed once the code is
written.

### 2.5 What it establishes, honestly

- **A failure is definitive.** The counterexample is real, in the real code.
- **Success is evidence proportional to coverage.** Ten thousand cases from a good generator over a
  small domain is strong; ten thousand from a poor generator over a huge domain may exercise one
  code path.
- **It says nothing about what the generator cannot produce.** A generator that never produces
  concurrent conflicts will not find concurrency bugs.
- **Coverage should be measured.** Track which branches and which state-space regions the generated
  cases reached; Hypothesis's `target()` and coverage-guided generation exist for this. A property
  test with 10% branch coverage is a slow unit test.

Compared with model checking (L04 §2.5):

| | Model checking | Property-based testing |
|---|---|---|
| Subject | a model | the real code |
| Coverage | exhaustive, bounded | sampled, unbounded |
| Cost to start | learn a notation | write a generator |
| Finds | design bugs | implementation bugs |
| Counterexample | a trace over the model | a minimal input to the code |
| Runs in CI | rarely | always |

**Use both.** The model check finds the design bug before you write the code; the property test finds
the implementation bug and keeps finding it in CI for years. Neither substitutes for the other, and
a team with the second and not the first is in a much better position than a team with the first and
not the second — which is the honest ranking for most engineering.

### 2.6 Fuzzing, and the boundary

**Fuzzing** is property-based testing where the property is usually "does not crash" and the
generator is coverage-guided — it mutates inputs and keeps those that reach new code paths
(AFL, libFuzzer, Atheris for Python).

The relationship: property-based testing has richer properties and structured generation; fuzzing
has better exploration through coverage feedback. They are converging — Hypothesis has
coverage-guided features and modern fuzzers support structured input.

Where each fits: **fuzz anything parsing untrusted input** (a protocol decoder, a file format, a
deserialiser) — the crash-freedom property alone finds real vulnerabilities. **Property-test
anything with semantic properties** — data structures, algorithms, stateful components.

The strongest combination for a parser: a fuzzer for crash-freedom and a property test for
round-tripping, sharing the same corpus.

## 3. Construction: derive tests from specifications

Build in `mpse/fm751/l08/`. Hypothesis in Python; the concepts transfer.

**Stage 1 — the property catalogue.** For ten components you have built in this programme, write
three properties each using §2.2's patterns. Name the pattern used for each. Then run them and
record how many found a bug — for a first pass over untested code, the rate is usually higher than
expected.

**Stage 2 — generators that reach the interesting cases.** Take one component and write a naive
generator. Measure branch coverage. Then improve the generator — bias toward edges, add
`st.composite` strategies producing structurally valid inputs, add duplicates and boundary values —
and re-measure. Report the coverage before and after, and any bug the improved generator found.

**Stage 3 — shrinking.** Deliberately introduce a bug that triggers on a complex input. Observe the
raw counterexample and the shrunk one. Then write a custom shrinker for a domain type where the
default shrinks badly, and compare. This makes vivid why shrinking is what makes the technique
usable.

**Stage 4 — model-based testing of the storage engine.** Take your DI-721 LSM. Model it as a dict.
Generate sequences of `put`, `get`, `delete`, `scan` and `compact`. Compare after every operation.
Run a hundred thousand sequences. **This will find a bug**; if it does not, your generator is not
producing the interleaving of deletes and compactions that matters, so fix the generator first.

**Stage 5 — CRDT laws.** Property-test your DS-701 L07 CRDTs against the three merge laws —
commutativity, associativity, idempotence — over randomly generated update sequences and delivery
orders. Then verify strong eventual consistency: replicas that received the same set of updates in
any order have the same state. Then break the merge subtly and confirm the properties catch it.

**Stage 6 — properties from the specification.** Take your L06 TLA+ specification. Translate its
invariants into runtime properties over your implementation's state. Translate its actions into a
command generator with the guards as preconditions. Run it. Report divergences — this is L07 Stage 8
by sampling rather than by trace validation, and comparing the two techniques' findings is
instructive.

**Stage 7 — counterexamples become tests.** Take every counterexample the model checker produced in
L06 and encode each as a directed test against the implementation. Run them. Report how many the
implementation actually fails — some will, because the design bug was fixed in the specification and
never in the code, or the code has an independent bug at the same point.

**Stage 8 — concurrent linearizability testing.** Generate concurrent operation sequences against a
shared structure from PY-601, record the real-time ordering of invocations and responses, and check
that a valid sequential ordering exists (a linearizability checker — yours from DS-701, or
Porcupine). Then introduce a subtle race and confirm it is caught. Report how many runs it took.

**Stage 9 — fuzzing.** Fuzz a parser — your own, or a format you can implement one for — with
Atheris. Run for an hour. Report crashes found and coverage reached. Then add a round-tripping
property test over the same corpus and compare what each found. Then combine them: seed the property
test's examples from the fuzzer's corpus.

## 4. Failure modes

- **Weak generators.** The property is fine and it never sees an interesting input.
- **Coverage unmeasured.** A property test with poor coverage is a slow unit test with extra
  ceremony.
- **Only round-trip properties.** Easy to write and they miss semantic bugs entirely.
- **A model as complex as the implementation.** Then it has the same bugs, and agreement proves
  nothing. Keep the model stupid.
- **Stateless testing of stateful code.** The sequence-dependent bugs are the ones you cannot write
  by hand.
- **No preconditions on commands.** Most generated sequences are rejected immediately and the
  effective sample size collapses.
- **Non-deterministic properties.** Flaky, so they get deleted, so the coverage is lost.
- **Failing seeds not recorded.** The bug returns and is rediscovered from scratch.
- **Too few examples.** The default (often 100) is a smoke test; run 10,000+ in a nightly job.
- **Treating success as proof.** It is evidence proportional to coverage; say so.
- **Model checking without then deriving tests.** The design is verified and the code is not.

## 5. Exercises

### Warm-up (30 min)

1. State property-based testing as verification with exhaustiveness removed, and give the
   three mechanisms that make it work.
2. Give seven patterns for finding properties, with an example of each from your own code.
3. Explain why model-based stateful testing is refinement checking by sampling.

### Core (3.5 h)

4. Complete Stages 1–3: thirty properties with their patterns named, the generator coverage
   improvement, and the shrinking comparison.
5. Complete Stage 4 and report the bug found — and if none was found, the generator fix that made
   one appear.
6. Complete Stage 5: CRDT laws, strong eventual consistency, and a deliberate break caught.
7. Complete Stage 7 and report how many model-checker counterexamples the implementation actually
   fails.

### Challenge

8. Complete Stages 6, 8 and 9: specification-derived properties with their divergences, concurrent
   linearizability testing with the run count to detection, and fuzzing compared and combined with
   property testing.
9. Build a **specification-to-test compiler**: given a TLA+ specification (or a restricted subset),
   generate a Hypothesis stateful test automatically — actions become commands, guards become
   preconditions, invariants become assertions, and the specification's next-state relation becomes
   the model. Run it against your implementation. Then assess it honestly: what did the generated
   test find that hand-written tests did not; what does the translation lose; and where did the
   generated test check something the implementation was never intended to satisfy (because the
   specification abstracted away something the code cares about)? That last category is the
   interesting one — it is the specification-to-code gap made concrete, item by item, and it is the
   most precise account of that gap you will produce in this course.

## 6. Self-check

1. State the relationship between property-based testing and model checking in terms of what each
   trades.
2. Give the three mechanisms and say which makes the technique usable in practice.
3. Give seven patterns for finding properties.
4. Why is metamorphic testing valuable when there is no oracle?
5. Explain model-based stateful testing and its relation to refinement.
6. Give the four-step workflow from a specification to a property test.
7. Why should model checker counterexamples become implementation tests?
8. What does a passing property test establish, and what does it not?
9. Compare model checking and property-based testing across the six rows of §2.5's table.
10. When would you fuzz rather than property-test, and what is the strongest combination for a
    parser?

## 7. Primary sources

- **Claessen & Hughes, "QuickCheck: A Lightweight Tool for Random Testing of Haskell Programs"
  (ICFP 2000)** — the original, and still the clearest statement.
- **Hughes, "Experiences with QuickCheck: Testing the Hard Stuff and Staying Sane" (2016)** and his
  talks on testing distributed databases — where the technique meets real systems; read this for the
  Riak and LevelDB findings.
- Arts, Hughes et al., "Testing Erlang Data Types with Quviq QuickCheck" — commercial experience,
  including stateful and concurrent testing.
- The Hypothesis documentation, particularly on stateful testing, `target()`, and shrinking —
  MacIver's writing on shrinking algorithms is unusually good.
- Chen et al., "Metamorphic Testing: A Review of Challenges and Opportunities" (ACM CSUR 2018).
- Zalewski, the AFL technical whitepaper — coverage-guided fuzzing, explained by its author.
- Godefroid, Levin & Molnar, "SAGE: Whitebox Fuzzing for Security Testing" (CACM 2012).
- Kingsbury, the Jepsen reports — generated concurrent histories checked for linearizability,
  applied to real databases; the industrial exemplar of Stage 8.
- SE-511 L04, re-read now that the properties can come from a specification.

---

**Previous:** [L07](L07-refinement.md) · **Next:**
[L09 — SMT Solvers and Z3 for Engineers](L09-smt-solvers.md)
