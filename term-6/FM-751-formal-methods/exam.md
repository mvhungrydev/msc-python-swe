# FM-751 — Written Examination

**Time allowed: 3 hours. Closed book, no machine.**
**Answer FOUR from Section A and ONE from Section B.**
Section A: 15 marks each. Section B: 40 marks.

Proofs must cover every case; a proof with a missing case is an incomplete proof, not a proof with a
small error. Temporal formulas and Hoare triples must be written in the standard notation. Where a
question asks what a technique establishes, state the bounds and assumptions — an unqualified claim
scores in the lowest band.

---

## Section A — answer FOUR

**A1.** Give five purposes of a specification, and state the criterion that distinguishes a
specification from documentation. Then give the seven rungs of the precision ladder with an example
of each, and identify the most cost-effective formal method in routine use. *(15)*

**A2.** Explain what a specification must separate, and why a guarantee without its assumptions is
meaningless. Then explain why a specification should say what rather than how, and what is lost when
it does not. *(15)*

**A3.** Define state, behaviour, specification and satisfaction in the trace model. Explain why
"more specification" means "fewer allowed behaviours", and how non-determinism is expressed. *(15)*

**A4.** Give the Hoare rules for assignment, sequence, conditional, consequence and while. Explain
why the assignment rule runs backwards, and why the while rule's content is entirely the invariant.
*(15)*

**A5.** State the three conditions a loop invariant must satisfy. Explain how the third makes
finding one a constructive rather than creative act, and give the general shape of an array-loop
invariant. Then give the invariant of binary search. *(15)*

**A6.** Define a representation invariant. Explain what it turns debugging into, and state the
relationship between an invariant and a critical section in a concurrent data structure. *(15)*

**A7.** Define `wp` and give its rules for assignment, sequence and conditional. Explain why loops
are the one place the calculation cannot proceed mechanically, and what that implies about
verification tools in practice. *(15)*

**A8.** Define a variant and state the property of the ordering that makes the termination argument
work. Then map partial correctness and termination onto safety and liveness, and explain what
follows for how each is proved. *(15)*

**A9.** Give the four core temporal operators and define `⇝` in terms of them. Distinguish `□◇P`
from `◇□P` with an example where confusing them would matter. *(15)*

**A10.** Define safety and liveness by how each is violated. State the shape of the counterexample
in each case, and explain why liveness properties are the ones nobody tests. *(15)*

**A11.** Distinguish weak from strong fairness. State which should be assumed by default and why,
and give the hazard of assuming the stronger one. *(15)*

**A12.** State the Alpern–Schneider decomposition theorem and apply it to consensus, to transactions
and to CAP. Explain what expressing CAP this way makes clearer than the folklore statement. *(15)*

**A13.** Give the model checking algorithm for a safety property and explain why breadth-first
search is used. Then describe how liveness is checked and why its counterexamples take a different
shape. *(15)*

**A14.** State the state space explosion problem quantitatively for n processes of k states with a
channel of depth d over an alphabet of size a. Give six mitigations, identifying which requires
human judgement rather than tooling. *(15)*

**A15.** State the small scope hypothesis. Explain that it is empirical rather than proved, and give
its four practical consequences for how a check is run and reported. *(15)*

**A16.** Give the five-step procedure for reading a safety counterexample. Explain the two possible
causes of a counterexample and why both are useful outcomes. *(15)*

**A17.** Give five things model checking does not establish, and then state precisely what it does
establish, in the form you would use with a colleague. *(15)*

**A18.** Explain the standard shape `Init /\ [][Next]_vars /\ WF_vars(Next)`, term by term. Explain
why a primed variable is not an assignment and what `UNCHANGED` does. *(15)*

**A19.** Explain what labels mean in PlusCal, and what a label placement decision asserts about the
implementation being modelled. Give the concrete example of a lost update and how label placement
determines whether it appears. *(15)*

**A20.** Give six common TLA+ modelling errors with the signature of each. Explain in particular how
you check that a specification is not vacuous, and why a vacuous specification is dangerous. *(15)*

