# CA-731 · Lesson 04 — Network Architecture and the Perimeter That Isn't

**Estimated study time:** 4.5 hours
**Prerequisites:** L01, L03; DS-701 L09

---

## 1. Orientation

L03 argued that identity, not network location, is the primary security boundary. That is correct,
and it is regularly over-read into "the network does not matter". The network matters for four
reasons that identity does not address:

1. **Reachability is a real control.** A service that is not routable from the internet cannot be
   attacked from the internet, whatever bugs it has. Identity fails open when there is a bug in the
   authentication code; a missing route does not.
2. **Blast radius.** Segmentation limits lateral movement after a compromise — which, per L03 §2.7,
   is the step that turns a single compromised credential into an incident.
3. **Cost.** Data transfer pricing is a function of network topology (L01 §2.5), and it is a
   frequently-large line item that architecture determines.
4. **Latency and failure domains.** Where things are placed determines round-trip time and which
   failures are correlated.

So the modern position is not "identity instead of network" but **defence in depth: identity as the
primary control, network as an independent second one whose failure modes are uncorrelated with
the first.** Two controls that fail for different reasons is the entire argument.

This lesson also aims to fix a specific gap: many engineers who are strong on distributed systems
are vague on the network layer, and treat a VPC as a checkbox. The consequence is designs that are
expensive, over-permissive, and impossible to debug at 3 a.m.

## 2. Theory

### 2.1 The building blocks

The primitives are broadly the same across providers, whatever they are called:

- **A virtual network** (VPC, VNet) — an isolated address space you control, with a CIDR block.
- **Subnets**, usually one per availability zone, classified as *public* (has a route to an internet
  gateway) or *private* (does not).
- **Route tables**, which determine where traffic for a destination goes. Routing is the actual
  reachability control; everything else is filtering on top of it.
- **Gateways**: internet gateway (bidirectional, for public subnets), NAT gateway (outbound only,
  for private subnets), and private endpoints (§2.3).
- **Security groups** — stateful filters attached to an interface. Return traffic is automatically
  allowed. Reference other security groups as sources, which is the important feature: "allow from
  the app tier" rather than "allow from 10.0.2.0/24".
- **Network ACLs** — stateless filters at the subnet level. Because they are stateless you must
  allow return traffic explicitly on ephemeral ports, which is the source of most NACL debugging
  pain. Use them sparingly, for coarse deny rules only.

The single most important design decision is boring: **the address plan**. Choose CIDR ranges that
do not overlap with anything you might ever need to connect to — other VPCs, on-premises networks,
partners, acquisitions. Overlapping CIDRs cannot be peered, and the remedy is either NAT (painful,
breaks source-IP-based anything, complicates debugging) or renumbering (a project). Leave room:
allocate generously per environment, and write the plan down.

### 2.2 Segmentation that reflects trust

The classical three-tier layout — public subnets for load balancers, private for application,
isolated (no outbound route at all) for data — is a good default, and the reason is that it maps
segments to trust levels rather than to org chart or convenience.

Principles that hold beyond the default:

- **Segment by trust level and blast radius**, not by team. A segment should answer "what can reach
  what, and what happens if something here is compromised?"
- **Default deny, both directions.** Egress filtering is the one people skip, and it is what turns
  a compromise into a *contained* compromise: exfiltration and command-and-control both require
  outbound connectivity, and a data tier with no outbound route cannot phone home.
- **Reference identities, not addresses.** Security groups referencing other security groups
  survive scaling and re-addressing; CIDR-based rules rot into a list nobody dares to prune.
- **Separate environments at the account level, not the subnet level.** A misconfigured rule in one
  subnet is a plausible mistake; crossing an account boundary requires several.

### 2.3 Private connectivity and the NAT gateway trap

A private subnet still needs to reach the provider's own services — object storage, the metadata
API, a managed database. Three ways, and the difference between them is large:

- **Via NAT gateway to the public endpoint.** Works immediately. Traffic leaves your VPC, crosses
  the NAT, and comes back through the provider's public network. You pay per GB *and* per hour, on
  every byte to your own object store. This is a very common and very large invisible line item.
