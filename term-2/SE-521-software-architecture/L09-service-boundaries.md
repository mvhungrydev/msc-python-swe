# SE-521 · Lesson 09 — Service Boundaries and the Distribution Decision

**Estimated study time:** 4 hours
**Prerequisites:** L05–L08

---

## 1. Orientation

Waldo, Wyant, Wollrath and Kendall wrote "A Note on Distributed Computing" in 1994, and the
industry has been relearning it ever since:

> "Objects that interact in a distributed system need to be dealt with in ways that are
> intrinsically different from objects that interact in a single address space... the
> difference is *not* one of degree."

A local call either returns or raises. A remote call may also: time out with the work
completed, time out with the work not started, succeed after the caller gave up, be
duplicated, arrive out of order, or return after the caller's process has restarted. That is
a different set of outcomes, and no framework can hide it — every attempt (CORBA, DCOM, RMI,
and every "just call it like a function" RPC layer since) has produced systems that work in
development and fail in production.

Distribution is therefore not a refactoring. It is a change in the *semantics* of every call
you moved.

## 2. Theory

### 2.1 What distribution costs

Stated plainly, because these costs are routinely underestimated:

- **Failure becomes partial.** In a monolith, a bug kills the request. In a distributed
  system, service B is down while A, C, and D work, and the system is in a state no test
  covered.
- **Latency compounds.** Ten sequential calls at p99 20 ms is *not* 200 ms at p99 — tail
  latencies multiply (Dean & Barroso, "The Tail at Scale"). If each of ten calls has a 1%
  chance of being slow, ~10% of requests hit at least one.
- **Consistency becomes eventual.** No cross-service transactions. Every invariant spanning
  services is a saga with compensations (L07 §2.6).
- **Debugging requires infrastructure.** Distributed tracing, correlation ids, log
  aggregation — none optional.
- **Deployment becomes coordination.** Version skew is permanent: at any moment, two
  versions of a service are running. Every change must be backward and forward compatible.
- **The organization changes.** On-call rotations, ownership, and interfaces between teams.
- **Local reasoning ends.** You can no longer read the code and know what happens.

Against these: independent deployment, independent scaling, technology diversity, fault
isolation (if you build it, which most do not), and — the real driver — **team autonomy**.

### 2.2 The honest reason people distribute

Most microservice migrations are not driven by scale. They are driven by **teams blocking
each other**: a shared codebase where every change requires coordination, a shared release
train, a shared database nobody can migrate.

That is a legitimate reason. It is also an *organizational* problem, and Conway's Law says
the architecture will mirror the communication structure whether you plan it or not. So the
correct sequence is: **decide the team boundaries and the ownership model first, then draw
the service boundaries to match.** Drawing service boundaries that cut across team
boundaries produces services that always change together, which has all the costs of
distribution and none of the autonomy.

The diagnostic question: *"Which changes currently require two teams to coordinate, and
would this boundary remove that?"* If a proposed split does not answer it, it is
decomposition for its own sake.

### 2.3 The modular monolith

The option that is usually correct and usually skipped.

A **modular monolith** has strict internal module boundaries — enforced by fitness functions
(L08) — with in-process calls. You get: clear ownership, enforced boundaries, the option to
extract later, and none of the distribution costs.

What it does *not* give you: independent deployment, independent scaling, fault isolation,
and technology diversity. Notice that only the first is usually the real requirement, and
it can often be addressed with better CI and trunk-based development (SE-511 L10) rather
than with a network.

The strategic argument: **a modular monolith is the correct precursor to microservices**,
because the hard part of microservices is finding the boundaries, and boundaries are cheap
to move inside a process and enormously expensive to move across a network. Get them right
in-process; extract only the ones that have proven stable and that have a concrete reason to
be independent.

This is the position of most experienced practitioners who have done both, and the
counter-argument worth engaging is that in-process boundaries decay without the physical
enforcement of a network. That is true if you do not enforce them — which is what L08 is
for.

### 2.4 Where to cut

If you are going to split, the boundary should be:

- **A bounded context** (L06 §2.4), not a layer and not an entity. "Order Service" and
  "Customer Service" split by noun are the classic mistake: every operation then needs both,
  and you have built a distributed monolith.
- **Aligned with a team.** §2.2.
- **Low-chatter.** If A calls B five times per request, they are one service with a network
  in the middle.
- **Independently changeable.** Measure it: from git history (L02 §2.5), how often do the
  two candidate halves change in the same commit? High co-change means they are one thing.
- **Owning its own data.** No shared database. A service that does not own its data is not
  independent — it is a stored procedure with HTTP.

