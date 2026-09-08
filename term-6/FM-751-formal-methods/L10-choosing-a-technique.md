# FM-751 · Lesson 10 — Runtime Verification, Contracts, and Choosing a Technique

**Estimated study time:** 4 hours
**Prerequisites:** L01–L09; CA-731 L10, ML-741 L10

---

## 1. Orientation

Two final things.

The first is **runtime verification**: checking properties of a system as it runs, rather than
before. It is the weakest form of evidence in this course — it tells you only about the executions
that actually happened — and it is the one that runs in production, on real inputs, under real load,
forever. That combination makes it more valuable than its epistemic status suggests.

The second is the course's tenth learning outcome and the reason the other nine exist: **judging
which technique is proportionate to a given risk.** An engineer who model-checks everything is as
badly calibrated as one who checks nothing, and will be listened to less. The professional skill is
matching the evidence to the stakes, and being able to say precisely what each kind of evidence
establishes.

The framing to carry out of the course:

> **Every verification technique answers a different question, with a different cost, about a
> different artifact.** There is no ranking; there is a matching problem. Knowing the answers, the
> costs and the artifacts is what you now have.

## 2. Theory

### 2.1 Contracts

Preconditions, postconditions and invariants (L02), enforced at runtime.

- **Precondition**: the caller's obligation. Violated → the *caller* has a bug.
- **Postcondition**: the callee's guarantee. Violated → the *callee* has a bug.
- **Invariant**: true whenever no operation is in progress. Violated → the callee has a bug.

That attribution is the practical value: **a contract violation names the culprit**, which is why
contract failures are so much cheaper to diagnose than the downstream symptoms they prevent. A
`ValueError` three modules away from the actual mistake costs an hour; a precondition failure at the
boundary costs a minute.

Design by Contract's rules, which are easy to get wrong:

- **Preconditions may be weakened by a subtype, never strengthened.** A subclass that demands more
  than its supertype breaks substitutability.
- **Postconditions may be strengthened, never weakened.**
- **Invariants may be strengthened.**

These are the Liskov substitution principle stated precisely, and they explain why an override that
adds a check on its arguments is a design error rather than defensive programming.

The engineering realities:

- **Contracts are documentation that cannot go stale.** They are checked.
- **They cost runtime.** Assertion-heavy code can be measurably slower, so make them switchable and
  measure.
- **The disable question.** Disabling in production removes the cost and the protection. The
  defensible middle: keep cheap preconditions always on, keep expensive invariant checks on in
  staging and in a sampled fraction of production traffic, and never disable *input validation* —
  which is a different thing from an assertion (§2.4).
- **Assertions must be side-effect free**, or the program behaves differently when they are
  disabled. This is a real and confusing bug class.

### 2.2 Runtime verification

Generalises contracts from single operations to properties over *executions* — L03's temporal
properties, evaluated incrementally as the system runs.

- **A monitor** consumes a stream of events and evaluates a property.
- **Safety properties can be monitored** — a violation is witnessed by a finite prefix, so the
  monitor can report the moment it occurs (L03 §2.2).
- **Liveness properties cannot be definitively monitored** — no finite observation refutes `◇P`. The
  practical substitute is a **bounded** version: "answered within 30 seconds" is a safety property
  and is monitorable, and it is usually what you actually meant.

Forms in practice, most of which you already use without the name:

- **Assertions and contracts** — the local case.
- **Distributed invariant checking**: periodically verify a global property (every acknowledged
  write is present on a majority; the sum of per-shard counts equals the total).
- **Trace validation against a specification** (L07 §2.6) — the strongest form, and the one that
  directly connects the model to the running system.
- **Consistency checkers**: background jobs verifying that derived data matches its source — the
  index matches the table (DI-721 L03), the cache matches the store, the projection matches the
  event log (DI-721 L09). **These find real bugs continuously and are absurdly under-used.**
- **Anomaly detection on invariants**: the count went negative, the balance does not reconcile, the
  timestamp is in the future.

**What to do on a violation** is a design decision that must be made deliberately, and the options
have very different profiles:

- **Crash** (fail-fast). Correct when continuing risks corruption, and it converts a silent data
  corruption into a loud, diagnosable outage. Erlang's "let it crash" is this position, and it
  depends on having a supervisor and a recovery path.
