# Appendix — Glossary

Terms defined once, with the lesson that develops them. Where a term is used differently in
different fields, both senses are given, because the collisions cause real confusion — "consistency"
means three different things in this programme alone.

---

## A

**ACID** — Atomicity, Consistency, Isolation, Durability. Note that the C is the application's
responsibility as much as the database's (DI-721 L04).

**Action** — In TLA+, a predicate on a *pair* of states, relating current to next. Not a statement.
(FM-751 L05)

**Aggregate** — In DDD, a cluster of objects treated as one unit of transactional consistency; the
boundary within which invariants hold synchronously. (SE-521 L07)

**Amortised analysis** — Bounding the average cost over a sequence of operations, not each one.
Distinct from *expected* (over randomness) and *worst-case* (over every input). (CS-621 L01)

**Amplification (write / read / space)** — Bytes written to storage ÷ logical bytes written; storage
reads ÷ logical reads; bytes stored ÷ live bytes. Every storage engine trades these. (DI-721 L02)

**Assumption** — What a specification takes for granted about its environment. A guarantee without
its assumptions is meaningless. (FM-751 L01)

**At-least-once / at-most-once / exactly-once** — Delivery semantics. Exactly-once *delivery* is
impossible; exactly-once *effect* is achieved by idempotent receivers. (DS-701 L01, L08)

## B

**Backpressure** — Signalling upstream to slow down when a queue is filling; the alternative is
unbounded growth then collapse. (PY-601 L09)

**Bandit feedback** — You observe the outcome only of the action taken, never of the actions not
taken. The root of ML feedback-loop problems. (ML-741 L08)

**Behaviour** — An infinite sequence of states. A specification is a predicate on behaviours.
(FM-751 L01)

**Bloom filter** — Probabilistic set membership with no false negatives. Makes LSM point lookups
viable; does nothing for range scans. (DI-721 L02)

**Blast radius** — What one failure or one mistaken action can damage. (CA-731 L05)

**Bulkhead** — Partitioned resources so one dependency's failure cannot consume all of them.
(DS-701 L09)

**Burn rate** — How fast an error budget is being consumed relative to spending it evenly.
(CA-731 L07)

## C

**CACE** — Changing Anything Changes Everything. Because a model jointly optimises over all inputs,
changing one changes the model's behaviour everywhere. (ML-741 L01)

**CALM** — Consistency As Logical Monotonicity: a program has a coordination-free implementation iff
it is monotonic. (DS-701 L07)

**CAP** — Under a network partition you must choose availability or consistency. Formally: a certain
safety property and a certain liveness property cannot both hold. (DS-701 L04)

**Causal consistency** — Operations related by happens-before are seen in that order by everyone;
concurrent ones may be seen in any order. (DS-701 L04)

**CDC (change data capture)** — Turning a database's change log into an event stream. Log-based CDC
reads the replication log; polling misses deletes and intermediate states. (DI-721 L09)

**Circuit breaker** — Fails fast when a dependency is failing, protecting the *caller's* resources.
Trip on rates, not consecutive counts. (DS-701 L09)

**Coherency / consistency** — Three distinct uses in this programme: the C in ACID (application
invariants); distributed consistency (ordering across replicas, DS-701 L04); and cache coherence.
Keep them separate.

**Compaction** — Merging LSM files, discarding superseded versions. Levelled vs size-tiered is a RUM
choice. (DI-721 L02)

**Concept drift** — P(y|X) changes: the relationship itself moved. Undetectable from inputs.
(ML-741 L06)

**Connascence** — A taxonomy of the ways two pieces of code must change together; a finer-grained
coupling vocabulary. (SE-521 L02)

**Consensus** — Agreement, validity, termination among processes. FLP says all three are impossible
in an asynchronous system with one faulty process. (DS-701 L05)

**Constant work** — Designing a component so its workload does not change with conditions, so the
failure path is the same code path as the normal one. (CA-731 L01)

**Contract (data)** — An explicit, versioned, enforced agreement between a data producer and its
consumers. (DI-721 L10)

**Contract (code)** — Precondition, postcondition, invariant. Violation attributes fault to caller
or callee. (FM-751 L10)

**Control plane / data plane** — Configuration changes versus request serving. Control planes fail
more often, so recovery must not depend on them. (CA-731 L01)

**Counterexample** — A concrete execution violating a property. Safety: a finite trace. Liveness: a
lasso. (FM-751 L04)

**Covariate shift** — P(X) changes while P(y|X) does not. (ML-741 L06)

