# CA-731 · Lesson 01 — The Cloud as a Distributed System with a Bill

**Estimated study time:** 4 hours
**Prerequisites:** DS-701 (all), SE-521 L09

---

## 1. Orientation

The most useful mental shift available to a cloud engineer is this one:

> A managed service is not a magic box. It is **a distributed system somebody else operates**,
> which means it has a failure model, a consistency model, a partitioning scheme, a capacity
> limit, and a set of trade-offs its designers chose — all of which you inherit, and most of which
> are documented if you know what to look for.

DS-701 gave you the vocabulary. This lesson gives you the reading skill: given a service's
documentation, work out what it actually is, what it guarantees, how it fails, and what its
pricing model is telling you about its architecture.

That last point deserves emphasis, because it is the one nobody teaches:

> **The pricing model is a leaked architecture diagram.** Providers charge for what is expensive
> to them. If a service charges per request but not per GB stored, storage is cheap in its design
> and request handling is not. If cross-AZ traffic costs money and same-AZ does not, the physical
> topology is telling you where the expensive links are. If a service has a "provisioned" and an
> "on-demand" mode with very different prices, that is a statement about how hard capacity
> planning is inside it.

Reading prices this way turns the bill from an accounting artifact into a design document.

## 2. Theory

### 2.1 What "managed" moves and what it does not

A managed service moves *operational* work — patching, replacing failed hardware, replication
mechanics, backup execution, capacity provisioning. It does not move:

- **The consistency model.** If the service is eventually consistent, your application must handle
  eventual consistency. No amount of management changes that.
- **The failure model.** Managed services fail. Regions fail. AZs fail. Control planes fail
  *separately from* data planes (§2.3). Your architecture handles it or it does not.
- **The limits.** Every service has quotas, throttles and partition-level throughput ceilings, and
  exceeding them produces errors your code must handle. The limits are the contract, and most of
  them are documented; almost nobody reads them until an incident.
- **Data modelling.** A managed database with a bad key design is a bad database.
- **The bill.** Which is a function of your architecture, not theirs.

The honest framing: managed services move the *undifferentiated* work and leave you the
*differentiated* work, which is exactly the work that requires understanding. That is a good trade
and it is not the same as the work disappearing.

### 2.2 Reading a service as a distributed system

A repeatable procedure. Apply it to any managed service and you will know more than most of its
users:

1. **What is the unit of partitioning?** DynamoDB partitions by hash of the partition key; Kafka
   by topic partition; S3 by key prefix (historically; now automatic); Kinesis by shard. Once you
   know this, you know where hot partitions can occur (DS-701 L06) and what the per-partition
   ceiling is.
2. **What is the replication and durability model?** How many copies, in how many failure domains,
   and what is acknowledged before a write returns? S3's eleven nines is a statement about
   *durability* — the probability of losing an object — not about availability, and the two are
   routinely confused.
3. **What is the consistency model?** Read-after-write? Eventually consistent reads at half price
   (which is DynamoDB telling you exactly what a quorum read costs)? Linearizable with a
   conditional write? Every managed data service answers DS-701 L04's question, usually explicitly.
4. **What is the failure domain?** AZ, region, or global. What is the blast radius of one
   component's failure, and what is shared across tenants (L02)?
5. **What are the limits?** Requests per second, per partition, per account; payload sizes;
   connection counts; and — most important — **what happens when you exceed them**. A throttle
   (429, `ProvisionedThroughputExceededException`) is a backpressure signal (PY-601 L09) and your
   client must treat it as one, with backoff and jitter (DS-701 L09).
6. **What is the control plane's availability versus the data plane's?** §2.3.
7. **What does it cost, and what does the cost structure imply?** §2.5.

Practise this on three services you already use. It reliably surfaces something you did not know.

### 2.3 The control plane / data plane distinction

The single most valuable structural idea in cloud architecture, and the one that most often
explains an incident.

- The **data plane** serves your workload's requests: reading an S3 object, invoking a function,
  routing a packet.
- The **control plane** changes the system's configuration: creating a bucket, launching an
  instance, updating a load balancer's targets, scaling a group.

