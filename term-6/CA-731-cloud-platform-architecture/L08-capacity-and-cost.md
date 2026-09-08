# CA-731 · Lesson 08 — Capacity, Cost, and the Economics of Architecture

**Estimated study time:** 4.5 hours
**Prerequisites:** L01, L02, L07; PY-601 L09 (backpressure and queueing), PY-602 L09 (capacity)

---

## 1. Orientation

Cost is the part of architecture that engineers are least trained in and most responsible for. The
finance team can renegotiate a contract; only engineers can change the design that generates the
bill, and by the time a bill is large enough to attract attention, the design that produced it is
several years old and load-bearing.

Two framings organise the lesson.

> **Cost is a non-functional requirement like latency or availability, and it should be designed
> for, measured, and reviewed the same way.** A design review that discusses failure modes and
> ignores cost is half a review.

> **The number that matters is not the bill — it is the unit cost.** Total cost growing with a
> growing business is fine. **Cost per tenant, per request, or per active user growing is an
> architecture problem**, and it is the one number that tells you whether your design scales
> economically rather than merely technically.

The lesson also covers capacity, because capacity and cost are the same question asked in different
units: capacity planning is deciding what to buy, and cost is what you paid for the decision.

## 2. Theory

### 2.1 What actually drives a cloud bill

Ranked roughly by how often they are the surprise:

1. **Data transfer.** Internet egress, cross-region, and cross-AZ (L04 §2.3). Frequently
   substantial and almost always under-attributed, because no single team owns it.
2. **Idle capacity.** Provisioned resources at low utilisation. Non-production environments running
   overnight and at weekends are the classic example — a dev environment running 168 hours a week
   for 40 hours of use is 76% waste, and it is invisible because it is spread across many small
   line items.
3. **Storage growth.** Data accumulates by default. Logs, metrics, backups, snapshots, old object
   versions, and orphaned volumes from deleted instances. Nobody deletes anything unless something
   deletes it automatically.
4. **Over-provisioning.** Instances sized for a peak that never happens, or sized by copying
   whatever the last team used.
5. **Managed service premiums.** Convenience has a price, sometimes a large one. Often worth it —
   the comparison must include the engineer-hours you are not spending — but it should be a
   decision, not a default.
6. **Architectural chattiness.** Per-request pricing plus a chatty design equals a bill
   proportional to your internal call graph rather than to your user traffic.

The observation that should change how you look at an architecture: **several of these are
consequences of design decisions that were made for other reasons entirely** — three AZs for
availability, microservices for team autonomy, a managed service for velocity. Each is defensible;
none is free; and almost none is costed at the time.

### 2.2 Unit economics

Define your unit — per tenant, per request, per active user, per transaction, per GB processed —
and compute cost per unit. Then the questions that matter become answerable:

- **Is unit cost flat, rising, or falling with scale?** Falling is a moat: fixed costs amortise and
  statistical multiplexing improves (L02 §2.5). Rising means something in your design scales
  superlinearly — an N² communication pattern, a data structure whose cost grows with total data
  rather than per-request data, or coordination overhead. Find it early; it is much cheaper to fix
  before it dominates.
- **Which tenants or features are unprofitable?** The answer is often uncomfortable and always
  useful, and only the platform team can produce it.
- **What is the marginal cost of one more unit?** This is the number pricing decisions need, and it
  is not the average.