**A21.** Give the anatomy of a protocol specification. Explain why modelling the network as a set
gives reordering and duplication for free, and what modelling it as a per-channel sequence assumes.
*(15)*

**A22.** Define an auxiliary variable and give an example property that requires one. Then explain
why crash modelling must distinguish durable from volatile state, and what class of bug is missed
if it does not. *(15)*

**A23.** Define an inductive invariant. Explain why the property you want is usually not inductive,
give the four-step strengthening workflow, and explain why the resulting invariant is often more
valuable than the check itself. *(15)*

**A24.** Give six state space controls specific to protocol specifications, and state the form of an
honest results claim after a protocol model check. *(15)*

**A25.** Define refinement as implication, and state the two things that must hold for the statement
to be meaningful. Explain why stuttering invariance is a precondition, with an example. *(15)*

**A26.** Define a refinement mapping and give three examples of the abstraction function for
concrete systems. Explain what a failure to find a mapping usually indicates. *(15)*

**A27.** Distinguish history from prophecy variables, giving the situation that calls for each.
Explain why adding either is sound, and state the Abadi–Lamport completeness result and what it
tells you when a mapping eludes you. *(15)*

**A28.** Express linearizability as a refinement. Explain why linearizability composes and
sequential consistency does not, and why finding the linearization point of a lock-free algorithm is
the hard part. *(15)*

**A29.** Give the five options for closing the specification-to-code gap, ranked, with the cost and
rigour of each. Explain what trace validation catches that property-based testing does not. *(15)*

**A30.** State property-based testing as verification with exhaustiveness removed. Give its three
mechanisms and explain which one makes it usable in practice and why. *(15)*

**A31.** Give seven patterns for finding properties, with an example of each. Explain why metamorphic
relations matter when no oracle exists. *(15)*

**A32.** Explain model-based stateful testing and its relationship to refinement. Give the four-step
workflow from a specification to a property test, and explain why model checker counterexamples
should become implementation tests. *(15)*

**A33.** Compare model checking and property-based testing on artifact, coverage, cost, what each
finds, counterexample shape, and CI suitability. State which you would adopt first for a team with
neither, and why. *(15)*

**A34.** Explain why SAT is NP-complete and yet industrial instances with millions of variables are
solved routinely. Name the key innovation and state the general lesson about complexity classes.
*(15)*

**A35.** Give six SMT theories with an application of each. Explain why bitvectors rather than
integers are required to reason about machine arithmetic, using the binary search overflow as the
example. *(15)*

**A36.** Give the three answers an SMT solver returns and the idiom for proving a property. Explain
what an `unsat` core is and why it turns a solver into a usable tool. Then explain why treating
`unknown` as `unsat` is the worst available error. *(15)*

**A37.** Give six rules for encoding SMT problems well, and the two validation checks every encoding
needs. Explain why a trivially-unsatisfiable encoding is the most dangerous failure mode. *(15)*

**A38.** Distinguish precondition, postcondition and invariant by fault attribution. Give the
contract substitution rules and explain how they restate the Liskov substitution principle. *(15)*

**A39.** Explain why safety properties can be monitored at runtime and liveness properties cannot,
and give the practical substitute. Then give four responses to a runtime violation and the rule for
choosing between them. *(15)*

**A40.** Give the decision procedure for choosing a verification technique, in order. Then give this
course's recommended defaults, and identify the single highest-value application of model checking
for a working engineer. *(15)*

---

## Section B — answer ONE

**B1. Specify and check.** You are designing a distributed lease service: clients acquire a
time-bounded exclusive lease on a resource, and a lease may be renewed or may expire. Clients may
crash. Messages may be lost, duplicated and reordered. Clocks are not perfectly synchronised.