**CRDT** — A data type whose merge is commutative, associative and idempotent, giving strong
eventual consistency with no coordination. (DS-701 L07)

**CQRS** — Separating the write model from the read model. Independent of event sourcing, though
often paired. (DI-721 L09)

## D

**Descriptor** — An object defining `__get__`/`__set__`/`__delete__`; the mechanism behind
properties, methods and slots. (PY-501 L03)

**Drift** — See covariate shift, label shift, concept drift. Also "configuration drift" (CA-731
L05): divergence between declared and actual infrastructure.

**Durability** — Committed data survives a stated class of failure. Always relative to a failure
model. (DS-701 L03)

## E

**Error budget** — 1 − SLO. A resource to spend, not a limit to avoid. (CA-731 L07)

**Eventual consistency** — Replicas converge if updates stop. Weaker than strong eventual
consistency, which requires no conflict resolution. (DS-701 L04, L07)

**Event sourcing** — Events are the state; current state is derived by folding them. (DI-721 L09)

**Exploration** — Deliberately taking actions the model would not choose, to generate data about
them. The structural fix for feedback loops. (ML-741 L08)

## F

**Fairness (weak / strong)** — WF: a continuously enabled action eventually occurs. SF: an
infinitely-often enabled action eventually occurs. Required for any liveness property. (FM-751 L03)

**Fencing token** — A monotonically increasing number checked at the resource, so a stale
lease-holder is rejected. Safety must not depend on a timeout. (DS-701 L03)

**FLP** — Fischer, Lynch, Paterson: no deterministic consensus algorithm in an asynchronous system
tolerates even one crash failure. (DS-701 L01)

**Free-threaded** — CPython without the GIL (PEP 703). (PY-601 L02)

## G

**GIL** — CPython's global interpreter lock. (PY-601 L02)

**Golden path** — A supported, opinionated, complete route through a common task. Paved, not
mandatory. (CA-731 L09)

**Grain** — What one row of a fact table represents. Mixed grain produces double-counted aggregates.
(DI-721 L10)

**Guardrail** — A policy setting a maximum boundary regardless of individual grants; more valuable
than fine-grained permissions. (CA-731 L03)

## H

**Happens-before** — The partial order induced by program order and message causality. Events
unrelated by it are concurrent. (DS-701 L02)

**Hedged request** — A second copy sent after a delay, taking whichever answers first; trades ~5%
load for a much better tail. Requires idempotence and cancellation. (DS-701 L09)

**HLC (hybrid logical clock)** — Preserves causality like a logical clock while staying close to
physical time. (DS-701 L02)

**HOT update** — A Postgres update that avoids touching indexes when no indexed column changed and
the new version fits the page. (DI-721 L03)

## I

**Idempotence** — Applying more than once has the same effect as once. The receiver-side
construction of exactly-once effect. (DS-701 L08)

**Inductive invariant** — True initially and preserved by every step *assuming only itself*. The
property you want usually is not one, and must be strengthened. (FM-751 L06)

**Invariant** — A property that always holds: of a loop between iterations, of a data structure
between operations, of a system in every reachable state. (FM-751 L02, L03)

**Isolation level** — What concurrency anomalies a database permits. Engine behaviour rarely matches
the standard's names. (DI-721 L04)

## L

**Label delay** — Time from a prediction to its ground truth. Bounds detection time and retraining
cadence. (ML-741 L02)

**Lamport clock** — A counter giving `a → b ⇒ L(a) < L(b)`; the converse does not hold. (DS-701 L02)

**Leakage** — A feature carrying information unavailable at prediction time. Inflates offline
metrics, so it looks like success. (ML-741 L02)

**Level-triggered** — Acting on current state rather than on an event. Robust to lost, duplicated
and reordered events. (CA-731 L06)

**Linearizability** — Every operation appears to take effect atomically at some point between its
invocation and response, consistent with real time. Composable. (DS-701 L04)

**Liveness** — "Something good eventually happens." Violated only by an infinite behaviour.
(FM-751 L03)

**Little's Law** — L = λW: concurrent items = arrival rate × time in system. (PY-601 L09)

**LSM-tree** — Log-structured merge tree: memtable, immutable sorted files, background compaction.
(DI-721 L02)

## M

**Metastable failure** — A degraded state sustained by a feedback loop after the trigger is gone.
Diagnostic: removing the trigger does not fix it. (DS-701 L09)

**MRO** — Method resolution order; CPython uses C3 linearisation. (PY-501 L04)

