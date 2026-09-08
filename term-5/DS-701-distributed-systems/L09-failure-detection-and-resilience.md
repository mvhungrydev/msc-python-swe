# DS-701 · Lesson 09 — Failure Detection, Timeouts, and Resilience

**Estimated study time:** 4 hours
**Prerequisites:** L01, L06; PY-602 L01 (measurement statistics), PY-601 L09 (backpressure)

---

## 1. Orientation

L01 established the theoretical position: in an asynchronous system you cannot distinguish a
crashed node from a slow one, and therefore perfect failure detection is impossible. This lesson
is about what you build anyway.

The practical framing is that **a timeout is a decision, not an observation.** When a call times
out you have not learned that the remote work failed; you have decided to stop waiting. The
remote side may be halfway through charging a credit card. Every piece of resilience machinery in
this lesson — retries, circuit breakers, hedging, bulkheads — is downstream of that one sentence,
and every one of them is dangerous if you forget it.

The second framing, which is what separates people who have operated systems from people who have
only read about them: **most large outages are not caused by a component failing. They are caused
by the system's response to a component failing.** A database gets slow; clients time out; clients
retry; the retries triple the load; the database gets slower; now everything is down and staying
down even after the original trigger is gone. The resilience patterns exist to break those loops,
and misconfigured, they *are* those loops.

## 2. Theory

### 2.1 Failure detectors as an abstraction

Chandra and Toueg's contribution was to stop treating failure detection as an implementation
detail and make it a first-class, formally specified module. A failure detector is characterised
by two properties:

- **Completeness**: every crashed process is eventually suspected.
- **Accuracy**: correct processes are not suspected (in various strengths).

The important class is **◇S — eventually strong**: strong completeness, plus *eventual* weak
accuracy (eventually some correct process is never suspected). The headline result: **◇S is the
weakest failure detector sufficient to solve consensus** with a majority of correct processes.
That is precisely the theoretical justification for what Raft does with randomised election
timeouts (L05) — it is implementing a ◇S detector, and its correctness never depends on the
timeouts being *right*, only on them eventually being stable.

Take from this the design principle that runs through the whole lesson:

> **Safety must never depend on a timeout. Only liveness may.**

If a timeout being wrong can cause two leaders to both commit, your design is broken; if a timeout
being wrong causes a spurious election and a few hundred milliseconds of unavailability, your
design is fine. The mechanism that enforces this in practice is the fencing token from L03/L06:
the lease-holder's *identity* is checked at the resource, so a node that wrongly believes it still
holds a lease is rejected rather than trusted.

### 2.2 Choosing a timeout

Every timeout embeds a bet about the latency distribution. The naive approach — pick a round
number, usually 30 seconds, usually copied from somewhere else — fails in both directions: too
long and a dead dependency consumes your threads and your users' patience; too short and you kill
healthy slow requests and *add* load through retries.

A defensible timeout comes from measurement (PY-602 L01):

1. Measure the dependency's latency distribution under *realistic* load, and look at high
   percentiles, not the mean. Latency distributions in distributed systems are heavy-tailed;
   the mean tells you almost nothing.
2. Set the timeout above the p99.9 of the *healthy* distribution, with headroom — not at some
   percentile of the current distribution, which is contaminated by exactly the misbehaviour you
   are trying to detect.
3. **Budget it downward through the call chain.** If the user-facing request has a 2-second
   budget and calls A which calls B, then B's timeout must be strictly less than A's remaining
   budget, which must be less than 2 seconds. Propagate the *deadline* (an absolute time), not
   the timeout (a duration) — gRPC's deadline propagation is the model. A deadline is
   composable; a duration re-starts the clock at every hop and lets a three-hop chain consume
   three times the budget.
4. **Cancel on deadline expiry, all the way down.** A timeout that abandons the caller without
   cancelling the downstream work is how you accumulate orphaned work that is still consuming
   the resources you need to recover. This is PY-601 L07's cancellation discipline applied across
   the network.

Adaptive timeouts (φ-accrual, §2.3) are the answer when the distribution shifts over time, which
in a cloud environment it always does.

### 2.3 The φ-accrual failure detector

The classical detector answers a Boolean question: is the node up? The **φ-accrual** detector
(Hayashibara et al., 2004) answers a continuous one: *how suspicious should I be right now?*

It keeps a sliding window of recent heartbeat inter-arrival times, fits a distribution (normally
an exponential or normal), and outputs

    φ(t) = −log10( P(heartbeat arrives later than the time since the last one) )

φ = 1 means about a 10% chance this delay is normal; φ = 8 means about 10⁻⁸. Downstream code
picks its own threshold, and different consumers can pick different ones — a routing layer might
act at φ = 3 while a destructive failover acts at φ = 12.