- **Log and continue.** Correct when the property is advisory, and dangerous when it is not — a
  logged invariant violation nobody reads is worse than no check.
- **Alert and degrade.** Usually the right answer for a distributed system: stop serving the
  affected data, keep serving the rest.
- **Reject the operation.** For preconditions on external input — which is validation, not
  assertion.

The general rule: **the more likely a violation indicates corruption, the more it should stop
things.** An invariant about your own data structure failing means your state is wrong and
continuing writes bad data; a violation of an advisory expectation does not.

### 2.3 The technique matrix

The core of this lesson. Each technique, what it checks, what it costs, and what it establishes:

| Technique | Artifact | Coverage | Cost to adopt | Establishes |
|---|---|---|---|---|
| **Types** | code | all executions, shallow properties | none (already there) | no type errors, statically |
| **Assertions/contracts** | code | executions that occur | very low | this invariant held here, then |
| **Unit tests** | code | the cases you wrote | low | these cases work |
| **Property-based tests** | code | sampled, unbounded domain | low | no counterexample in N samples |
| **Fuzzing** | code | coverage-guided sampling | low | no crash found in this budget |
| **Runtime verification** | running system | real executions only | low–medium | these executions satisfied these safety properties |
| **Trace validation** | running system vs spec | real executions | medium | the implementation matched the spec on these runs |
| **Model checking** | a model | exhaustive, bounded | medium–high | no violation in any behaviour within these bounds |
| **SMT-based analysis** | a model of state | exhaustive over the encoding | low–medium | no state satisfies these constraints |
| **Refinement proof** | two specs | all behaviours | high | the design implements the spec |
| **Program verification** | code | all executions | very high | the code satisfies its specification |

Two observations. **The cheap techniques check code and the expensive ones check designs**, which is
why a good portfolio uses both ends rather than the middle. And **coverage and fidelity trade
off**: model checking gives exhaustive coverage of a model you wrote; property testing gives sampled
coverage of the code you shipped. Neither dominates.

### 2.4 Choosing, concretely

The decision procedure, in the order to apply it:

1. **What is the cost of being wrong?** A rendering glitch, a corrupted database, a security breach,
   a safety-critical failure. Everything follows from this.
2. **What kind of wrong?** A logic error in sequential code (types, tests, contracts); a concurrency
   or failure-interleaving bug (model checking — testing structurally cannot reach it); a
   configuration or reachability question (SMT); a performance problem (none of this — measure).
3. **What is the reversibility?** A two-way door needs less evidence than a one-way one (CA-731
   L10 §2.3).
4. **What is your team's capacity?** A technique nobody but you can maintain will be abandoned. This
   is a real constraint, not an excuse.
5. **What will the evidence be used for?** Your own confidence, a code review, a customer's security
   team, a regulator? Different audiences need different artifacts.

The defaults this course recommends, and they are deliberately modest:

- **Everything**: types, and assertions for the invariants you can state.
- **Data structures and pure logic**: property-based tests. Cheap, and they find real bugs.
- **Anything parsing untrusted input**: fuzzing. Non-negotiable.
- **Any distributed protocol you designed yourself**: model check it. **This is the highest-value
  application in the course** — the bugs are unreachable by testing and the cost of finding them in
  production is enormous.
- **Access control, network reachability, and configuration constraints**: SMT. Cheap, and it
  answers questions nobody can answer by reading.
- **Production**: runtime invariant checking and consistency checkers, with an explicit decision
  about what a violation does.
- **A critical component you will rely on for years**: consider trace validation, so the
  specification stays connected to the code.
- **Full program verification**: only for a kernel, a crypto primitive, an allocator, or where a
  regulator requires it.

### 2.5 The proportionality argument

The case to be able to make to colleagues, because you will have to:

Formal methods are **not** all-or-nothing, and the version most people are resisting — full
verification of everything — is a straw man. **Specifying and checking one protocol takes days, not
months**, and the reported industrial experience is consistent: the specification finds bugs, and it
finds them at design time.

Three arguments that work, in roughly this order of persuasiveness:

1. **The bug it would have caught.** If you have a production incident whose cause was a subtle
   interleaving, model it retrospectively and show the checker finding it in ten seconds (L04 Stage
   9). This is the single most persuasive artifact available.
2. **The cost comparison.** Two weeks of specification against the cost of the incident: the outage,
   the data loss, the engineering time, the customer trust. For a protocol at the heart of a
   system, the arithmetic is not close.
