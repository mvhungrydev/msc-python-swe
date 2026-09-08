# FM-751 — Formal Methods & Verification for Practitioners

**Term:** 6 · **Credits:** 15 · **Nominal hours:** 140
**Prerequisites:** CS-641, DS-701
**Co-requisites:** CA-731, ML-741

---

## Driving question

> How do you know a design is right before you build it?

Testing tells you a system works on the cases you thought of. That is valuable, and it is
structurally incapable of telling you about the case you did not think of — which, for concurrent
and distributed systems, is where essentially all the serious bugs live (DS-701 L10 §2.4: you cannot
attach a debugger to "the system", and the timing *is* the bug).

Formal methods offer a different kind of evidence: **a claim about all executions, established by
mathematics rather than by sampling.** The claim is bounded — it is about a model, under stated
assumptions — and within those bounds it is genuinely a proof rather than an absence of failures.

This course is deliberately practitioner-facing. It is not a course in program verification, and you
will not verify a compiler. The position it takes:

> **The highest-value formal methods for a working engineer are the lightweight ones**: writing a
> specification precisely enough to expose an ambiguity, model-checking a protocol design before
> implementing it, using property-based testing to explore a space you could not enumerate, and
> using an SMT solver to answer a question about a configuration that no amount of reading would
> settle. The heavyweight ones — full functional verification of an implementation — are real,
> valuable in specific domains, and out of proportion for most work.

The most persuasive evidence for this position is industrial: Amazon's use of TLA+ on S3, DynamoDB
and EBS found bugs that had survived design review, code review and extensive testing, in designs
their best engineers were confident about. The technique's value was not proving correctness — it
was **finding the counterexample nobody could have imagined**.

## Learning outcomes

On completion you will be able to:

1. **Write** a specification precisely enough that ambiguity is exposed, and explain what a
   specification is for.
2. **Use** Hoare logic, invariants and variants to reason about the correctness of sequential code,
   and to write assertions that carry real weight.
3. **Model** a concurrent or distributed design in TLA+ and PlusCal.
4. **Express** safety and liveness properties in temporal logic, and know which of your requirements
   are which.
5. **Run** a model checker, interpret its counterexamples, and manage state space explosion.
6. **Apply** refinement to relate a design to a specification and to an implementation.
7. **Use** property-based testing as lightweight verification, including stateful and model-based
   testing.
8. **Use** an SMT solver to answer engineering questions about constraints, configurations and
   policies.
9. **Apply** runtime verification and contracts where static verification is impractical.
10. **Judge** which technique is proportionate to a given risk, and what each one's evidence
    actually establishes.

## Lessons

| # | Title | Hours |
|---|---|---|
| L01 | Specifications as Artifacts | 4 |
| L02 | Hoare Logic, Invariants, and Reasoning About Code | 5 |
| L03 | Temporal Logic: Safety and Liveness | 4.5 |
| L04 | Model Checking: How It Works and Where It Stops | 4.5 |
| L05 | TLA+ and PlusCal | 5 |
| L06 | Specifying a Distributed Protocol | 5 |
| L07 | Refinement | 4.5 |
| L08 | Property-Based Testing as Lightweight Verification | 4.5 |
| L09 | SMT Solvers and Z3 for Engineers | 4.5 |
| L10 | Runtime Verification, Contracts, and Choosing a Technique | 4 |

## Assessment

| Component | Weight |
|---|---|
| Problem set 1 (L01–L04): specifications, proofs, and a first model check | 20% |
| Problem set 2 (L05–L07): a distributed protocol specified, checked, and refined | 20% |
| Problem set 3 (L08–L10): lightweight verification applied to real systems | 20% |
| Written exam | 25% |
| Course position paper (1,500 words) | 15% |

## Term 6 build artifact (with CA-731 and ML-741)

**Formal evidence for the systems you built.** Specifically:

- A **TLA+ specification of one distributed protocol** from DS-701 — your Raft implementation, your
  saga engine, or your CRDT merge — with safety and liveness properties stated and model-checked.
- **A bug found by the model checker**, or a documented demonstration that the design is correct
  within stated bounds. Either outcome is a pass; a specification that found nothing and was never
  going to is not.
- A **refinement argument** relating your specification to your implementation, with the abstraction
  function written down and the places where the correspondence is weakest identified.
- **Property-based tests derived from the specification**, run against the implementation, closing
  the gap between the verified model and the running code.
- An **SMT-based analysis** of one real configuration question from CA-731 — a network reachability
  claim, an IAM policy's blast radius, or a capacity constraint.
- **Runtime assertions** for the invariants that could not be verified statically, deployed with
  the system.

The through-line of the term: CA-731 built a platform, ML-741 put a workload on it, and FM-751
provides the evidence that one of its critical protocols is right. "We tested it" and "we proved
the design satisfies these properties under these assumptions, and here is the counterexample it
found before we shipped" are different claims, and the difference is the point.

## Required reading

- **Lamport, *Specifying Systems* (2002)** — free from the author; the TLA+ book, and the best
  argument for specification as an engineering practice. Read Parts I and II.
- **Newcombe et al., "How Amazon Web Services Uses Formal Methods" (CACM 2015)** — the industrial
  case, honestly reported including the costs. Read it first.
- **Lamport, "The Temporal Logic of Actions" (TOPLAS 1994)** — for L03; hard, and the source.
- Hoare, "An Axiomatic Basis for Computer Programming" (CACM 1969) — six pages, and it founded the
  field.
- Clarke, Grumberg, Kroening, Peled & Veith, *Model Checking*, 2nd ed. — the reference for L04.
- Wayne, *Practical TLA+* and the Learn TLA+ material — the practical on-ramp; use alongside Lamport.
- de Moura & Bjørner, "Z3: An Efficient SMT Solver" (TACAS 2008), and the Z3 Python tutorial.
- Claessen & Hughes, "QuickCheck" (ICFP 2000) — where property-based testing comes from, and still
  the clearest statement of the idea.

## Recommended

- Lamport, *The TLA+ Video Course* — genuinely good, and faster than the book for getting started.
- Jackson, *Software Abstractions* (Alloy) — a different and complementary style of lightweight
  formal method; worth knowing even if you use TLA+.
- Hillel Wayne's writing on formal methods in industry, including the honest accounts of where it
  does not pay.
- Kingsbury, the Jepsen reports (with DS-701) — the empirical counterpart to this course's formal
  one; the two together are the argument.
- Nipkow, Klein, *Concrete Semantics* — if you want the mechanised-proof direction.
- Zave, "Using Lightweight Modeling to Understand Chord" (SIGCOMM CCR 2012) — a published protocol
  shown to be incorrect by specification, and a model of how to do this.

## A note on what this course claims

Formal methods are frequently oversold and frequently dismissed, and both positions are
unproductive. This course's claims:

- A model check establishes a property **of a model**, under **stated assumptions**, within **stated
  bounds**. It says nothing about your implementation unless you separately relate the two (L07),
  and it says nothing about assumptions you did not model.
- The primary value in practice is **finding bugs in designs**, not proving their absence — and the
  bugs found are disproportionately the subtle concurrency and failure-interleaving ones that
  testing cannot reach.
- A second, less-discussed value is **the specification itself**: the act of writing a design
  precisely enough to check reliably exposes ambiguities and unstated assumptions *before* the
  checker runs. Practitioners report this consistently, and it costs a fraction of the full effort.
- The cost is real: learning the notation, writing the spec, managing the state space. For most
  code, it is not proportionate. For a consensus protocol, a distributed transaction, a cache
  coherence scheme or an access control policy, it very often is.

Judging that proportionality is the tenth learning outcome and the most valuable thing this course
teaches.