Two reasons this is the right shape:

- It **adapts** to the observed network. A detector tuned in a datacentre stays sane across a WAN.
- It **decouples** detection from policy. "How suspicious am I" is a measurement; "what do I do at
  this suspicion level" is a decision, and they belong to different components.

Akka and Cassandra both ship φ-accrual detectors; it is worth reading Cassandra's, which is short.

### 2.4 Retries, and how they cause outages

Retries are the first thing anyone adds and the most common cause of the metastable failures in
§2.7. The rules:

- **Retry only what is safe to retry.** Idempotent operations (L08), or non-idempotent ones with
  an idempotency key. Retrying a non-idempotent operation is a correctness bug, not a resilience
  feature.
- **Retry only what is worth retrying.** A 400 will be a 400 next time. Distinguish *retriable*
  (connection refused, 503, timeout) from *terminal* (4xx other than 429) — and note that a
  timeout is ambiguous, which is why the idempotency key matters.
- **Exponential backoff with full jitter.** `sleep = random(0, min(cap, base × 2^attempt))`. The
  jitter is not a refinement; without it, all clients that failed at the same instant retry at
  the same instant, forever. AWS's "Exponential Backoff and Jitter" post has the simulation, and
  full jitter wins.
- **Bound the total attempts and the total elapsed time**, and honour the deadline from §2.2: a
  retry that cannot complete within the remaining budget should not be issued at all.
- **Budget retries system-wide, not per-call.** The critical technique: allow retries only while
  the *ratio* of retries to original requests stays under some fraction (10% is the usual number),
  measured over a sliding window. This is the **retry budget** or **adaptive throttling**, and it
  is what caps retry amplification in a deep call graph. Without it, three layers each retrying
  three times means 27× load on the bottom layer at exactly the moment it is least able to take it.
- **Retry at one layer.** Retries at layers 1, 2 and 3 multiply. Pick the layer with the best
  context — usually the highest one that can still tell retriable from terminal — and make the
  others pass failures through.

### 2.5 Circuit breakers, bulkheads, and load shedding

**Circuit breaker.** A state machine wrapping a dependency: `CLOSED` (calls pass; count
failures), `OPEN` (calls fail immediately without trying), `HALF-OPEN` (let a limited number of
trial calls through; on success close, on failure re-open). Its purpose is *not* primarily to
protect the callee — it is to stop the caller from spending its own threads, connections and
latency budget on calls that are going to fail, and to fail fast so a fallback can run. Trip on
*rates* over a window, never on consecutive counts, and require a minimum request volume before
the rate is meaningful. The pitfalls: a half-open state that lets a thundering herd through
(limit it to one or a few probes), and a breaker around a *partially* failing dependency, where
tripping it turns a 20% failure into a 100% one.

**Bulkhead.** Partition resources so one dependency's problem cannot consume all of them: a
separate connection pool or a bounded concurrency limiter per dependency. Named after ship
compartments; the failure it prevents is the classic one where a single slow downstream
dependency occupies every worker thread and takes down endpoints that never touch it.

**Load shedding.** When you cannot serve everything, reject early and cheaply rather than degrading
for everyone. Shed at the edge (cheapest to reject), shed by priority (health checks and
retryable background work go first), and shed *before* queues grow — a request that has waited
past its deadline in a queue should be dropped rather than served, since serving it does nobody
any good and consumes capacity. This is Little's Law and the queueing material from PY-601 L09
applied at the service boundary.

**Concurrency limits over rate limits.** A concurrency limit (at most N in flight) self-adjusts
to the dependency's actual speed in a way that a requests-per-second limit does not: if the
dependency slows, in-flight requests linger and the limiter naturally admits fewer. Netflix's
adaptive concurrency limits use a TCP-congestion-control-style algorithm (gradient of observed
latency versus a measured minimum) to find the limit automatically.

### 2.6 Hedging and the tail

Dean and Barroso's "The Tail at Scale" makes the arithmetic vivid: if one server's p99 is 1
second, a request that fans out to 100 servers and needs all of them has a *median* latency near
that 1 second. Tail latency is not a rare event at scale; it is the common case.

**Hedged requests**: send the request; if no response by, say, the p95, send a second copy to a
different replica and take whichever answers first. This converts a small amount of extra load
(about 5%) into a dramatic tail reduction. Requirements: the operation must be idempotent, and
you must cancel the loser — an uncancelled hedge is pure added load.

**Tied requests**: send to two replicas at once, each told the identity of the other; whichever
starts work first tells the other to drop it. Cheaper than hedging in tail terms, and what BigTable
does.