- **Gateway endpoints** (for object storage and a few others): a route table entry sending that
  traffic directly. Usually free, and often the single easiest cost saving available. Check whether
  yours exist; many systems are paying NAT charges for traffic that a route table entry would
  eliminate.
- **Private link / interface endpoints**: a private IP in your subnet for a specific service.
  Costs per hour and per GB, but keeps traffic off the public network entirely and is what you
  want for third-party SaaS and cross-account service exposure.

The general principle: **know which of your traffic crosses a NAT and what it costs.** Then check
the same for cross-AZ: an internal call between AZs is charged in both directions by many
providers, so a three-AZ deployment of chatty services pays continuously for its availability.
That is a defensible trade, but only if it is a decision.

### 2.4 Ingress, and where TLS terminates

The path in: DNS → global edge (CDN/anycast) → load balancer → service.

The decisions that matter:

- **Layer 4 versus layer 7.** L4 (TCP) is faster and preserves the client IP naturally; L7 (HTTP)
  can route by path or header, retry, and enforce per-request policy. Most application traffic
  wants L7 at the edge.
- **Where TLS terminates.** At the edge (simplest, and internal traffic is then plaintext unless
  re-encrypted), at the load balancer, or end-to-end to the service (mTLS, §2.5). "TLS everywhere
  internally" is now the default expectation, and a service mesh is the usual way to get it without
  every application implementing it.
- **The client IP.** After a proxy, the source address is the proxy's. `X-Forwarded-For` is
  attacker-controllable unless you strip and re-set it at a trusted edge — and rate limiting or
  geo-blocking on a spoofable header is not a control. Get this right; it is a common subtle bug.
- **Health checks.** A shallow check (the process is up) versus a deep one (dependencies reachable).
  DS-701 L09's rule applies: a deep check that fails on a non-critical dependency takes down a
  service that could have degraded gracefully. Distinguish *liveness* (should I be restarted?) from
  *readiness* (should I receive traffic?), and give them different logic.
- **Connection draining** on deregistration, or every deploy drops in-flight requests.

DNS deserves its own note, because it is the most common cause of "the network is broken":

- **TTLs are a commitment.** A five-minute TTL means failover takes at least five minutes, and
  resolvers and application runtimes frequently ignore TTLs — the JVM's historical
  cache-forever-by-default behaviour has caused many incidents.
- **DNS is a global shared dependency** (L01 §2.4) and a single point of correlated failure.
- **Health-check-based DNS failover is slow and unreliable** as a primary mechanism. Use it for
  regional failover, not for instance-level availability.

### 2.5 Service to service

Once inside, the questions are authentication, encryption, discovery, and policy.

**Service meshes** (Istio, Linkerd) provide mTLS with automatic certificate rotation (SPIFFE
identities, L03 §2.2), L7 policy, retries and circuit breaking (DS-701 L09), and per-call
observability — all without application changes, via a sidecar proxy. What they cost is real: a
proxy per pod (memory and latency), a control plane to operate, and a substantial increase in the
number of things that can be misconfigured. The honest assessment: **a mesh is worth it when you
have enough services that implementing mTLS and policy per service is worse than operating a mesh
— which is a larger number than vendors suggest and a smaller one than sceptics claim.** For under
about a dozen services, library-based mTLS is usually simpler.

Note also that a mesh's retry and circuit-breaking features are DS-701 L09's mechanisms in
configuration form, and they carry the same hazards: mesh-level retries stack with
application-level retries and multiply (L09 §2.4). Decide which layer retries. One of them.

### 2.6 Debugging connectivity

Because you will do this at 3 a.m., have a procedure. Work outward in layers:

1. **DNS**: does the name resolve, to what, and is the answer cached somewhere stale?
2. **Routing**: is there a route to that destination from this subnet? (Reachability analysers
   exist on all major providers and answer this authoritatively — use them rather than guessing.)
3. **Filtering**: security group inbound *on the destination*, outbound *on the source*, and NACLs
   in both directions including ephemeral ports.
4. **Listening**: is the process bound to the right interface? Bound to `127.0.0.1` rather than
   `0.0.0.0` is the classic.
5. **TLS**: certificate validity, hostname match, trust chain, protocol version.
6. **Application**: authentication, authorisation, and the request itself.