**The distributed monolith** is the failure state: services that must be deployed together,
share a database, and call each other synchronously in long chains. It has every cost of
distribution and every cost of coupling. Its symptoms: a release requires ordering; a schema
change requires coordinating three teams; local development requires running eight services;
one service being down takes everything down.

### 2.5 Synchronous versus asynchronous

The most consequential per-boundary decision.

**Synchronous (request/response)** — simple to reason about, immediate consistency of the
response, and it propagates failure and latency directly. Every synchronous dependency
reduces your availability: if you call three services each at 99.9%, your ceiling is 99.7%,
and that assumes independence (which does not hold — shared infrastructure correlates
failures).

**Asynchronous (events/messages)** — the caller does not wait, failures are absorbed by the
broker, and the two services are temporally decoupled. Costs: eventual consistency,
at-least-once delivery requiring idempotent consumers (L07 §2.5), out-of-order arrival,
harder debugging, and a broker to operate.

The rule of thumb: **synchronous when the caller genuinely needs the answer to proceed;
asynchronous otherwise.** Most calls that look synchronous are not: "notify the customer",
"update the search index", "recalculate the recommendation" — none of these need to block
the response, and making them asynchronous both improves latency and removes an availability
dependency.

The variant worth knowing: **synchronous for queries, asynchronous for commands** is a
useful default. Reading someone else's data usually must be immediate; telling them
something happened usually need not be.

### 2.6 Resilience is not optional

Once you distribute, these stop being nice-to-haves:

- **Timeouts on every call.** A call with no timeout is a resource leak waiting for a slow
  dependency. Set them from measured p99s, not from round numbers.
- **Retries with backoff and jitter** (PY-501 L10 §3), and **only for idempotent
  operations**. Retrying a non-idempotent command is how duplicate charges happen.
- **Circuit breakers.** After N failures, stop calling and fail fast. This protects the
  *callee* (a struggling service is not helped by retries) as much as the caller.
- **Bulkheads.** Separate connection pools and thread pools per dependency, so that one slow
  dependency cannot exhaust the resources needed for the others.
- **Graceful degradation.** Decide, per dependency, what happens when it is down: fail,
  serve stale, serve a default, or queue. This is a *product* decision and should be made by
  someone who can answer it.
- **Idempotency keys** on every mutating endpoint, so that a retry is safe.
- **Backpressure.** A queue that grows without limit is a delayed outage.

Nygard's *Release It!* is the reference and the failure stories are worth reading in full;
this list is a summary, and DS-701 L09 treats the theory.

### 2.7 The decision procedure

Before splitting anything:

1. **What problem is this solving?** If the answer is not "teams block each other on X" or
   "X needs to scale independently by an order of magnitude" or "X has a different
   availability/compliance requirement", stop.
2. **Would a module boundary solve it?** Try that first, enforced (L08).
3. **Is the boundary a bounded context, aligned with a team?**
4. **What is the co-change rate** between the two halves, measured from history?
5. **What invariants would cross the boundary?** Each becomes a saga. List them and design
   the compensations *before* deciding.
6. **What is the chattiness?** Calls per request across the boundary.
7. **Who owns the data?** If the answer is "both", stop.
8. **What is the operational cost?** On-call, deployment, observability, local development.
9. **Can you reverse it?** Merging services is much harder than splitting a module.

Then, if you proceed: extract one service, run it for a quarter, and measure whether the
problem in step 1 actually went away *before* extracting the second. Almost nobody does
this, and it is the difference between a migration that succeeds and one that produces a
distributed monolith.

## 3. Construction: analysing a candidate split

Take a real system and a real proposed split.

**Step 1 — the problem statement.** One paragraph: what is painful today, who feels it, how
often, and what it costs. If you cannot write it, that is your finding.

**Step 2 — co-change analysis.** From git history, compute how often the two candidate
halves change in the same commit, over a year. Report the number. Above ~20%, they are one
thing and the split will produce coupled services.

**Step 3 — the call graph.** Instrument or trace one representative request. Count the calls
that would cross the proposed boundary. Above three per request, reconsider — or redesign
the boundary so the chatty calls are internal.

**Step 4 — the invariants.** List every invariant that would span the boundary (L07 §2.7).
For each: does it become a saga, an eventual guarantee with reconciliation, or does it
disappear? Design the compensation for each saga. This step frequently kills the proposal,
and that is a success.

**Step 5 — the data.** Which tables move? Are any read by both sides today? Design the
migration: expand/contract, dual write, backfill, cut over, contract. Estimate its duration
honestly — this is usually the largest single cost and it is usually omitted.

