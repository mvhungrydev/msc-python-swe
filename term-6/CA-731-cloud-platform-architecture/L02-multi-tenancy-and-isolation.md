# CA-731 · Lesson 02 — Multi-Tenancy and Isolation

**Estimated study time:** 4.5 hours
**Prerequisites:** L01; DS-701 L06, DS-701 L09; SE-521 L07

---

## 1. Orientation

Multi-tenancy is what makes cloud economics work: one instance of a system serving many customers,
amortising fixed costs across all of them. It is also where the two hardest questions in platform
engineering live:

1. **What stops one tenant from seeing another's data?** (isolation as a security property)
2. **What stops one tenant from degrading another's experience?** (isolation as a performance
   property)

These are different problems with different mechanisms, and conflating them is the source of most
bad multi-tenancy designs. A system can have perfect data isolation and terrible performance
isolation — that is, in fact, the default outcome, because data isolation is easy to test and
performance isolation is only visible under load.

The framing that organises the lesson:

> **"Isolated" is not a property, it is a spectrum with a stated boundary.** The engineering
> question is never "is it isolated?" but "isolated against what, to what degree, and what happens
> at the boundary?"

## 2. Theory

### 2.1 The isolation spectrum

From most shared to most isolated:

| Model | Shared | Isolation | Cost/tenant | Fits |
|---|---|---|---|---|
| **Shared everything** | one database, tenant ID column | application logic only | lowest | many small tenants, uniform |
| **Shared DB, separate schema** | one instance, schema per tenant | database access control | low | moderate tenant count |
| **Separate database** | one cluster, DB per tenant | database-level | medium | tens to hundreds |
| **Separate instance / cluster** | infrastructure account | process and network | high | large or regulated tenants |
| **Separate account / project** | nothing but code | provider-level | highest | compliance boundaries |

The right answer is usually **not one row**. Mature platforms are tiered: pooled infrastructure for
the long tail, dedicated resources for large or regulated customers, with a migration path between
them. Which means the migration path — moving a tenant from pooled to dedicated without downtime —
is a first-class feature, and designing it in from the start is much cheaper than retrofitting it.

The dominant cost driver: **shared everything has the lowest marginal cost per tenant and the
highest blast radius; separate accounts invert both.** Pick per tenant tier, not per platform.

### 2.2 Data isolation, and the one-line catastrophe

In a shared-everything model, isolation rests on every query filtering by tenant ID. Which means
**one missing `WHERE tenant_id = ?` is a data breach.** Not a bug — a breach, reportable, with
regulatory consequences.

Defence in depth, because relying on developer discipline is not a control:

1. **Never pass tenant ID as a normal parameter.** It comes from the authenticated context, is set
   once at request entry, and is not something a handler can choose. If a function *can* take a
   tenant ID as an argument, someone will eventually pass the wrong one.
2. **Enforce it below the application.** PostgreSQL **row-level security** with a session variable
   means the database filters even if the query forgot. The predicate lives in one place, is
   audited once, and cannot be bypassed by a new query. This is the single highest-value control
   in this lesson.
3. **Make the ORM or data layer enforce it** so it is impossible to write an unfiltered query
   without deliberately reaching around the layer — and make that reach-around visible in review.
4. **Test it adversarially**: a test suite that, for every endpoint, authenticates as tenant A and
   attempts to access tenant B's resources by ID, asserting 404 (not 403 — a 403 confirms the
   resource exists, which is itself a small leak).
5. **Audit at the boundary**: log the tenant on every access, and run a periodic check for queries
   returning rows from more than one tenant.

For blob storage, the equivalent is prefix-per-tenant plus a policy that scopes credentials to that
prefix — so the *credential* cannot address another tenant's data, rather than the code choosing
not to. The general principle: **prefer controls that make the wrong thing impossible over controls
that make it detectable.**

### 2.3 Performance isolation: the noisy neighbour

Data isolation is a correctness problem with a clear test. Performance isolation is a queueing
problem, and it is where multi-tenant systems actually fail.

The shapes it takes:

- **One tenant's traffic spike** consumes shared capacity — connections, threads, CPU, IOPS.
- **One tenant's expensive query** occupies a worker for minutes.
- **One tenant's data volume** blows the cache for everyone (L01/DI-721's cache pollution).
- **One tenant's key** is hot and saturates a partition (DS-701 L06).
- **One tenant's retry storm** amplifies their own outage into everyone's (DS-701 L09).

The mechanisms, roughly in order of how much they help:

1. **Per-tenant rate limits and quotas.** The baseline. Set them, publish them, and return a
   documented 429 with `Retry-After` so clients can behave. Note that a limit nobody is told about
   is a trap, not a control.
2. **Per-tenant concurrency limits.** Better than rate limits (DS-701 L09 §2.5): a concurrency cap
   self-adjusts to how expensive the work actually is, where a request/second limit does not.
3. **Fair queueing.** Round-robin or weighted-fair across tenant queues rather than one FIFO, so a
   flood from one tenant cannot starve others. Kubernetes' API Priority and Fairness is a
   production example worth studying.
4. **Bulkheads.** Separate resource pools per tenant tier, so the pooled tier cannot exhaust the
   dedicated tier's capacity.
5. **Shuffle sharding.** The best idea in this lesson. Assign each tenant a *random subset* of
   workers rather than one shard. With 8 workers and 2 per tenant there are 28 combinations, so two
   random tenants share both workers with probability 1/28 — one bad tenant fully degrades only
   ~3.5% of others, and with retries to the non-overlapping worker, most see no impact at all.
   The arithmetic is combinatorial and the effect is dramatic: it converts "one tenant takes down
   the service" into "one tenant slightly inconveniences a small, computable fraction". Read
   Brooker's article and do the arithmetic yourself; it is the single most useful thing here.
6. **Admission control and load shedding**, prioritised by tier (DS-701 L09).
7. **Cost-based limits**: bound the *work* a request may do (a query timeout, a row-scan limit, a
   result-size cap), not merely the request count. One request is not one unit of load, and
   pretending otherwise is why rate limits alone are insufficient.

### 2.4 Tenant-aware everything

Multi-tenancy leaks into every part of the system, and the parts people forget are the ones that
cause incidents:

- **Observability**: every metric, log and trace carries the tenant. Without it you cannot answer
  "is this slow for everyone or for one customer?", which is the first question in every
  multi-tenant incident. This is DI-721/DS-701 L10's cardinality argument with a concrete payoff —
  and note that tenant ID is exactly the high-cardinality dimension a metrics system cannot hold,
  which is why wide events matter here.
- **Deployment**: tenant-aware rollouts let you deploy to internal tenants, then small ones, then
  large ones. Your deployment pipeline is otherwise a correlation mechanism (L01 §2.4).
- **Backup and restore**: restoring *one tenant* from a shared database is genuinely hard, and it
  is a requirement you will be given eventually. Design for it or be honest that you cannot do it.
- **Data deletion**: "delete everything about this tenant" across a shared database, caches,
  search indexes, logs, backups and analytics is a project, not a query (DI-721 L09's
  crypto-shredding is relevant here too).
- **Migration between tiers**: moving a tenant from pooled to dedicated without downtime.
- **Per-tenant configuration**: feature flags, limits, and settings — which quickly becomes a
  system of its own, and one that must not become per-tenant *code*.

The rule: **if a component is not tenant-aware, it is a place where tenants are mixed.** Enumerate
them.

### 2.5 The cost dimension

Multi-tenancy exists for cost, so measure whether it is delivering:

- **Cost per tenant** = allocable direct costs + shared costs apportioned by some measure. Choose
  the measure deliberately (requests, storage, compute-seconds) and be aware it is an
  approximation.
- **Utilisation** is the point of pooling: tenants' peaks do not coincide, so pooled capacity is
  smaller than the sum of dedicated capacities. Measure the actual peak-to-average ratio across
  your tenant base — the statistical multiplexing gain is the entire economic argument for shared
  infrastructure, and if your tenants' peaks *do* coincide (a business-hours SaaS in one timezone),
  the gain is much smaller than assumed.
- **The long tail** is where pooling pays: thousands of tenants using almost nothing each cannot be
  served profitably on dedicated infrastructure.
- **The large tenant** may be cheaper to serve dedicated, and is usually the one who wants it
  anyway.