3. **The design clarity.** Even with no bug found, the specification forces the ambiguities out and
   documents the design in a form that survives the team (L01 §1). Practitioners report this
   consistently as an independent benefit.

And the counter-arguments to concede, because conceding them makes the rest credible: **it does not
verify your code**; **it is not free**; **it requires a person to learn it and keep it current**; and
**a stale specification is worse than none**, because people trust it.

### 2.6 What you can now claim

The professional obligation this course has been building toward. After the term artifact, you can
say:

> "The consensus protocol's design is specified in TLA+ and model-checked for these five safety
> properties and this liveness property, exhaustively, over all executions with three nodes, two
> values, terms bounded by four, and at most one crash — under a network model that drops,
> duplicates and reorders messages but does not corrupt them. The model check found this bug, which
> we fixed. The implementation is tested against properties derived from the specification, and
> validated against it by trace checking during fault injection. The correspondence between
> specification and implementation is weakest here, and here is what could diverge undetected."

Every clause of that is a specific, defensible claim, and the last one is what makes the rest
trustworthy. Compare it against "we tested it thoroughly", and the difference is the reason this
course exists.

Overclaiming is the fastest way to discredit the technique and yourself. **Say exactly what you
checked, under what assumptions, within what bounds, and what remains unverified.**

## 3. Construction: instrument, decide, and write it up

Build in `mpse/fm751/l10/`.

**Stage 1 — contracts.** Add preconditions, postconditions and invariants to three components of
your term artifact. Make them switchable. Measure the performance cost of each level. Then decide,
per component, what runs in production, and write down the reasoning.

**Stage 2 — the substitution rules.** Construct a class hierarchy that violates the contract rules —
an override that strengthens a precondition. Show the substitutability failure concretely. Then fix
it. Then check whether any of your real code has this defect.

**Stage 3 — violation policy.** For each contract and invariant in Stage 1, decide what a violation
does: crash, log, alert and degrade, or reject. Implement each policy. Then trigger each violation
deliberately in a controlled environment and observe the resulting behaviour end to end — including
what the on-call engineer would see. Revise any policy whose behaviour surprised you.

**Stage 4 — distributed invariant checking.** Implement a background checker for a global property
of your DS-701 system: every acknowledged write is present on a majority; committed log prefixes
agree across nodes. Run it during chaos experiments. Report any violation found — and report the
checker's own cost and its false positive rate under normal churn, because a checker that fires
during ordinary rebalancing will be turned off.

**Stage 5 — consistency checkers.** Implement one for a derived-data relationship in your DI-721
system: the index matches the table, or the projection matches the event log. Run it continuously.
Then deliberately corrupt the derived data and confirm detection, and measure the detection latency.

**Stage 6 — bounded liveness as safety.** Take three liveness properties from your system. Rewrite
each as a bounded safety property with an explicit deadline. Monitor them at runtime. Report which
bound you chose and why — this is exactly CA-731 L07's SLO reasoning arriving from the formal
methods side, and noticing the connection is the point.

**Stage 7 — the technique portfolio.** For your entire term artifact — the CA-731 platform, the
ML-741 system, the DS-701 protocol — produce the table: each component, its risk, its reversibility,
the technique(s) applied, the cost, and what each establishes. Identify the component with the
highest risk and the weakest evidence, and say what you would do about it.

**Stage 8 — the persuasion artifact.** Take a real production incident (yours, or a public
postmortem) whose cause was a design-level concurrency or failure-interleaving bug. Model it
retrospectively and show the checker finding it. Time how long the modelling took. Write the
one-page case you would make to an engineering lead, with that time and that counterexample in it.

**Stage 9 — the evidence statement.** Write, for your term artifact, the §2.6 paragraph: exactly
what has been verified, under what assumptions, within what bounds, by what technique, and what
remains unverified. One page. **Then have someone try to poke a hole in it** — to find a claim
stronger than the evidence supports. Revise. This document is the term artifact's formal methods
deliverable and the course's final exercise in intellectual honesty.

## 4. Failure modes

- **Assertions with side effects.** Behaviour changes when they are disabled.
- **Assertions used for input validation.** Disabling them removes the validation.
- **Contracts disabled everywhere in production.** All cost, no benefit — you paid to write them and
  get nothing.
