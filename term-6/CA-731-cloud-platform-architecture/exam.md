# CA-731 — Written Examination

**Time allowed: 3 hours. Closed book, no machine.**
**Answer FOUR from Section A and ONE from Section B.**
Section A: 15 marks each. Section B: 40 marks.

Where a question asks for arithmetic, show it. Where it asks for a trade-off, name what is given up
as well as what is gained — an answer that lists only benefits scores in the lowest band regardless
of its technical accuracy. Provider-specific answers are acceptable where the question invites them,
provided the underlying principle is stated.

---

## Section A — answer FOUR

**A1.** Explain what a managed service moves and what it leaves with the customer. Then give the
seven questions for reading a managed service as a distributed system, and apply them to a service
of your choosing. *(15)*

**A2.** Distinguish control plane from data plane across availability, complexity, request rate and
blast radius. State the architectural rule that follows, and explain why a recovery path depending
on the control plane is dangerous in a *large* event specifically. *(15)*

**A3.** Define static stability and constant work. Explain what each protects against, what each
costs, and give a concrete design for each. *(15)*

**A4.** Give four sources of correlated failure in a cloud architecture, with a concrete example of
each. Then explain why availability arithmetic overstates real availability, giving three distinct
reasons. *(15)*

**A5.** Explain how a pricing model reveals an architecture, with two worked inferences. Then give
six drivers of a cloud bill ranked by how often they surprise people, and say which of them are
consequences of decisions made for other reasons. *(15)*

**A6.** Give the multi-tenancy isolation spectrum with cost and blast radius at each point. Explain
why a mature platform is tiered rather than uniform, and what feature that implies must exist from
the beginning. *(15)*

**A7.** Distinguish data isolation from performance isolation. Give five defence-in-depth controls
for data isolation, identifying which make the wrong thing impossible rather than merely
detectable, and explain why 404 rather than 403 is the correct response. *(15)*

**A8.** Explain shuffle sharding and derive the overlap probability for N workers with k per tenant.
Compute it for N=8, k=2, and state what the resulting figure means for the fraction of tenants
affected by one bad tenant. Then say what client-side retry adds. *(15)*

**A9.** Explain what each layer of isolation protects against and what crosses it, from application
filtering through to separate accounts. Then write the honest paragraph you would give a customer
asking whether their data is isolated from other tenants. *(15)*

**A10.** State the identity-as-boundary principle and give the technical content of "zero trust".
Then give the four levels of workload identity in order, saying what each fixes about the one below.
*(15)*

**A11.** Explain the SSRF-to-metadata-endpoint attack and the three independent controls that stop
it. Say which you would rely on if you could have only one, and why. *(15)*

**A12.** Compare RBAC, ABAC and ReBAC, giving for each a question it makes easy and one it makes
hard. Then explain why authorisation decisions should be made in one place against a version
controlled policy. *(15)*

**A13.** Give four reasons least privilege is hard in practice. Then give six practices that
actually work, explaining why a guardrail is more valuable than a fine-grained grant and why
IAM-modifying permissions are effectively administrative. *(15)*

**A14.** Give four reasons the network remains a control even when identity is primary. Then explain
segmentation by trust level, and why egress filtering is both the most skipped control and the one
that contains a compromise. *(15)*

**A15.** Compare NAT gateways, gateway endpoints and interface endpoints on traffic path and cost.
Then explain the continuous cost of a three-AZ chatty architecture and what trade it represents.
*(15)*

**A16.** Distinguish liveness from readiness probes and give the failure produced by conflating
them. Then explain why a deep health check on a non-critical dependency is harmful, and why
`X-Forwarded-For` is not a control unless something specific is true. *(15)*

**A17.** Explain why infrastructure as code is a distributed systems problem, naming the four
properties of the world that make it one. Then explain what state is, and why it must be locked,
encrypted and versioned. *(15)*