**MVCC** — Multi-version concurrency control: readers see a snapshot, so readers never block
writers. Cost: garbage collection pinned by the oldest transaction. (DI-721 L05)

## O

**Outbox (transactional)** — Writing an event into a table in the same transaction as the business
change, with a relay publishing it. Eliminates the dual-write problem. (DS-701 L08)

## P

**PACELC** — If Partitioned, choose Availability or Consistency; Else, choose Latency or
Consistency. (DS-701 L04)

**PAX** — Row groups with column-wise layout inside; the basis of Parquet. (DI-721 L06)

**Point-in-time correctness** — Features computed using only data available at the event's
timestamp. (ML-741 L02)

**Property (verification)** — A universally quantified claim about behaviour. Safety or liveness.
(FM-751 L01)

**Propensity** — The probability the logging policy assigned to the action taken. Must be recorded
for off-policy evaluation; cannot be retrofitted. (ML-741 L08)

## Q

**Quorum** — A set large enough that any two intersect; R + W > N. (DS-701 L03)

## R

**Refinement** — Every behaviour of the implementation is permitted by the specification. In TLA+,
implication. (FM-751 L07)

**Refinement mapping** — The abstraction function from concrete states to abstract ones. Its
non-existence usually means a bug. (FM-751 L07)

**Representation invariant** — A property of a data structure's internal state, true whenever no
operation is in progress. (FM-751 L02)

**RUM conjecture** — Read, Update, Memory: optimise at most two. (DI-721 L02)

## S

**Saga** — A sequence of local transactions with compensating transactions. ACD, not ACID — no
isolation. (DS-701 L08)

**Safety** — "Something bad never happens." Violated by a finite prefix. (FM-751 L03)

**Sargable** — A predicate an index can serve. `YEAR(col) = 2024` is not. (DI-721 L03)

**SCD (slowly changing dimension)** — Type 1 overwrites (loses history); Type 2 adds a row with
validity dates. (DI-721 L10)

**Shuffle sharding** — Assigning each tenant a random *subset* of workers, so one bad tenant affects
a small computable fraction of others. (CA-731 L02)

**Skew (data)** — One key holding a large share of rows; the dominant failure of distributed joins.
(DI-721 L07)

**Skew (training/serving)** — Features computed differently in training and serving. The most common
silent ML failure. (ML-741 L01)

**SLI / SLO / SLA** — Measured ratio / internal target / contractual commitment. (CA-731 L07)

**SMT** — Satisfiability modulo theories: SAT plus arithmetic, bitvectors, arrays, and more.
(FM-751 L09)

**Static stability** — Pre-provisioning so recovery requires no control-plane action. (CA-731 L01)

**Strong eventual consistency** — Replicas that received the same updates have the same state,
regardless of order, with no conflict resolution. (DS-701 L07)

**Stuttering** — A step that changes nothing. Permitting it is what makes refinement work.
(FM-751 L03)

## T

**Tombstone** — A marker that a key was deleted; cannot be collected until compacted against every
older file that could hold the key. (DI-721 L02, DS-701 L07)

**Toil** — Manual, repetitive, automatable work that scales with the service and has no enduring
value. (CA-731 L07)

**Trace validation** — Checking a running implementation's recorded state trace against a
specification. (FM-751 L07)

## U

**Unsat core** — The minimal contradictory subset of assertions; turns a solver's verdict into a
diagnosis. (FM-751 L09)

**USL** — Universal Scalability Law: throughput limited by a serial fraction *and* a coherency term,
so it peaks and then declines. (PY-602 L09)

## V

**Variance (types)** — Covariant, contravariant, invariant: how a type constructor's subtyping
relates to its parameters'. Mutable containers must be invariant. (CS-641 L05)

**Variant (termination)** — An expression decreasing on every iteration and bounded below.
(FM-751 L02)

**Vector clock** — A per-node counter vector; detects concurrency, which Lamport clocks cannot.
(DS-701 L02)

## W

**WAL** — Write-ahead log: the log record reaches stable storage before the data page. Turns random
durable writes into sequential ones. (DI-721 L01)

**Watermark** — An assertion that no more events with timestamps below W are expected. A heuristic.
(DI-721 L08)

**Weakest precondition** — `wp(S, Q)`: the weakest P such that S terminates in a state satisfying Q.
The basis of program verifiers. (FM-751 L02)

**Write skew** — Two transactions read an overlapping set, check an invariant, and write *different*
rows, violating it. Permitted by snapshot isolation. (DI-721 L04)