They have different characteristics, deliberately:

| | Data plane | Control plane |
|---|---|---|
| Request rate | very high | low |
| Availability target | highest | lower |
| Complexity | simple, constant-work | complex, stateful workflows |
| Blast radius of failure | one request | everything that needs to change |
| Typical failure mode | throttle a request | cannot make changes |

**Control planes fail more often than data planes**, and that is by design: they are more complex,
change more, and are held to a lower availability bar. The architectural consequence, which is the
lesson's most actionable rule:

> **Your recovery path must not depend on the control plane.** If your response to an AZ failure is
> "launch instances in another AZ", you are depending on the control plane at exactly the moment it
> is most likely to be degraded — and, in a large event, most likely to be overwhelmed by everyone
> else doing the same thing.

The design response is **static stability**: pre-provision capacity so that recovery requires no
control-plane action. Run at N+1 across AZs so losing one requires *nothing to happen*. It costs
more in steady state and it is the difference between a system that survives a large event and one
that queues behind everyone else's recovery. Brooker's writing on this is the best available.

The related idea is **constant work**: design components so their workload does not change with
conditions. A configuration distributor that pushes the *full* configuration on a fixed schedule
does the same work whether nothing changed or everything did — so the failure case exercises the
same code path as the normal case, at the same rate. Systems whose recovery path is a rarely-used
code path fail in recovery.

### 2.4 Failure domains and the shape of correlated failure

- **Availability zone**: independent power, cooling and network within a region; single-digit
  millisecond latency between AZs, and physically far enough apart to fail independently for most
  causes.
- **Region**: a set of AZs. Regions are meant to be independent — but note that some services are
  regional with a global control plane, and that dependency is where cross-region incidents
  originate.
- **Global services**: DNS, IAM, CDN. These are the shared-fate components, and their failure is
  correlated across everything you own.

Correlation is what matters. Independent failures are easy to reason about with the arithmetic of
availability (§2.6); correlated ones break the arithmetic entirely. The correlated failures that
actually happen:

- **A shared dependency**: everything in your architecture depends on one identity service, one
  DNS zone, one certificate authority, one config store.
- **A deploy**: the same bad change rolled to every AZ simultaneously. Your deployment pipeline is
  a correlation mechanism, which is why staged rollouts and bake times exist.
- **A retry storm**: everyone's recovery logic firing at once (DS-701 L09's metastable failure at
  provider scale).
- **A capacity pool**: "multi-AZ" instances drawing from a shared regional capacity pool means an
  AZ failure can exhaust the remaining pool for everyone at once.

The design discipline: **enumerate what is shared.** Draw the dependency graph, mark every node
that appears in more than one path, and for each ask what happens when it is unavailable. That
exercise is Stage 3 of §3, and it is the most valuable hour in this lesson.

### 2.5 The bill as a design document

Cost is an architectural property, and treating it as a finance problem is how organisations end
up with a bill they cannot explain. The structural facts:

- **Data transfer** is often the surprise. Egress to the internet, cross-region and cross-AZ
  transfer all cost money; same-AZ generally does not. A chatty service mesh spread across three
  AZs for availability is paying cross-AZ transfer on every internal call, and this is a common,
  large, invisible line item. That trade — availability against transfer cost — is a real
  architectural decision, not a finance one.
- **Request-based pricing** (per API call, per invocation) makes chattiness expensive and rewards
  batching. It also makes cost proportional to load, which is either a feature or a denial-of-wallet
  vulnerability, depending on whether an unauthenticated endpoint is involved.
- **Provisioned versus on-demand** is a bet on predictability. Provisioned is cheaper per unit and
  you pay for the peak; on-demand costs more per unit and you pay for what you use. The break-even
  is a utilisation percentage you can compute, and most organisations do not.
- **Storage tiers** encode an access-frequency assumption, with retrieval costs and minimum
  durations that make the cheap tier expensive if you read it — a lifecycle policy that moves data
  to archive and a workload that then reads it monthly can cost more than doing nothing.