- **A logged invariant violation nobody reads.** Worse than no check, because it creates the
  impression of coverage.
- **A consistency checker that fires during normal operation.** Turned off within a week.
- **Monitoring liveness directly.** Cannot be refuted in finite time; use a bounded version.
- **No decided violation policy.** Whatever happens is an accident.
- **Model checking everything.** Miscalibrated, expensive, and it discredits the technique.
- **Model checking nothing.** The interleaving bugs are found in production.
- **A stale specification.** Trusted, and wrong.
- **Overclaiming.** "We proved it correct" when you checked three nodes.
- **Adopting a technique nobody else on the team can maintain.**

## 5. Exercises

### Warm-up (30 min)

1. Distinguish precondition, postcondition and invariant by who is at fault when each is violated.
2. Give the contract substitution rules and explain how they restate the Liskov substitution
   principle.
3. Explain why safety can be monitored at runtime and liveness cannot, and the practical
   substitute.

### Core (3 h)

4. Complete Stages 1–3: contracts with measured costs, the substitution violation demonstrated and
   checked for in real code, and every violation policy triggered and observed.
5. Complete Stages 4–5: the distributed invariant checker with its false positive rate, and the
   consistency checker with its detection latency.
6. Complete Stage 6 and connect the bounds you chose to your SLOs.
7. Complete Stage 7 — the technique portfolio table with the highest-risk/weakest-evidence component
   identified.

### Challenge

8. Complete Stages 8–9: the retrospective model check with its timing, the one-page persuasion case,
   and the evidence statement survived an adversarial reading.
9. Write a **verification strategy** (1,500–2,000 words) for a real system — your team's, or one you
   know well. Cover: the components ranked by the cost of being wrong; the technique proposed for
   each with its cost in engineer-days and what it would establish; what you explicitly would *not*
   verify and why; how the artifacts stay current as the system changes (the stale-specification
   problem is the one that kills these efforts); who maintains them; and how you would demonstrate
   value within a quarter so the effort survives its first budget review. Then include the section
   most such documents omit: **what would make you abandon this**, and what evidence would tell you
   the effort was not paying. A strategy that cannot fail cannot be evaluated, and being able to
   write that section is the difference between advocacy and engineering.

## 6. Self-check

1. Distinguish the three contract kinds by fault attribution.
2. Give the substitution rules and the principle they restate.
3. Why must assertions be side-effect free, and why is validation different from assertion?
4. Why can safety be monitored and liveness not? What replaces it?
5. Give four responses to a violation and the rule for choosing.
6. Give the eleven-row technique matrix's key columns for five techniques.
7. Give the five questions of the decision procedure, in order.
8. Give this course's recommended defaults.
9. Give the three persuasion arguments and the four concessions.
10. State the evidence claim you can now make, with every qualifying clause.

## 7. Primary sources

- **Meyer, *Object-Oriented Software Construction*, 2nd ed.**, the Design by Contract chapters — the
  substitution rules, argued properly.
- **Leucker & Schallhart, "A Brief Account of Runtime Verification" (JLAP 2009)** — the survey.
- Havelund & Roşu, "Monitoring Programs Using Rewriting" (ASE 2001) and the runtime verification
  literature generally.
- **Newcombe et al., "How Amazon Web Services Uses Formal Methods" (CACM 2015)** — the
  proportionality argument, from people who made it successfully inside a large organisation.
  Read the section on how they persuaded engineers.
- Hillel Wayne's writing on when formal methods do and do not pay — the honest practitioner
  perspective, including the failures.
- Armstrong, *Programming Erlang*, on "let it crash" and supervision — §2.2's fail-fast position,
  developed into an architecture.
- Google, *Site Reliability Engineering*, on monitoring and alerting (with CA-731 L07) — the
  operational counterpart to runtime verification.
- CA-731 L10 and ML-741 L10 — the review methods this lesson's portfolio table feeds into.

---

**Previous:** [L09](L09-smt-solvers.md) ·
**Next:** [Problem sets](problem-sets.md) · [Exam](exam.md)

*This completes FM-751, and Term 6. The course asked how you know a design is right before you build
it. The answer it gives is not a technique but a discipline: state what you are claiming, choose
evidence proportionate to what it would cost to be wrong, and be precise about what your evidence
establishes and what it does not. That discipline is what the capstone will ask you to demonstrate.*