Cost allocation is the enabling machinery, and it requires: a **tagging policy enforced at creation**
(L05's policy-as-code, so untagged resources cannot be created), a method for **apportioning shared
costs** (choose one — by request count, by data volume, by compute-seconds — and be explicit that it
is an approximation), and **showback before chargeback** (show teams their costs before billing them
internally, because chargeback with bad data produces arguments rather than savings).

### 2.3 Capacity: the queueing view

Capacity planning is a queueing problem, and PY-601 L09's material applies directly.

**Little's Law**: `L = λW`. Concurrent requests = arrival rate × time in system. Which gives you
the connection everyone needs and few derive: if you serve 1,000 requests/second with a 200 ms mean
latency, you have 200 requests in flight on average, and that is what determines your thread pool,
connection pool, and memory footprint — not the request rate alone.

**Utilisation and the knee.** Queueing delay grows as `1/(1−ρ)` where ρ is utilisation. At 50%
utilisation the queueing delay equals the service time; at 90% it is nine times; at 99%, ninety-nine
times. This is why **you cannot run at high utilisation and low latency simultaneously**, and why
"our servers are only at 60% CPU, we have plenty of headroom" is wrong for a latency-sensitive
service. The headroom is the latency budget.

**The Universal Scalability Law** (PY-602 L09) extends Amdahl: throughput is limited by a serial
fraction *and* by a coherency (coordination) term that makes throughput *decrease* beyond some
concurrency. Real systems have a peak, after which adding capacity makes things worse. Knowing
roughly where yours is means you can stop adding instances to a problem that adding instances
worsens.

The practical planning procedure:

1. **Measure the demand distribution** — peak, average, peak-to-average ratio, growth rate, and the
   shape of the peaks (a smooth daily cycle and a spiky event-driven pattern need different
   answers).
2. **Load test to find the knee**, not the maximum. The knee — where latency starts climbing — is
   your usable capacity; the maximum is where it falls over.
3. **Choose a headroom target** derived from your SLO (L07), your scaling speed, and how correlated
   your failure domains are. N+1 across three AZs means each AZ runs at ~67% of its own capacity so
   that losing one leaves the others able to absorb the load — which is the arithmetic behind
   static stability (L01 §2.3) and is *why* it costs more.
4. **Decide on autoscaling versus static provisioning** — §2.4.
5. **Re-measure quarterly**, because the workload changes.

### 2.4 Autoscaling, and its limits

Autoscaling is the default answer and it is frequently the wrong one, for reasons worth being
precise about:

- **It is reactive, and reaction takes time.** Detect (metrics delay), decide (evaluation period),
  provision (instance start, image pull, application warm-up, JIT warm-up, cache fill). A minute is
  optimistic; several is common. **A spike faster than your scaling time is not handled by
  autoscaling**, and many real spikes are.
- **It depends on the control plane** (L01 §2.3), which is least reliable exactly when everyone is
  scaling at once.
- **It can oscillate.** Scale up, load drops, scale down, load returns. Hysteresis (different
  thresholds for up and down) and cooldowns exist for this, and getting them wrong is common.
- **It can amplify a failure.** If latency rises because a dependency is slow, scaling up adds
  concurrency to an already-overloaded dependency — DS-701 L09's metastable failure, accelerated by
  your own automation.
- **Scaling on the wrong metric.** CPU is the default and is often uncorrelated with the actual
  constraint. Scale on the queue depth or the concurrency (PY-601 L09), which is what Little's Law
  says is the real signal.

The alternatives, each with a fit:

- **Static provisioning at peak.** Simple, statically stable, expensive at low duty cycle. Correct
  for latency-critical services with a predictable peak.
- **Scheduled scaling** for predictable cycles — most business workloads have one, and a schedule
  is more reliable than reacting to it.
- **Predictive scaling** from historical patterns, scaling *before* the demand rather than after.
- **Buffering with a queue** so demand spikes become latency rather than failure — the most robust
  answer where the work can be asynchronous, and the one that requires the least infrastructure.
- **Serverless** where the platform absorbs the scaling problem, at a higher unit cost and with its
  own cold-start behaviour.

The honest summary: **autoscale for cost, provision for availability.** Use autoscaling to shed cost
during troughs, and keep enough static capacity that your availability does not depend on scaling
succeeding during an incident.

### 2.5 The purchasing decisions

Provider-specific in detail, universal in structure:

- **On-demand**: maximum flexibility, maximum unit price. Correct for unpredictable or short-lived
  workloads.
- **Commitment-based discounts** (reserved instances, savings plans, committed use): 30–70% cheaper
  for a 1–3 year commitment. The break-even is a utilisation percentage you can compute. The risk
  is committing to a shape you then change — so commit to the *floor* of your usage, not the
  average, and cover the rest on demand.
- **Spot / preemptible**: 60–90% cheaper, can be reclaimed with short notice. Excellent for batch,
  CI, and anything checkpointable (DI-721 L07's batch reliability model makes batch a natural fit).
  Requires the workload to handle interruption, and requires diversification across instance types
  so a single pool's exhaustion does not take everything.
- **Graviton / ARM and newer generations**: often 20–40% better price-performance for a rebuild and
  a test cycle. Frequently the largest easy saving available, and frequently deferred indefinitely
  because nobody owns it.

The structural insight: **the discount is payment for predictability.** You are being paid to make
the provider's capacity planning easier. Which means the more predictable your workload, the more of
the discount you can capture — and making a workload more predictable (scheduling batch off-peak,
smoothing with queues) is itself a cost optimisation.

### 2.6 Optimising, in the right order

The order matters, because the effort is very unevenly distributed relative to the savings:

1. **Delete what is unused.** Orphaned volumes, unattached IPs, old snapshots, dev environments
   nobody uses, forgotten load balancers. Free savings, no risk, and every organisation has some.
2. **Turn off what is not needed.** Non-production on a schedule. Usually a large percentage of the
   non-production bill for an afternoon's work.
3. **Right-size.** Match instance sizes to measured usage rather than to habit. Beware of sizing to
   average when the constraint is peak.
4. **Fix the transfer paths.** Gateway endpoints instead of NAT, zone affinity where availability
   permits, compression, caching at the edge (L04 §2.3).
5. **Manage the storage lifecycle.** Tier and expire logs, metrics, backups and old object versions.
   Check retrieval costs before moving anything to a cold tier you will actually read.
6. **Commit** to the stable floor.
7. **Architect.** Change the design: batch instead of per-request, cache, denormalise, move to a
   cheaper service model. Highest potential savings, highest cost and risk, and only worth it once
   1–6 are done — because the architectural work is expensive and the earlier steps are nearly free.

And the meta-rule: **the biggest line item is not always the biggest opportunity.** Compute is
usually the largest number and often the hardest to reduce; transfer and idle capacity are usually
smaller and much easier. Sort by *achievable* saving, not by size.

### 2.7 Cost as a design input

Concretely, at design time:

- **Estimate the cost of each option** before choosing, at expected scale and at 10× expected
  scale. The 10× estimate catches the design that is cheap now and catastrophic later.
- **Identify the dominant term** and design against it. If transfer dominates, colocate; if requests
  dominate, batch; if storage dominates, tier and expire.
- **Know the cost of your reliability decisions.** Three AZs, cross-region replication and static
  stability all have prices, and they should be presented alongside the availability they buy (L07
  §2.4) so the decision is made once, explicitly, by someone empowered to make it.
- **Watch for denial-of-wallet.** An unauthenticated endpoint on a per-request-priced service is a
  financial vulnerability. Rate limit, cap, and alert on anomalous spend — this is a security
  control, not a finance one.

## 3. Construction: model, measure, and reduce

Build in `mpse/ca731/l08/`, applied to your L02/L04 platform.

**Stage 1 — the cost model.** Build a parameterised model: monthly cost as a function of request
volume, tenant count, data stored, and transfer split by path (same-AZ, cross-AZ, cross-region,
internet). Validate it against a real bill (or a realistic synthetic one) and report the error. A
model you have not validated is a spreadsheet.

**Stage 2 — sensitivity analysis.** Vary each input by ±50% and by 10× and identify which terms
dominate at each scale. Produce the tornado chart. The finding that matters: the dominant term at
10× is frequently not the dominant term today, and it is the one your architecture should be able
to accommodate.

**Stage 3 — unit economics.** Instrument per-tenant resource consumption (from L02 Stage 9) and
compute cost per tenant across a simulated tenant distribution with a realistic long tail. Plot unit
cost against tenant count and determine whether it is flat, rising or falling. If it is rising, find
the superlinear term and explain it.

**Stage 4 — capacity, measured.** Load test to find the knee: plot latency and throughput against
offered load, and mark the knee and the maximum. Compute your utilisation at the knee. Then apply
Little's Law to derive the concurrency at your target load and check it against your configured pool
sizes — this comparison usually finds a misconfiguration.

**Stage 5 — the headroom arithmetic.** Compute the capacity required for N+1 across three AZs at
your SLO, and the resulting steady-state utilisation. Then compute the cost of that headroom, and
compare it against the cost of an outage at your SLO's downtime budget (L07). Present both numbers.
This is the static stability decision (L01 Stage 4), now with prices attached.

**Stage 6 — autoscaling, and where it fails.** Implement autoscaling on CPU. Then construct three
failures: a spike faster than the scaling time; an oscillation from badly-chosen thresholds; and
the amplification case where scaling up worsens an overloaded dependency. Then re-implement scaling
on concurrency or queue depth and compare all three scenarios. Report which metric handled which
failure.

**Stage 7 — the buffered alternative.** Take the spiky workload from Stage 6 and put a queue in
front of it, converting the spike into latency rather than failure or scaling. Compare cost,
latency distribution and failure behaviour against the autoscaled version. State the condition under
which each is correct.

**Stage 8 — the optimisation pass.** Apply steps 1–6 of §2.6 to a real environment (yours, or a
realistic synthetic one). For each step record: time spent, monthly saving, and risk introduced.
Produce the table sorted by saving per hour of effort. Then estimate what step 7 — architectural
change — would save, and whether it is worth it.

**Stage 9 — denial of wallet.** Build an endpoint on a per-request-priced service, then simulate an
abusive client and compute the bill. Implement the controls — per-caller rate limits, a hard spend
cap, and an anomaly alert on spend rate — and demonstrate each working. Write the runbook for "the
bill is spiking right now", which should be short enough to act on in five minutes.

## 4. Failure modes

- **Cost discovered at the invoice.** By then the design is years old.
- **Optimising the largest line item** rather than the largest achievable saving.
- **Total cost tracked, unit cost not.** You cannot tell growth from waste.
- **No tag enforcement.** Allocation is impossible and every cost conversation is a guess.
- **Chargeback before showback.** Produces arguments about the data instead of savings.
- **Non-production running 24/7.** Large, easy, and routinely ignored.
- **Storage with no lifecycle policy.** Grows forever by default.
- **Autoscaling as the availability strategy.** It is reactive and control-plane dependent.
- **Scaling on CPU when the constraint is concurrency.** Scales the wrong thing at the wrong time.
- **Committing to average rather than floor usage.** Paying for capacity you stopped using.
- **Running at high utilisation for cost while promising low latency.** Queueing theory says pick
  one.
- **Unauthenticated per-request-priced endpoints.** A financial vulnerability.
- **Presenting an availability decision without its price.** The decision gets made by default
  rather than deliberately.

## 5. Exercises

### Warm-up (30 min)

1. Give six drivers of a cloud bill and say which of them are consequences of decisions made for
   other reasons.
2. Apply Little's Law: 2,000 requests/second, 150 ms mean latency — how many in flight, and what
   does that determine?
3. Compute queueing delay as a multiple of service time at 50%, 80%, 90% and 95% utilisation, and
   state what that implies for a latency SLO.

### Core (3.5 h)

4. Complete Stages 1–3, including the model's validation error and the unit-cost curve with the
   superlinear term identified (or its absence explained).
5. Complete Stages 4–5 and deliver the knee measurement, the Little's Law check against your pool
   sizes, and the headroom-cost versus outage-cost comparison.
6. Complete Stage 6 with all three autoscaling failures and the metric comparison.
7. Complete Stage 8 and deliver the saving-per-hour table.

### Challenge

8. Complete Stages 7 and 9, including the five-minute runbook.
9. Build a **cost-aware architecture review tool**: given infrastructure code (L05) and a traffic
   model, estimate monthly cost before deployment, break it down by component and by transfer path,
   flag the terms that grow superlinearly with the traffic model's parameters, and compare two
   candidate architectures side by side. Then validate it against a real deployment and report the
   error. Finally — the part that makes it useful rather than clever — integrate it into the L05
   pipeline so that a pull request whose estimated monthly cost increases by more than a threshold
   requires explicit acknowledgement. Report how the team responded to it after a month, because a
   tool that is uniformly ignored has failed regardless of its accuracy.

## 6. Self-check

1. Give six cost drivers ranked by how often they surprise people.
2. Why is unit cost more informative than total cost, and what does a rising unit cost indicate?
3. State Little's Law and derive the in-flight concurrency for a given rate and latency.
4. Why can you not have high utilisation and low latency at the same time?
5. What does the Universal Scalability Law add to Amdahl's law, and what does it imply about adding
   capacity?
6. Give five reasons autoscaling fails, and the summary rule about autoscaling versus provisioning.
7. Why is scaling on concurrency better than scaling on CPU?
8. What are you being paid for by a commitment discount, and what does that imply for making
   workloads more predictable?
9. Give the seven optimisation steps in order and say why the order matters.
10. What is denial of wallet, and why is it a security control rather than a finance one?

## 7. Primary sources

- **Google, *Site Reliability Engineering*, chapter 11 and the capacity planning material**; and
  *The SRE Workbook* on managing load.
- **Gunther, *Guerrilla Capacity Planning*** — the USL and the discipline of capacity modelling.
- Little's Law and basic queueing theory: any operations research text, or Gunther's treatment.
- AWS Builders' Library: "Using Load Shedding to Avoid Overload" and Brooker's writing on capacity
  and static stability.
- The FinOps Foundation framework — read it as an organisational process description; the technical
  content is thin but the allocation and showback material is sound.
- Cloud provider pricing documentation, read alongside the architecture documentation per L01 §2.5.
- Hamilton, "On Designing and Deploying Internet-Scale Services" (LISA 2007) — old, and most of it
  is still right, including the cost arguments.

---

**Previous:** [L07](L07-reliability-engineering.md) · **Next:**
[L09 — Platform Engineering as a Product Discipline](L09-platform-as-product.md)