Write the specification and the verification plan. Your answer must: state the abstract
specification of what a lease provides, in ten lines or fewer; state the safety property precisely
(at most one holder) and the liveness property, distinguishing them; describe how you would model
clocks and time, and justify the choice — including the option of not modelling time at all; give the
failure model as a list of actions, and state explicitly what you exclude; describe the invariant you
expect to need and why the safety property alone will not be inductive; describe the fencing
mechanism required for the safety property to survive clock skew and explain why the timeout alone
is insufficient; state the configuration you would check and the honest results claim you could
make; and describe how you would connect the specification to the implementation. *(40)*

**B2. The counterexample.** A model check of a two-phase commit specification produces this
counterexample: the coordinator sends `PREPARE`; both participants vote `YES` and durably record the
vote; the coordinator logs `COMMIT`; the coordinator crashes before sending anything; participant 1
times out and is stuck; an operator issues a heuristic `ABORT` to participant 1; the coordinator
recovers and sends `COMMIT`; participant 2 commits.

Write the analysis. Your answer must: state which property is violated and what the resulting system
state is; explain why this is not an implementation bug but a property of the protocol under the
stated assumptions; identify precisely which assumption the heuristic decision violates and what
that means for the specification's assumptions section; explain the relationship to FLP and to the
Skeen–Stonebraker blocking result; describe the three design responses (Paxos Commit, sagas,
avoiding the distributed transaction) and what each gives up; state what the specification should
have said about heuristic decisions; and explain how you would turn this counterexample into a test
of an implementation. *(40)*

**B3. The verification strategy.** You lead a team of eight engineers building a multi-tenant
platform with a custom control plane, a replicated metadata store and a per-tenant authorisation
model. Nobody on the team has used formal methods. You have a quarter to demonstrate value or the
effort will be cut.

Write the strategy. Your answer must: rank the components by the cost of being wrong and by the
reversibility of their design decisions; select techniques per component with an estimate in
engineer-days and a statement of what each would establish; identify what you would explicitly not
verify and why; state which single artifact you would produce first to demonstrate value within the
quarter, and why that one; describe how the artifacts stay current as the system changes, addressing
the stale-specification problem directly; state who maintains them and what happens when that person
leaves; give the argument you would make to a sceptical engineer and the concessions you would make
to keep it credible; and state what evidence would tell you the effort is not paying and should be
stopped. *(40)*

**B4. The gap.** A team has a TLA+ specification of their replication protocol, model-checked for
five safety properties at three nodes, with no violations found. They have a production incident:
an acknowledged write was lost.

Write the investigation. Your answer must: enumerate systematically the ways this is possible
despite the model check, covering the model, the properties, the bounds, the assumptions and the
implementation; explain how you would determine which of these applies, in order, and what evidence
each would produce; describe how trace validation would help and what instrumentation it requires;
describe how property-based testing derived from the specification would help and what it would
catch that the model check did not; state what should change in the specification, the testing and
the runtime monitoring as a result; and write the revised evidence statement the team should be able
to make afterwards — including what it does not claim. *(40)*

**B5. Choose the evidence.** For each of the following five components, choose the verification
technique or techniques you would apply, justify the choice against the cost of being wrong, and
state precisely what your chosen evidence would establish and what it would leave unverified:
(a) a JSON parser handling untrusted input from the internet; (b) a distributed lock service used by
every service in the company; (c) an IAM-style policy evaluation engine; (d) an LSM-tree compaction
routine; (e) a UI component that formats currency values.

Your answer must treat all five, and must: apply the decision procedure explicitly rather than
asserting a conclusion; identify the one where you would apply the most expensive technique and the
one where you would apply almost none, with the reasoning in both cases; identify a component where
two techniques are complementary and say what each contributes; state, for the component with the
highest residual risk after your chosen techniques, what remains unverified and what you would do
operationally about it; and state, in one paragraph, the general principle your five decisions
illustrate. *(40)*

---

*Marks in Section A are awarded for precision, for complete proofs, and for stating the bounds and
assumptions of any claim about what a technique establishes. In Section B, an answer that claims
more than its evidence supports cannot reach the upper band, however sophisticated the technique
proposed; and an answer that does not say what remains unverified has not finished.*