The uncomfortable finding this exercise often produces is that a small number of tenants consume a
large majority of resources while paying a small fraction of the revenue. That is a pricing
problem, and the platform team is the only group that can surface it.

### 2.6 The security boundary, stated honestly

What each level actually protects against:

- **Application-level filtering**: protects against nothing if the application has a bug. It is a
  correctness measure, not a security boundary.
- **Database row-level security**: protects against application bugs. Does not protect against a
  compromised application credential that can set the session variable.
- **Separate databases**: protects against application bugs and credential scope errors. Shares the
  database engine, so an engine-level vulnerability crosses it.
- **Separate compute (containers on shared kernel)**: protects against most application compromise.
  A container escape crosses it — shared-kernel isolation is real but not equal to a VM, which is
  why lightweight VMs (Firecracker, gVisor) exist for genuinely untrusted workloads.
- **Separate VMs**: protects against container escape. Shares hardware, so side-channel attacks
  (Spectre-class) are the remaining theoretical concern.
- **Separate accounts**: the provider's own strongest boundary, and what regulators generally
  recognise.

State your boundary explicitly, in writing, and state what it does not protect against. **A
customer asking "is our data isolated?" deserves the honest specific answer, and a platform team
that cannot give one does not know its own design.** Being able to write that paragraph is a
professional skill, and it is Stage 7 of §3.

## 3. Construction: build a multi-tenant service and break it

Build in `mpse/ca731/l02/`. The service can be simple — a per-tenant document store with an API —
because the interesting content is the isolation, not the domain.

**Stage 1 — shared everything.** One database, a `tenant_id` column, filtering in application code.
Authentication that establishes tenant from a token. Get it working.

**Stage 2 — the breach.** Write the adversarial test suite: for every endpoint, authenticate as
tenant A and attempt tenant B's resources by ID. Then deliberately introduce a missing filter in
one handler and confirm the suite catches it. Then check the negative case — does it return 404 or
403? Fix it if it leaks existence.

**Stage 3 — defence in depth.** Add PostgreSQL row-level security with the tenant set from a
session variable at connection checkout. Re-introduce the missing filter from Stage 2 and show that
the data does *not* leak. Then write down what this control does not protect against, precisely.

**Stage 4 — the noisy neighbour, demonstrated.** Add a workload generator with per-tenant traffic
profiles. Have one tenant issue an expensive query pattern (or simply 100× the traffic) and measure
the p99 latency for the other tenants. Plot it. This chart is the problem statement.

**Stage 5 — mitigations, measured.** Implement, and measure the effect of each in turn, keeping
Stage 4's chart as the baseline: (a) per-tenant rate limits, (b) per-tenant concurrency limits,
(c) a query cost limit (timeout plus row cap), (d) fair queueing across tenant queues. Report which
gave the most improvement per unit of complexity. The answer is usually not the one people
implement first.

**Stage 6 — shuffle sharding.** Implement it: N workers, k per tenant, deterministic assignment
from a hash of the tenant ID. Compute the overlap probability for your N and k, then verify it
empirically. Now simulate one tenant saturating its workers and measure the fraction of other
tenants affected, against the prediction. Then add client-side retry to a non-overlapping worker
and measure again. Produce the comparison against simple sharding.

**Stage 7 — the isolation statement.** Write the document a customer's security team would receive:
what is shared, what is isolated, at which layer each control sits, what each protects against and
what it does not, and what your blast radius is for each failure class. One page, honest. Then have
someone technical read it and ask the questions it fails to answer.

**Stage 8 — the hard operations.** Implement: (a) export all of one tenant's data, (b) delete all
of one tenant's data across the database, cache and search index with verification, and (c) migrate
one tenant from the shared database to a dedicated one with no downtime and no lost writes — which
is DI-721 L10's expand/migrate/contract applied to a tenant. Time each on a realistic data volume.

**Stage 9 — cost per tenant.** Instrument requests, storage and compute-seconds per tenant. Produce
the distribution across a simulated tenant base with a realistic long tail, and compute cost per
tenant and the statistical multiplexing gain versus dedicated infrastructure. Then find the
crossover point at which a tenant becomes cheaper to serve dedicated.

## 4. Failure modes