The general principle is worth stating on its own: **at scale, redundancy in *requests* buys you
latency the same way redundancy in *machines* buys you availability.**

### 2.7 Metastable failure

The most important operational concept in this lesson, and the one least covered in textbooks.

> A system is in a **metastable failure state** when it remains in a degraded, self-sustaining bad
> state *after the trigger that caused it has been removed*.

The structure is always the same: a **trigger** (a load spike, a deploy, a dependency blip) pushes
the system into a state where a **sustaining effect** — usually a work-amplifying feedback loop —
keeps it there. Retries are the canonical sustaining effect: load rises, latency rises, clients
time out and retry, effective load rises further. Cache-fill storms after a cache flush, connection
storms after a restart, and GC death spirals are the same shape.

The diagnostic property, and the reason this concept earns its name: **removing the trigger does
not fix it.** Traffic returns to normal and the system stays down. The fixes are structural:

- Cap the amplification (retry budgets, §2.4).
- Add a way to shed load so the system can drain (§2.5).
- Give operators an explicit **drain and restart** procedure — sometimes the only exit is to shed
  nearly all load until the queue clears, then admit traffic gradually.
- Do not restart everything at once, which produces its own thundering herd on cold caches.

Bronson et al.'s "Metastable Failures in Distributed Systems" (HotOS 2021) is short and should be
read in full; it gives you the vocabulary to say, in an incident review, *why* the system did not
recover on its own.

## 3. Construction: a resilience layer, measured

Build in `mpse/ds701/l09/`, on top of the L01 harness. The rule for this lesson: **every pattern
must be demonstrated by a measurement that shows it working, and by a measurement that shows it
misconfigured making things worse.** A resilience library you have not seen fail is a resilience
library you do not understand.

**Stage 1 — a controllable dependency.** Write a fake service whose latency is drawn from a
configurable distribution (log-normal with an injectable tail), whose error rate you can set, and
which can be told to become slow at time *t*. Add a load generator with a fixed arrival rate, and
record per-request latency and outcome.

**Stage 2 — the baseline outage.** No timeouts, no limits. Make the dependency slow at t = 30s.
Plot in-flight requests, latency percentiles and throughput. You should see thread exhaustion and
a system that does not recover when the dependency heals at t = 60s. This is your metastable
failure, reproduced on your laptop.

**Stage 3 — timeouts and deadlines.** Add a per-call timeout, then a propagated deadline across a
three-hop chain. Show that the deadline version does not let the chain consume 3× the budget.
Verify that a timed-out call actually cancels the downstream work (count the work the fake service
still performs after the caller has given up).

**Stage 4 — retries, done badly first.** Add fixed-interval retries with no budget. Re-run
Stage 2 and show that the outage is now *worse* and lasts longer. Then add full jitter, then a
retry budget, and plot the three curves on one chart. This chart is the deliverable.

**Stage 5 — the circuit breaker.** Implement `CLOSED`/`OPEN`/`HALF-OPEN` with rate-based tripping,
a minimum volume threshold and a limited half-open probe. Show recovery time versus no breaker.
Then construct the failure case: a dependency failing 20% of the time, where a badly tuned breaker
converts partial availability into none.

**Stage 6 — bulkheads and adaptive concurrency.** Add a per-dependency concurrency limiter. Show
that a slow dependency A no longer affects an endpoint that calls only B. Then implement a simple
gradient-based adaptive limit and compare it to a fixed limit under a shifting latency profile.

**Stage 7 — hedging.** Add hedged requests at the p95 with cancellation of the loser. Measure p50,
p99 and total request volume with and without. Then remove the cancellation and measure the added
load, so you can see what the "5% extra" becomes when the hedge is not cleaned up.

**Stage 8 — φ-accrual.** Implement the detector over your L06 heartbeats. Plot φ over time as you
inject a slow network, a partition, and a genuine crash, and mark where a fixed 5-second timeout
would have fired versus where thresholds of φ = 3 and φ = 8 fire. Write the paragraph explaining
which you would use for routing and which for failover, and why.

## 4. Failure modes

- **Timeouts with no cancellation.** The caller gives up, the callee keeps working, and the
  resources you need to recover are held by work nobody is waiting for.
- **Timeouts that do not decrease down the call chain.** An inner timeout longer than the outer
  one can never fire usefully; you have simply guaranteed the outer one fires first.
- **Retrying non-idempotent operations.** Duplicate charges. The most expensive bug in this
  lesson.
- **Per-call retry limits with no system-wide budget.** Amplification is multiplicative across
  layers; per-call limits do not bound it.