**A18.** Explain drift: its causes, why it is dangerous rather than merely untidy, and the four ways
to handle a detected drift with the rule for choosing. *(15)*

**A19.** Explain why state is part of a module's public contract, with the concrete failure that
results from ignoring it. Then give the criteria that distinguish a real module abstraction from a
pass-through wrapper. *(15)*

**A20.** Describe what a Terraform provider implements per resource type. Explain what `Read` must
do when a resource no longer exists and why, and give two other provider subtleties with the
operator-visible symptom of getting each wrong. *(15)*

**A21.** State the reconciliation pattern in pseudocode and give its four properties. Explain
level-triggered versus edge-triggered with the concrete failure of the latter, and say why every
serious controller needs rate limiting. *(15)*

**A22.** Narrate the eight steps from `kubectl apply` to a serving pod, marking each asynchronous
boundary, and give the command that localises a failure at each step. *(15)*

**A23.** Give the eight rules for writing a controller and the bug each prevents. Explain in
particular what `observedGeneration` is for and what a finalizer that cannot complete does. *(15)*

**A24.** Explain how an admission webhook can render a cluster unrecoverable, and the specific
configuration choices that prevent it. Then state the honest case for and against Kubernetes as a
platform substrate, with the threshold rule. *(15)*

**A25.** Distinguish SLI, SLO and SLA. Give four criteria for a good SLI, an example of a gameable
one, and explain why a latency SLI is expressed as a ratio rather than as a percentile. *(15)*

**A26.** Explain the error budget, why an unspent budget represents a cost, and why the policy must
be agreed before the budget is exhausted. Then address the three details that make a policy survive
contact with a business. *(15)*

**A27.** Give the price-of-a-nine table from 99% to 99.999%. Explain why humans cannot be in the
recovery path above about 99.9%, and why high targets constrain deployment practice more than
hardware. *(15)*

**A28.** Explain multi-window multi-burn-rate alerting, giving the rules for a 99.9%/30-day SLO and
what each dimension buys. Then explain why a multi-tenant service requires per-tenant SLIs. *(15)*

**A29.** State Little's Law and apply it to a service at 2,000 requests/second with 150 ms mean
latency. Then explain why utilisation and latency trade against each other, with the queueing
relationship, and what the Universal Scalability Law adds. *(15)*

**A30.** Give five reasons autoscaling fails as an availability strategy, and state the summary rule
about autoscaling versus provisioning. Then explain why scaling on concurrency beats scaling on
CPU. *(15)*

**A31.** Give the seven cost optimisation steps in order and explain why the order matters. Then
explain what a commitment discount is actually paying for, and what that implies about making
workloads more predictable. *(15)*

**A32.** State the platform-as-product claim and the test for whether something is platform work.
Give the five properties of a golden path and explain why escapability and completeness both
matter. *(15)*

**A33.** Distinguish guardrails from gates and explain why the distinction determines whether a
platform scales. Then explain why error message quality is a platform feature and why changing a
default is a breaking change. *(15)*

**A34.** Give three groups of platform metrics with an example of each, and one commonly-used metric
that misleads, with the reason. Then explain what the teams who routed around your platform are
worth to you. *(15)*

**A35.** Give the four dimensions of a design review and a trade-off between each pair you can
construct. Explain why reviewing them separately produces an incoherent design. *(15)*

**A36.** Give the ten-step design review method in order. Explain why requirements come before the
design, why "what happens when it is slow?" is more useful than "what happens when it fails?", and
why review effort should be proportionate to reversibility. *(15)*

---

## Section B — answer ONE

**B1. Design the platform.** A company with twelve product teams, each running three to eight
services, currently has no platform: every team writes its own Terraform, its own pipeline, and its
own observability, with wide variation in quality. You have been asked to build a platform.