The signal that saves the most time: **a connection that hangs is usually a filtering problem
(packets dropped silently); a connection refused is usually nothing listening; a TLS error is
usually a certificate or SNI problem.** Also enable flow logs *before* you need them — they answer
"was the packet allowed?" definitively, and turning them on during an incident tells you nothing
about the incident.

### 2.7 Multi-region

Three postures, and the cost difference between them is an order of magnitude:

- **Single region, multi-AZ.** Correct for the large majority of systems. Survives an AZ failure,
  which is the failure that actually happens most.
- **Active-passive multi-region.** A warm standby with replicated data and DNS failover. The
  failover path must be exercised regularly or it does not work — an untested failover is a plan,
  not a capability. The honest questions are RPO (how much data can you lose?) and RTO (how long
  can you be down?), and both must be *measured*, not asserted.
- **Active-active multi-region.** Every DS-701 problem at once: replication lag, conflict
  resolution, split brain, and cross-region latency in the request path. Genuinely hard, genuinely
  expensive, and correct for a small number of systems.

The rule: **multi-region is a business decision with a technical implementation, and its cost is
usually underestimated by a factor of several.** The first question is not "how do we go
multi-region?" but "what is the cost of an hour of regional downtime, and how often do we expect
one?" — and for most systems the honest answer does not justify active-active.

## 3. Construction: build, break, and cost a network

Build in `mpse/ca731/l04/`. Terraform against a real account (small free-tier resources), or
LocalStack where the API supports it.

**Stage 1 — the address plan.** Design CIDR allocation for four environments across two regions,
with room to grow and no overlap with a hypothetical on-premises range and two hypothetical
acquisitions. Document the plan. Then deliberately create an overlap and attempt to peer the VPCs;
observe the failure and write down what the remediation would cost.

**Stage 2 — the three-tier VPC.** Build it in code: public, private and isolated subnets across
three AZs, with route tables, an internet gateway and a NAT gateway. Deploy a service in the
private tier behind a load balancer in the public tier, with a database in the isolated tier.

**Stage 3 — default deny.** Start with security groups allowing nothing and add only what is
required, using security-group references rather than CIDRs. Document each rule with what it is for.
Then add **egress** filtering and find out what breaks — package installs, telemetry, provider API
calls. Fix each properly rather than by widening the rule.

**Stage 4 — private endpoints and the NAT bill.** Measure traffic to object storage through the NAT
gateway and compute the monthly cost at a realistic volume. Add a gateway endpoint and re-measure.
Then add an interface endpoint for a second service and compute its break-even volume against NAT.
Report all three numbers.

**Stage 5 — cross-AZ cost.** Deploy a chatty pair of services and measure inter-AZ traffic under
load. Compute the monthly cost of the three-AZ deployment versus zone-affinity routing (prefer a
same-AZ peer, fall back across AZs). Then state what zone affinity costs you in availability, and
make the call.

**Stage 6 — connectivity debugging.** Deliberately introduce five distinct connectivity faults —
missing route, security group missing on the destination, missing egress on the source, stateless
NACL blocking the return path, and a process bound to `127.0.0.1`. For each, work through §2.6's
procedure and record what the *symptom* was. Build the symptom-to-cause table; it is worth more at
3 a.m. than any diagram.

**Stage 7 — ingress and TLS.** Terminate TLS at the load balancer, then re-encrypt to the backend.
Verify that the client IP is correctly propagated and that `X-Forwarded-For` is stripped and re-set
at the trusted edge. Demonstrate the spoofing attack against a rate limiter that trusts the header
naively, then fix it.

**Stage 8 — health checks.** Implement shallow liveness and deep readiness checks with different
logic. Then construct the failure: a non-critical dependency goes down, the deep check fails, and
the service is removed from the load balancer despite being able to serve most requests. Fix it by
separating critical from non-critical dependencies in the readiness logic.

**Stage 9 — failover, measured.** Set up an active-passive second region with data replication.
Then actually fail over: measure RTO (time from decision to serving) and RPO (data lost), including
DNS propagation with clients that ignore TTLs. Fail back. Write up what you learned — this exercise
routinely produces numbers several times worse than the team's assumption, and that gap is the
finding.