- **The idle cost** is what you pay when nothing is happening. Serverless is near zero; a
  provisioned cluster is not. For spiky or low-volume workloads this dominates everything else.

The unit that makes cost architectural is **cost per unit of business value** — per tenant, per
request, per active user. An absolute bill that grows is uninformative; a unit cost that grows is a
design problem, and one that shrinks with scale is a moat. L08 develops this properly.

### 2.6 Availability arithmetic, and its limits

Serial dependencies multiply: a service depending on three components at 99.9% each has a ceiling
of 99.7%. Redundant components multiply their *unavailability*: two independent 99% components in
parallel give 99.99%.

Which yields the useful budget table:

| Availability | Downtime per month |
|---|---|
| 99% | ~7.2 hours |
| 99.9% | ~43 minutes |
| 99.95% | ~22 minutes |
| 99.99% | ~4.3 minutes |
| 99.999% | ~26 seconds |

Now the honest part, which most treatments omit. **This arithmetic is nearly always wrong in
practice**, for three reasons: failures are correlated (§2.4), so the parallel calculation
overstates redundancy badly; the arithmetic counts only the components you listed, and real systems
have dependencies nobody drew; and most downtime is caused by *changes*, not by component failure,
so a number derived from hardware reliability is measuring the wrong thing.

Use the arithmetic to find the *ceiling* imposed by serial dependencies — that use is sound and
often surprising ("we promised four nines and we depend on a three-nines service"). Do not use it
to justify a redundancy claim. For that, you need DS-701 L10's chaos experiments: measure, do not
compute.

## 3. Construction: reading services, and the dependency graph

Build in `mpse/ca731/l01/`. Much of this lesson's construction is analytical rather than code —
that is deliberate, and the artifacts are reusable.

**Stage 1 — the service reading.** Take three managed services you use or plan to use (a
key-value store, a queue or stream, and an object store are a good spread). For each, answer all
seven questions from §2.2 *from the documentation*, with citations. Where the documentation does
not answer a question, say so — the gaps are informative, and they are where you will be surprised
later.

**Stage 2 — verify a claim.** Pick one guarantee from Stage 1 and test it. Read-after-write
consistency, a throttling threshold, the behaviour at a partition's throughput ceiling, the actual
latency distribution under a hot key. Write the experiment, run it, and report whether the
documentation was accurate, incomplete, or misleading. This is the habit the course is trying to
build.

**Stage 3 — the dependency graph.** For a system you work with (or a reference architecture you
design here), draw the full dependency graph including the invisible ones: DNS, identity, secrets,
certificate issuance, container registry, the CI system, the observability stack. Mark every shared
node. For each shared node, write one line: what happens when it is unavailable, and whether your
recovery path depends on it. Most people find at least one dependency they did not know they had.

**Stage 4 — the control-plane dependency audit.** From Stage 3, identify every recovery action
that requires a control-plane call. For each, design the statically stable alternative and cost it.
Then state, per scenario, whether you would pay for static stability — the answer is legitimately
"no" for some, and the point is to decide rather than to default.

**Stage 5 — throttling behaviour.** Take a service with a documented rate limit. Drive it past the
limit under load and observe: the error returned, whether it is retriable, whether the service
degrades gracefully or cliffs, and what your client library does by default. Then implement the
correct client behaviour (DS-701 L09: backoff, full jitter, a retry budget) and re-measure.
Client libraries' defaults are often wrong for your case, and you should know which.

**Stage 6 — the cost model.** For a reference architecture, build a spreadsheet or script that
computes monthly cost as a function of: request volume, data stored, data transferred (split by
same-AZ, cross-AZ, cross-region, internet), and idle time. Then run three scenarios — 10× traffic,
a 10× larger dataset, and traffic at 5% of current — and identify which term dominates in each. The
term that dominates at your *expected* scale is the one your architecture should optimise, and it
is frequently not the one people focus on.

**Stage 7 — availability arithmetic, then reality.** Compute the theoretical ceiling for your
architecture from its serial dependencies. Then list the last five incidents in that system (or in
published postmortems for a similar one) and classify each by cause: component failure, change,
dependency, capacity, or human. Compare the distribution against what the arithmetic models. Write
400 words on the gap.