**Step 6 — the availability arithmetic.** Compute the composite availability of the
synchronous path after the split, and compare with today. State the independence assumption
explicitly and say why it does not hold.

**Step 7 — the alternative.** Design the modular-monolith version: the same boundary,
enforced by fitness functions, in-process. Estimate its cost and what it does not solve.

**Step 8 — recommend.** With the numbers. A recommendation of "not yet, do step 7 first, and
revisit when X" is the most common correct answer and should be argued as confidently as any
other.

## 4. Failure modes

- **Splitting by noun** (entity services) rather than by capability.
- **Shared database across services.** The single most common and most damaging error.
- **Distributed monolith.** §2.4.
- **Synchronous chains.** Latency and availability multiply.
- **No timeouts.** A slow dependency becomes a total outage.
- **Retrying non-idempotent operations.**
- **No circuit breakers**, so a struggling service is hammered until it dies.
- **Splitting before the boundary is proven** in-process.
- **Distributing to solve a scaling problem that is actually a query problem.** Measure
  first; a missing index has caused more microservice migrations than it should have.
- **Ignoring the organizational precondition.** Services without owners rot faster than
  modules without owners.
- **No plan to merge.** Splits are treated as irreversible; sometimes the right answer later
  is to merge two services back, and having said so in advance makes it possible.

## 5. Exercises

### Warm-up (25 min)

**W1.** Compute the composite availability of a synchronous path in a system you know. State
the independence assumption and give one reason it fails.

**W2.** Find a synchronous call that does not need to be. Describe what changes if it
becomes asynchronous — including what the caller can no longer promise.

**W3.** Find a call without a timeout. Determine what happens if the callee hangs.

### Core (2.5 h)

**C1 — The split analysis.** Complete §3, all eight steps, for a real candidate split.
Deliverable: the eight sections with numbers, and a recommendation you would defend in a
design review.

**C2 — Resilience audit.** For every outbound call in one service, record: timeout,
retry policy, idempotency, circuit breaker, bulkhead, and degradation behaviour. Report the
table. Fix the three worst gaps and write the test for each (a fake that hangs, one that
fails, one that is slow).

**C3 — Modular monolith.** Take a monolith without internal boundaries and impose them:
packages by bounded context, `import-linter` contracts, per-module ownership in CODEOWNERS,
and separate test suites. Report the violation count and what it revealed about the real
boundaries.

**C4 — Data migration plan.** For a split that would move tables, write the full
expand/contract plan: schema changes, dual-write period, backfill strategy, verification,
cutover, and rollback at each stage. Estimate the calendar time. Then find someone who has
done one and ask whether your estimate is realistic.

### Challenge

**X1.** Read Waldo et al. (1994), Dean & Barroso "The Tail at Scale" (2013), and Fowler's
"MicroservicePremium". Write 1,500 words on the claim that most microservice adoptions solve
an organizational problem with a technical mechanism, and that the technical mechanism has
costs the organizational problem does not justify. Give the strongest counter-case and
answer it, using measurements from C1.

**X2.** Simulate the tail-latency effect: build a service that fans out to N backends, each
with a realistic latency distribution including a 1% slow tail. Measure end-to-end p50, p99,
and p999 for N = 1, 3, 10, 30. Then implement hedged requests (send a second request after
the p95 deadline) and measure the improvement and the extra load. Report both curves and
state when hedging is worth it.

## 6. Self-check

1. State Waldo et al.'s claim and give three outcomes a remote call has that a local one does
   not.
2. Give six costs of distribution.
3. What is the honest reason most organizations distribute, and what does Conway's Law imply
   about the sequence?
4. What does a modular monolith give and not give? Why is it the right precursor?
5. Give five criteria for where to cut a service boundary.
6. State the sync/async rule and the query/command default.
7. Name six resilience mechanisms and what each protects.
8. Give the first three questions of the decision procedure.

## 7. Primary sources

- Waldo, Wyant, Wollrath & Kendall, "A Note on Distributed Computing" (Sun, 1994).
- Dean & Barroso, "The Tail at Scale" (CACM, 2013).
- Nygard, *Release It!*, 2nd ed. — the stability patterns.
- Fowler, "MicroservicePremium", "MonolithFirst", "MicroservicePrerequisites" (bliki).
- Newman, *Building Microservices*, 2nd ed., and *Monolith to Microservices*.
- Skelton & Pais, *Team Topologies*.

---

**Previous:** [L08](L08-evolutionary-architecture.md) · **Next:**
[L10 — Legacy Systems, Seams, and Architectural Decision Records](L10-legacy-and-adrs.md)