Write the plan. Your answer must: state what the platform will and will not do, and which teams are
out of scope; describe the first golden path and justify the choice; state the self-service
interface and why it takes the form it does; explain how you would earn adoption without a mandate,
including what you would do about the strongest team who will not want it; give the interface
stability and deprecation commitment; give the support and on-call model; state the three metrics by
which you would judge the platform after a year and the threshold at which you would call it a
failure; and describe what you would build *second*, and why not first. *(40)*

**B2. The multi-tenant design.** A B2B SaaS product has 3,000 tenants: 2,900 small ones consuming
almost nothing each, 90 medium, and 10 large enterprises, two of whom are regulated and demanding
dedicated infrastructure. Load is concentrated 09:00–18:00 in two time zones.

Design the tenancy model. Your answer must: choose a tiering strategy and justify it economically,
addressing the statistical multiplexing gain and what the load concentration does to it; state the
data isolation controls at each tier and what each protects against; state the performance isolation
mechanisms and why you chose them, including the arithmetic if you use shuffle sharding; describe
the migration path between tiers and why it must exist from day one; explain how a single tenant's
data would be exported and deleted; address per-tenant SLOs and how you would alert; and state the
honest paragraph you would give the regulated customer's security team. *(40)*

**B3. The incident and the redesign.** A platform's control plane became unavailable for two hours
during a regional event. During that window: autoscaling could not add capacity as load shifted;
deploys were impossible, so a fix could not be shipped; the certificate manager could not renew a
certificate that expired mid-incident; and several teams' recovery runbooks began with "launch
replacement instances". Customer-visible availability was 12% for ninety minutes.

Write the analysis and the redesign. Your answer must: explain why each of the four consequences
followed, in terms of the control plane / data plane distinction; identify which were correlated
failures and what they shared; state what static stability would have changed for each, and what it
would have cost; explain why "add more automation to recover faster" is the wrong lesson here and
what the right one is; describe how you would find every remaining control-plane dependency in the
recovery path across twelve teams, given that you cannot read all their runbooks; and give the
error budget and postmortem consequences, including whether this incident should trigger a freeze
and why. *(40)*

**B4. The cost intervention.** A company's cloud bill has grown 3× in eighteen months while revenue
grew 1.4×. Engineering says the growth is "just scale". You have been asked to find out.

Write the investigation and the plan. Your answer must: state what you would measure first and why
unit cost is the key number; describe how you would establish allocation given that tagging is
inconsistent; enumerate the specific things you would expect to find, ranked by likelihood, and say
which are architecture problems versus hygiene problems; give the optimisation sequence with the
reasoning for its order and an honest estimate of effort versus saving at each step; identify the
architectural changes you would consider only after the cheap wins, with their risk; explain how
you would make cost a standing engineering input rather than a periodic panic; and state what you
would do if the finding is that a small number of customers are unprofitable — being clear about
what is an engineering decision and what is not. *(40)*

**B5. The design review.** You are given the design for a new multi-region, event-driven order
processing system. It uses: active-active across two regions with bidirectional replication;
microservices communicating over a service mesh with retries enabled; a managed queue per service;
autoscaling on CPU; and a shared identity service. The stated target is 99.99% availability. The
document has sections for architecture, data model and API, and nothing else.

Write the review. Your answer must: list the questions you would ask before looking at the design at
all; identify at least five failure-dimension concerns, being specific about the *slow* cases and
the retry interaction between the mesh and the applications; identify the correlated failures and
the shared dependencies; assess whether the availability target is achievable given the design and
its dependencies, with arithmetic; identify the cost consequences of the design choices and which
term you would expect to dominate; identify the security and operability gaps; state which of your
concerns are blocking and which are preferences, and why that distinction matters; and say what you
would ask the author to add to the document. *(40)*

---

*Marks in Section A are awarded for precision, arithmetic where asked for, and for naming trade-offs
rather than only benefits. In Section B, an answer that does not state what it gave up, and a
condition under which its recommendation would be wrong, cannot reach the upper band however
technically sound it is. Answers that recommend a specific product without stating the underlying
requirement it satisfies will be marked down.*