## 4. Failure modes

- **Treating managed as "not my problem".** The consistency model, the limits and the failure modes
  are all still yours.
- **Recovery paths that depend on the control plane.** They will be exercised at the worst moment.
- **"Multi-AZ" without knowing what is actually shared** — a capacity pool, a control plane, a
  global service.
- **Ignoring documented limits** until they are hit in production.
- **Client libraries' default retry behaviour**, unexamined. Some retry aggressively without
  jitter, which is DS-701 L09's amplification problem.
- **Availability arithmetic used to justify a redundancy claim.** Correlation invalidates it.
- **Cost as a finance problem.** It is an architecture problem; the finance team cannot fix a
  chatty cross-AZ design.
- **Cross-AZ data transfer, unnoticed.** Frequently a large fraction of the bill of an otherwise
  well-designed system.
- **Optimising the wrong cost term.** Compute is the visible one and often not the dominant one.
- **A single region with an implicit assumption of regional durability.** Decide, and write down,
  what you would actually do in a region-wide event.

## 5. Exercises

### Warm-up (30 min)

1. Give the seven questions for reading a managed service, and answer them for one you know well.
2. Distinguish control plane from data plane across all five rows of §2.3's table, and state the
   architectural rule that follows.
3. Compute the availability ceiling of a service depending serially on components at 99.99%,
   99.9% and 99.95%. Then give three reasons the number is optimistic.

### Core (3 h)

4. Complete Stages 1–2: three service readings and one verified (or falsified) claim.
5. Complete Stages 3–4: the dependency graph with shared nodes marked, and the control-plane audit
   with a decision per scenario.
6. Complete Stage 5 and report what your client library does by default versus what it should do.
7. Complete Stage 6 and identify the dominant cost term at three scales.

### Challenge

8. Complete Stage 7, then take a published cloud provider postmortem of a major incident, and
   write a 1,000-word analysis: what failed, what the correlated failure was, which customers were
   affected and why, what the provider changed, and — the part that matters — **what a customer
   architecture would have had to look like to survive it**. Be honest about whether that
   architecture is one you would actually pay for.
9. Build a **static stability simulator**: model a three-AZ service with autoscaling, and simulate
   an AZ loss under two designs — one that scales up in response (control-plane dependent, with a
   modelled control-plane delay and failure probability) and one pre-provisioned at N+1. Include a
   scenario where the control plane is degraded and where the remaining capacity pool is contended
   by other tenants. Report availability and cost for both, and state the utilisation level at
   which static stability stops being worth it.

## 6. Self-check

1. What does "managed" move, and what does it leave with you?
2. Give the seven questions for reading a managed service.
3. Why do control planes fail more often than data planes, and what rule follows?
4. Define static stability and constant work, and say what each protects against.
5. Give four sources of correlated failure.
6. Why does S3's durability figure say nothing about its availability?
7. What does a pricing model tell you about an architecture? Give two concrete inferences.
8. Give the availability/downtime table and the two multiplication rules.
9. Give three reasons availability arithmetic overstates real availability.
10. Why is cost per tenant more informative than total cost?

## 7. Primary sources

- **Brooker, "Static Stability Using Availability Zones", "Reliability, Constant Work, and a Good
  Cup of Coffee", and "Workload Isolation Using Shuffle Sharding"** (AWS Builders' Library) — the
  clearest public writing on this material.
- **Vogels et al., "Amazon DynamoDB: A Scalable, Predictably Performant, and Fully Managed NoSQL
  Database Service" (USENIX ATC 2022)** — a managed service explained by its builders; use it as
  the worked example for §2.2.
- Google, *Site Reliability Engineering*, chapters 3 and 22.
- Cloud provider public post-incident reports (AWS, Azure, GCP). Read six; they teach the
  correlated-failure material better than any textbook.
- The AWS Well-Architected Framework, read critically.
- DS-701's entire reading list, which is the substrate this lesson sits on.

---

**Next:** [L02 — Multi-Tenancy and Isolation](L02-multi-tenancy-and-isolation.md)