- **Backoff without jitter.** Synchronised retry waves that keep the system down.
- **Circuit breakers tripping on consecutive failures.** Noisy, and blind to a low-volume endpoint;
  use failure *rate* over a window with a minimum volume.
- **A half-open state with no concurrency limit.** The recovery attempt becomes the next outage.
- **Health checks that only check the process.** A process that is alive but cannot reach its
  database should fail its readiness check. Conversely, a *deep* health check that fails when a
  non-critical dependency is down takes down a service that could have degraded gracefully — check
  what you actually require.
- **No load shedding.** Without it, an overloaded system serves everyone badly instead of most
  people well, and cannot drain.
- **Treating an incident as "the database was slow".** If the system did not recover when the
  database recovered, the database was the trigger and something else was the sustaining effect.
  Find it, or it will happen again.

## 5. Exercises

### Warm-up (30 min)

1. Explain why safety must not depend on a timeout, and give a concrete example of a design that
   violates this and the corruption it produces.
2. You have a 2-second user-facing budget and a chain A → B → C. Assign timeouts, and explain
   what changes if you propagate deadlines instead.
3. Three layers each retry 3 times with no budget. How many requests reach the bottom layer in the
   worst case, and what is the retry budget's effect on that number?

### Core (3 h)

4. Complete Stages 1–4. Deliver the single chart from Stage 4 with a paragraph of commentary.
5. Complete Stages 5–6. For the circuit breaker, include the misconfiguration case and state the
   tuning rule you derived from it.
6. Take a real service you work with and write its **timeout and retry policy**: every outbound
   call, its deadline, whether it is retriable and why, where the retry budget lives, and what
   the fallback is when it fails. Most teams have never written this down, and writing it down is
   most of the value.
7. Write a **metastable failure analysis** of an outage you have experienced or read a public
   postmortem of: identify the trigger, the sustaining effect, and the structural change that
   would have prevented the loop. Name which of §2.4–§2.5's mechanisms was missing.

### Challenge

8. Complete Stages 7–8, then combine everything into a single configurable resilience layer and
   run a sweep over its parameters (timeout, retry budget, breaker threshold, concurrency limit)
   against a fixed fault scenario. Produce a table of goodput and p99 for each configuration, and
   identify which parameters interact — that is, where changing one alone makes things worse.
9. Reproduce a metastable failure that is *not* retry-driven — a cache-fill storm is the easiest —
   and show that removing the trigger does not restore service. Then implement request coalescing
   (single-flight) and show that it does.

## 6. Self-check

1. What are completeness and accuracy, and what is ◇S sufficient for?
2. Why is a timeout a decision rather than an observation, and what follows from that for
   non-idempotent operations?
3. What does φ measure, and what does decoupling detection from policy buy you?
4. Why full jitter rather than a fixed backoff, and what does a retry budget bound that a
   per-call retry limit does not?
5. Whom does a circuit breaker primarily protect, and why should it trip on rates rather than
   consecutive failures?
6. What is a bulkhead and which specific failure does it prevent?
7. Why does a concurrency limit adapt better than a rate limit?
8. Explain hedged requests and the two conditions required to use them.
9. Define metastable failure and give its diagnostic property.
10. Why should a request that has exceeded its deadline while queued be dropped rather than served?

## 7. Primary sources

- **Chandra & Toueg, "Unreliable Failure Detectors for Reliable Distributed Systems" (JACM 1996)**
  — the failure detector abstraction and the ◇S result.
- **Bronson, Aghayev, Charapko & Zhu, "Metastable Failures in Distributed Systems" (HotOS 2021)**
  — short, and the vocabulary you will use in incident reviews for the rest of your career.
- **Dean & Barroso, "The Tail at Scale" (CACM 2013)** — hedged and tied requests, and the
  arithmetic of fan-out.
- Hayashibara, Défago, Yared & Katayama, "The φ Accrual Failure Detector" (SRDS 2004).
- Brooker, "Exponential Backoff and Jitter" and "Timeouts, Retries and Backoff with Jitter"
  (AWS Builders' Library) — the simulations behind the recommendations here.
- Nygard, *Release It!*, 2nd ed. — circuit breaker, bulkhead, and the stability antipatterns.
- Netflix, "Performance Under Load" — adaptive concurrency limits.
- Google, *Site Reliability Engineering*, chapters 21–22 ("Handling Overload", "Addressing
  Cascading Failures") — the best single treatment of load shedding and cascading failure.

---

**Previous:** [L08](L08-transactions-and-idempotence.md) · **Next:**
[L10 — Observability, Tracing, and Chaos Engineering](L10-observability-and-chaos.md)