## 4. Failure modes

- **No address plan.** Overlapping CIDRs discovered when a merger or a partner connection is
  needed.
- **Security groups by CIDR.** They rot, and nobody dares prune them.
- **No egress filtering.** A compromise becomes an exfiltration.
- **NAT gateway charges for provider-service traffic.** A gateway endpoint would have been free.
- **Cross-AZ traffic unmeasured.** A significant recurring cost nobody attributes to a design
  decision.
- **Trusting `X-Forwarded-For`.** Spoofable, and therefore not a control.
- **Deep health checks on non-critical dependencies.** Takes down a service that could have
  degraded.
- **Liveness and readiness with the same logic.** A failed dependency causes a restart loop.
- **DNS TTL assumed to be honoured.** It frequently is not; failover takes longer than planned.
- **Flow logs enabled during the incident.** They tell you nothing about what already happened.
- **Retries at both the mesh and the application.** Multiplied load at the worst moment.
- **Multi-region failover never exercised.** It is a document, not a capability.

## 5. Exercises

### Warm-up (30 min)

1. Give four reasons the network still matters when identity is the primary boundary.
2. Distinguish security groups from NACLs, including why the stateless one causes more debugging
   pain.
3. Explain the three ways a private subnet reaches provider services and the cost of each.

### Core (3.5 h)

4. Complete Stages 1–3, including the peering failure and the egress-filtering breakages with
   proper fixes.
5. Complete Stages 4–5 and report the NAT, endpoint and cross-AZ numbers with the availability
   trade-off stated.
6. Complete Stage 6 and deliver the symptom-to-cause table.
7. Complete Stages 7–8, including the `X-Forwarded-For` spoofing demonstration and the health-check
   failure case.

### Challenge

8. Complete Stage 9 with measured RTO and RPO, and a written comparison against what the team (or
   you) assumed beforehand.
9. Design and implement a **network policy compliance checker**: given the infrastructure code,
   verify a set of assertions — no security group allows `0.0.0.0/0` on a port other than 80/443,
   no isolated subnet has an outbound route, every data-tier resource is unreachable from the
   internet, and every environment's CIDR is within its allocation. Run it in CI. Then extend it to
   *reachability* rather than configuration: use the provider's reachability analyser (or model the
   routes and rules yourself) to answer "can anything on the internet reach this database?"
   definitively, rather than by inspecting rules one at a time. Report the class of misconfiguration
   the configuration-level check misses and the reachability check catches.

## 6. Self-check

1. Give four reasons the network is a control even when identity is primary.
2. Why is the address plan the most consequential early decision?
3. State the segmentation principles, and say why egress filtering is the one usually skipped.
4. Compare NAT, gateway endpoints and interface endpoints on cost and traffic path.
5. Why does a three-AZ chatty architecture have a continuous cost, and what is the trade?
6. Why is `X-Forwarded-For` not trustworthy, and what makes it trustworthy?
7. Distinguish liveness from readiness, and give the failure of conflating them.
8. Give three reasons DNS failover is slower than its TTL suggests.
9. Give the six-layer connectivity debugging procedure and the three symptom heuristics.
10. Give the three multi-region postures and the question that should precede choosing one.

## 7. Primary sources

- **Google, *Building Secure and Reliable Systems*, chapters on design for least privilege and
  understandability** — the segmentation argument, done well.
- The AWS VPC documentation on route tables, endpoints and NAT — read the *pricing* pages alongside
  the feature pages, per L01 §2.5.
- AWS Builders' Library: "Using Load Shedding to Avoid Overload", "Implementing Health Checks",
  "Avoiding Fallback in Distributed Systems" — the health check article in particular.
- The SPIFFE specification, and the Istio and Linkerd architecture documentation. Read Linkerd's
  case for simplicity alongside Istio's for capability.
- Kubernetes network policy documentation and the CNI specification.
- RFC 7239 (`Forwarded` header) — and the reason it exists.
- Published network-related cloud postmortems; DNS and BGP incidents in particular are unusually
  instructive.

---

**Previous:** [L03](L03-identity-and-trust.md) · **Next:**
[L05 — Infrastructure as Code and Provider Design](L05-infrastructure-as-code.md)