- **Isolation by application-code discipline alone.** One missed filter is a breach.
- **Tenant ID as a normal function parameter.** Eventually the wrong one is passed.
- **403 where 404 is correct.** Confirms existence of another tenant's resource.
- **Rate limits without cost limits.** One request can be a thousand times more expensive than
  another.
- **Rate limits with no documentation and no `Retry-After`.** Clients cannot behave well; you have
  built a trap rather than a control.
- **Untenanted observability.** You cannot answer the first question of any multi-tenant incident.
- **No per-tenant deployment control.** The pipeline correlates all your tenants' fates.
- **Single-tenant restore not designed for.** It will be requested, and retrofitting it is
  expensive.
- **No tier migration path.** Every large customer becomes an escalation.
- **Assuming pooling gains without measuring peak coincidence.** If all tenants peak at 09:00, the
  pool is nearly the sum of the peaks.
- **An isolation claim the team cannot substantiate.** Worse than a weaker claim honestly stated.

## 5. Exercises

### Warm-up (30 min)

1. Give the five points on the isolation spectrum with cost and blast radius for each, and say when
   a platform should be tiered rather than uniform.
2. Distinguish data isolation from performance isolation, giving the mechanism and the test for
   each.
3. With 10 workers and 3 per tenant, compute the number of possible shuffle-shard combinations and
   the probability that two random tenants share all three workers.

### Core (3.5 h)

4. Complete Stages 1–3. Deliver the adversarial test suite and the RLS demonstration, plus the
   precise statement of what RLS does not protect against.
5. Complete Stages 4–5 and deliver the noisy-neighbour chart with all four mitigations overlaid,
   and your judgement of improvement per unit of complexity.
6. Complete Stage 6, including the analytical prediction and its empirical verification.
7. Complete Stage 7 and have a colleague interrogate the document. Record the questions it could
   not answer; those are your design gaps.

### Challenge

8. Complete Stage 8, with timings, and Stage 9 with the cost distribution and crossover point.
9. Design and implement a **tenant tiering system**: pooled, dedicated-database, and
   dedicated-infrastructure tiers, with automated placement based on measured resource consumption
   and an automated migration between tiers that preserves availability. Then run a simulation over
   a year of synthetic tenant growth and report: total cost under your tiering versus
   all-pooled versus all-dedicated, the number of migrations triggered, and any tenant that
   oscillated between tiers (which is a hysteresis bug, and finding it is part of the exercise).

## 6. Self-check

1. Give the isolation spectrum and the cost/blast-radius trade at each point.
2. Why is application-level tenant filtering not a security boundary?
3. Give five defence-in-depth controls for data isolation, and say which makes the wrong thing
   impossible rather than merely detectable.
4. Give five shapes the noisy-neighbour problem takes.
5. Why is a concurrency limit better than a rate limit for performance isolation?
6. Explain shuffle sharding and do the arithmetic for N=8, k=2.
7. Why must observability be tenant-aware, and why can metrics alone not carry it?
8. Name four operations that are hard in a shared-everything model.
9. What does each isolation layer protect against, and what crosses it?
10. What is the statistical multiplexing gain, and what workload property destroys it?

## 7. Primary sources

- **Brooker, "Workload Isolation Using Shuffle Sharding" (AWS Builders' Library)** — do the
  arithmetic yourself as you read it.
- **Brooker, "Fairness in Multi-Tenant Systems"** and the AWS Builders' Library articles on
  throttling and load shedding.
- Google, *Site Reliability Engineering*, chapter 21 ("Handling Overload") and chapter 22.
- The Kubernetes API Priority and Fairness design document — a production fair-queueing
  implementation, well documented.
- The PostgreSQL documentation on row-level security.
- AWS SaaS Lens (Well-Architected) and the SaaS tenant isolation whitepapers — vendor material,
  but the isolation taxonomy is sound; read it critically.
- Agache et al., "Firecracker: Lightweight Virtualization for Serverless Applications" (NSDI 2020)
  — where the isolation boundary is drawn when tenants are genuinely untrusted, and why.

---

**Previous:** [L01](L01-cloud-as-distributed-system.md) · **Next:**
[L03 — Identity, Trust Boundaries, and Least Privilege](L03-identity-and-trust.md)
